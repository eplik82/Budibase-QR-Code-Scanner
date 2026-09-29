<script>
  //By: Chungchun Wang (https://github.com/chungchunwang)
  //To make the plugin fit the look of Budibase, some of this code and the styling is from the Star Rating component (https://github.com/andz-bb/budibase-component-star-rating) referenced in the docs.
  //Scanning is done with jsQR, which (unlike html5-qrcode) can also decode inverted, light-on-dark QR codes.

  import { getContext, onDestroy, onMount, tick } from "svelte";
  import jsQR from "jsqr";

  //Property Fields
  export let field;
  export let label;
  export let autoStartCamera;
  export let fps;
  export let inversionMode;
  export let allowFileScan;
  export let scannerBox;
  export let scannerBoxWidth;
  export let scannerBoxHeight;

  const { styleable, builderStore } = getContext("sdk");
  const component = getContext("component");
  const formContext = getContext("form");
  const formStepContext = getContext("form-step");
  const fieldGroupContext = getContext("field-group");

  let fieldApi;
  let fieldState;

  const formApi = formContext?.formApi;
  const labelPos = fieldGroupContext?.labelPosition || "above";
  $: formStep = formStepContext ? $formStepContext || 1 : 1;
  $: formField = formApi?.registerField(
    field,
    "text",
    0,
    false,
    null,
    formStep
  );

  $: unsubscribe = formField?.subscribe((value) => {
    fieldState = value?.fieldState;
    fieldApi = value?.fieldApi;
  });

  $: labelClass =
    labelPos === "above" ? "" : `spectrum-FieldLabel--${labelPos}`;

  $: inBuilder = $builderStore?.inBuilder;

  // Frames are downscaled to at most this many pixels on the long side before decoding
  const MAX_DECODE_SIZE = 800;
  const CAMERA_STORAGE_KEY = "budibase-qr-scanner-camera";

  let video;
  let stream;
  let scanTimer;
  let fileInput;
  const canvas = document.createElement("canvas");
  const ctx = canvas.getContext("2d", { willReadFrequently: true });

  let cameras = [];
  let cameraId = readStoredCamera();
  let scanning = false;
  let starting = false;
  let success = false;
  let qrCodeValue = "";
  let errorMessage = "";

  function readStoredCamera() {
    try {
      return localStorage.getItem(CAMERA_STORAGE_KEY) || "";
    } catch (e) {
      return "";
    }
  }

  function storeCamera(id) {
    try {
      localStorage.setItem(CAMERA_STORAGE_KEY, id);
    } catch (e) {
      // Storage may be unavailable (private mode etc.) - not critical
    }
  }

  const decode = (imageData) =>
    jsQR(imageData.data, imageData.width, imageData.height, {
      inversionAttempts: inversionMode || "attemptBoth",
    });

  // Draws the given source region into the work canvas (downscaled) and decodes it
  function decodeRegion(source, sx, sy, sw, sh) {
    const scale = Math.min(1, MAX_DECODE_SIZE / Math.max(sw, sh));
    const w = Math.max(1, Math.round(sw * scale));
    const h = Math.max(1, Math.round(sh * scale));
    canvas.width = w;
    canvas.height = h;
    ctx.drawImage(source, sx, sy, sw, sh, 0, 0, w, h);
    return decode(ctx.getImageData(0, 0, w, h));
  }

  function scanVideoFrame() {
    const vw = video?.videoWidth;
    const vh = video?.videoHeight;
    if (!vw || !vh) return null;
    if (!scannerBox) return decodeRegion(video, 0, 0, vw, vh);

    // The scanner box is given in displayed pixels; map it to video pixels
    const ratio = vw / (video.clientWidth || vw);
    const sw = Math.min(vw, (scannerBoxWidth || 250) * ratio);
    const sh = Math.min(vh, (scannerBoxHeight || 250) * ratio);
    return decodeRegion(video, (vw - sw) / 2, (vh - sh) / 2, sw, sh);
  }

  function scanLoop() {
    if (!scanning) return;
    const started = performance.now();
    let result = null;
    try {
      result = scanVideoFrame();
    } catch (e) {
      // A frame that cannot be read yet is simply skipped
    }
    if (result?.data) {
      onScanSuccess(result.data);
      return;
    }
    const interval = 1000 / Math.min(60, Math.max(1, fps || 15));
    const elapsed = performance.now() - started;
    scanTimer = setTimeout(scanLoop, Math.max(0, interval - elapsed));
  }

  async function loadCameras() {
    try {
      const devices = await navigator.mediaDevices.enumerateDevices();
      cameras = devices.filter((d) => d.kind === "videoinput");
    } catch (e) {
      cameras = [];
    }
  }

  async function startCamera() {
    if (!formContext || inBuilder || starting) return;
    stopCamera();
    errorMessage = "";
    success = false;
    qrCodeValue = "";
    fieldApi?.setValue(qrCodeValue);

    if (!navigator.mediaDevices?.getUserMedia) {
      errorMessage =
        "Camera access is not available. The page must be served over HTTPS.";
      return;
    }

    starting = true;
    try {
      const videoConstraints = cameraId
        ? { deviceId: { exact: cameraId } }
        : { facingMode: "environment" };
      try {
        stream = await navigator.mediaDevices.getUserMedia({
          video: videoConstraints,
          audio: false,
        });
      } catch (e) {
        // The remembered camera may be gone - fall back to any camera
        if (!cameraId || e?.name !== "OverconstrainedError") throw e;
        cameraId = "";
        stream = await navigator.mediaDevices.getUserMedia({
          video: { facingMode: "environment" },
          audio: false,
        });
      }

      scanning = true;
      await tick();
      video.srcObject = stream;
      await video.play();

      const activeId = stream.getVideoTracks()[0]?.getSettings?.().deviceId;
      if (activeId) {
        cameraId = activeId;
        storeCamera(activeId);
      }
      await loadCameras();
      scanLoop();
    } catch (e) {
      stopCamera();
      errorMessage =
        e?.name === "NotAllowedError"
          ? "Camera permission was denied."
          : `Could not start camera: ${e?.message || e}`;
    } finally {
      starting = false;
    }
  }

  function stopCamera() {
    scanning = false;
    clearTimeout(scanTimer);
    stream?.getTracks().forEach((track) => track.stop());
    stream = null;
    if (video) video.srcObject = null;
  }

  function onCameraChange(e) {
    cameraId = e.target.value;
    storeCamera(cameraId);
    if (scanning) startCamera();
  }

  async function scanFile(e) {
    const file = e.target.files?.[0];
    e.target.value = "";
    if (!file) return;
    stopCamera();
    errorMessage = "";

    let bitmap;
    try {
      bitmap = await createImageBitmap(file);
      const result = decodeRegion(bitmap, 0, 0, bitmap.width, bitmap.height);
      if (result?.data) {
        onScanSuccess(result.data);
      } else {
        errorMessage = "No QR code was found in the image.";
      }
    } catch (err) {
      errorMessage = `Could not read the image: ${err?.message || err}`;
    } finally {
      bitmap?.close?.();
    }
  }

  function onScanSuccess(decodedText) {
    stopCamera();
    qrCodeValue = decodedText;
    fieldApi?.setValue(qrCodeValue);
    success = true;
  }

  onMount(() => {
    if (autoStartCamera) startCamera();
  });

  onDestroy(() => {
    stopCamera();
    fieldApi?.deregister();
    unsubscribe?.();
  });
</script>

<div class="spectrum-Form-item" use:styleable={$component.styles}>
  {#if !formContext}
    <div class="placeholder">Form components need to be wrapped in a form</div>
  {:else}
    <label
      class:hidden={!label}
      for={fieldState?.fieldId}
      class={`spectrum-FieldLabel spectrum-FieldLabel--sizeM spectrum-Form-itemLabel ${labelClass}`}
    >
      {label || " "}
    </label>
    <div class="spectrum-Form-itemField">
      <div class="scanner">
        {#if inBuilder}
          <div class="placeholder">The camera is disabled in the builder preview.</div>
        {:else}
          {#if scanning}
            <div class="video-wrapper">
              <!-- svelte-ignore a11y-media-has-caption -->
              <video bind:this={video} muted playsinline></video>
              {#if scannerBox}
                <div
                  class="scanner-box"
                  style={`width: ${scannerBoxWidth || 250}px; height: ${scannerBoxHeight || 250}px;`}
                ></div>
              {/if}
            </div>
          {/if}

          {#if success}
            <p class="result">Scanned Result: {qrCodeValue}</p>
          {/if}

          {#if errorMessage}
            <p class="error">{errorMessage}</p>
          {/if}

          <div class="controls">
            {#if scanning}
              {#if cameras.length > 1}
                <select value={cameraId} on:change={onCameraChange}>
                  {#each cameras as camera, i (camera.deviceId)}
                    <option value={camera.deviceId}>
                      {camera.label || `Camera ${i + 1}`}
                    </option>
                  {/each}
                </select>
              {/if}
              <button type="button" on:click={stopCamera}>Stop Scanning</button>
            {:else}
              <button type="button" on:click={startCamera} disabled={starting}>
                {success ? "Rescan?" : "Start Scanning"}
              </button>
            {/if}
            {#if allowFileScan}
              <button type="button" on:click={() => fileInput.click()}>
                Scan an Image File
              </button>
              <input
                bind:this={fileInput}
                type="file"
                accept="image/*"
                hidden
                on:change={scanFile}
              />
            {/if}
          </div>
        {/if}
      </div>
    </div>
  {/if}
</div>

<style>
  .placeholder {
    color: var(--spectrum-global-color-gray-600);
  }
  label {
    white-space: nowrap;
  }
  label.hidden {
    padding: 0;
  }
  .spectrum-Form-itemField {
    position: relative;
    width: 100%;
  }
  .scanner {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 8px;
    text-align: center;
  }
  .video-wrapper {
    position: relative;
    width: 100%;
    display: flex;
    justify-content: center;
    align-items: center;
    overflow: hidden;
  }
  video {
    display: block;
    width: 100%;
    height: auto;
  }
  .scanner-box {
    position: absolute;
    max-width: 100%;
    max-height: 100%;
    box-sizing: border-box;
    border: 2px solid white;
    box-shadow: 0 0 0 9999px rgba(0, 0, 0, 0.4);
    pointer-events: none;
  }
  .controls {
    display: flex;
    flex-wrap: wrap;
    justify-content: center;
    gap: 8px;
  }
  .result {
    margin: 0;
    word-break: break-all;
  }
  .error {
    margin: 0;
    color: var(
      --spectrum-semantic-negative-color-default,
      var(--spectrum-global-color-red-500)
    );
    font-size: var(--spectrum-global-dimension-font-size-75);
  }
  .spectrum-FieldLabel--right,
  .spectrum-FieldLabel--left {
    padding-right: var(--spectrum-global-dimension-size-200);
  }
</style>
