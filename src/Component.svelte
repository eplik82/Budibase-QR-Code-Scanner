<script>
  //By: Chungchun Wang (https://github.com/chungchunwang)
  //To make the plugin fit the look of Budibase, some of this code and the styling is from the Star Rating component (https://github.com/andz-bb/budibase-component-star-rating) referenced in the docs.
  //Scanning is done with jsQR, which (unlike html5-qrcode) can also decode inverted, light-on-dark QR codes.

  import { getContext, onDestroy, onMount, tick } from "svelte";
  import jsQR from "jsqr";

  //Property Fields
  export let field;
  export let label;
  export let inversionMode;
  export let autoStartCamera;
  export let continuousScan;
  export let scanDelay;
  export let showResult;
  export let fps;
  export let preferredCamera;
  export let resolution;
  export let zoom;
  export let showZoomControl;
  export let showTorchButton;
  export let soundOnScan;
  export let soundType;
  export let soundVolume;
  export let vibrateOnScan;
  export let allowFileScan;
  export let scannerBox;
  export let scannerBoxWidth;
  export let scannerBoxHeight;
  export let maxVideoHeight;
  export let onScan;

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
  // Signature: field, type, defaultValue, disabled, readonly, validationRules, step
  $: formField = formApi?.registerField(
    field,
    "string",
    null,
    false,
    false,
    [],
    formStep
  );

  $: unsubscribe = formField?.subscribe((value) => {
    fieldState = value?.fieldState;
    fieldApi = value?.fieldApi;
  });

  $: labelClass =
    labelPos === "above" ? "" : `spectrum-FieldLabel--${labelPos}`;

  $: inBuilder = $builderStore?.inBuilder;

  // Frames are scaled to at most this many pixels on the long side before decoding
  const MAX_DECODE_SIZE = 800;
  const MAX_ZOOM = 8;
  const CAMERA_STORAGE_KEY = "budibase-qr-scanner-camera";
  const RESOLUTIONS = {
    sd: { width: 640, height: 480 },
    hd: { width: 1280, height: 720 },
    fullhd: { width: 1920, height: 1080 },
  };
  // [frequency (Hz), start offset (s), duration (s)]
  const SOUNDS = {
    beep: [[1000, 0, 0.12]],
    double: [
      [1200, 0, 0.08],
      [1200, 0.13, 0.08],
    ],
    chime: [
      [880, 0, 0.12],
      [1320, 0.12, 0.2],
    ],
  };

  let video;
  let videoWrapper;
  let videoAspect = 4 / 3;
  let stream;
  let track;
  let scanTimer;
  let fileInput;
  let audioContext;
  const canvas = document.createElement("canvas");
  const ctx = canvas.getContext("2d", { willReadFrequently: true });

  let cameras = [];
  let cameraId = readStoredCamera();
  let scanning = false;
  let starting = false;
  let success = false;
  let qrCodeValue = "";
  let errorMessage = "";
  let lastScanValue = null;
  let lastScanSeen = 0;

  // Zoom: hardware zoom is used when the camera supports it, the rest is done digitally
  let zoomLevel = 1;
  let hardwareZoom = null;
  let digitalZoom = 1;
  let torchSupported = false;
  let torchOn = false;

  $: zoomLevel = clampZoom(zoom);

  // Keeps the camera view from growing taller than the screen (portrait phone
  // streams are taller than wide) and hiding the controls below it
  $: videoHeightLimit = Math.min(100, Math.max(10, Number(maxVideoHeight) || 60));

  function clampZoom(value) {
    const z = Number(value);
    return Number.isFinite(z) ? Math.min(MAX_ZOOM, Math.max(1, z)) : 1;
  }

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

  // Audio can only start after a user gesture, so this is called from click handlers too
  function unlockAudio() {
    if (!soundOnScan) return;
    try {
      const AudioContextClass = window.AudioContext || window.webkitAudioContext;
      if (!AudioContextClass) return;
      audioContext = audioContext || new AudioContextClass();
      if (audioContext.state === "suspended") audioContext.resume();
    } catch (e) {
      audioContext = null;
    }
  }

  function playScanSound() {
    unlockAudio();
    if (!audioContext) return;
    const volume = Math.min(100, Math.max(0, soundVolume ?? 50)) / 100;
    const now = audioContext.currentTime;
    for (const [frequency, offset, duration] of SOUNDS[soundType] || SOUNDS.beep) {
      const oscillator = audioContext.createOscillator();
      const gain = audioContext.createGain();
      oscillator.type = "sine";
      oscillator.frequency.value = frequency;
      gain.gain.setValueAtTime(0.0001, now + offset);
      gain.gain.exponentialRampToValueAtTime(Math.max(0.0001, volume), now + offset + 0.01);
      gain.gain.exponentialRampToValueAtTime(0.0001, now + offset + duration);
      oscillator.connect(gain).connect(audioContext.destination);
      oscillator.start(now + offset);
      oscillator.stop(now + offset + duration + 0.02);
    }
  }

  const decode = (imageData) =>
    jsQR(imageData.data, imageData.width, imageData.height, {
      inversionAttempts: inversionMode || "attemptBoth",
    });

  // Draws the given source region into the work canvas and decodes it.
  // Digitally zoomed regions are upscaled so small codes get more pixels.
  function decodeRegion(source, sx, sy, sw, sh, upscale = 1) {
    const longSide = Math.max(sw, sh);
    const target = Math.min(MAX_DECODE_SIZE, longSide * upscale);
    const scale = target / longSide;
    const w = Math.max(1, Math.round(sw * scale));
    const h = Math.max(1, Math.round(sh * scale));
    canvas.width = w;
    canvas.height = h;
    ctx.imageSmoothingEnabled = true;
    ctx.imageSmoothingQuality = "high";
    ctx.drawImage(source, sx, sy, sw, sh, 0, 0, w, h);
    return decode(ctx.getImageData(0, 0, w, h));
  }

  function scanVideoFrame() {
    const vw = video?.videoWidth;
    const vh = video?.videoHeight;
    if (!vw || !vh) return null;

    // Only the visible centre of the frame is scanned. The video is shown with
    // object-fit: cover in a box digitalZoom times the size of the wrapper,
    // so it may be cropped by the height limit and by the zoom.
    const ww = videoWrapper?.clientWidth || vw;
    const wh = videoWrapper?.clientHeight || vh;
    const ratio = 1 / (Math.max(ww / vw, wh / vh) * digitalZoom);
    let sw = Math.min(vw, ww * ratio);
    let sh = Math.min(vh, wh * ratio);
    if (scannerBox) {
      // The scanner box is given in displayed pixels; map it to video pixels
      sw = Math.min(sw, (scannerBoxWidth || 250) * ratio);
      sh = Math.min(sh, (scannerBoxHeight || 250) * ratio);
    }
    return decodeRegion(video, (vw - sw) / 2, (vh - sh) / 2, sw, sh, digitalZoom);
  }

  function updateVideoAspect() {
    if (video?.videoWidth && video?.videoHeight) {
      videoAspect = video.videoWidth / video.videoHeight;
    }
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
    if (result?.data && handleDecoded(result.data)) {
      if (!continuousScan) return;
    }
    const interval = 1000 / Math.min(60, Math.max(1, fps || 15));
    const elapsed = performance.now() - started;
    scanTimer = setTimeout(scanLoop, Math.max(0, interval - elapsed));
  }

  // Returns true when the value was accepted as a new scan
  function handleDecoded(value) {
    const now = Date.now();
    if (continuousScan) {
      // The same code only counts again after it has been out of view for the scan delay
      const isRepeat =
        value === lastScanValue && now - lastScanSeen < Math.max(0, scanDelay ?? 1500);
      lastScanValue = value;
      lastScanSeen = now;
      if (isRepeat) return false;
    }
    onScanSuccess(value);
    return true;
  }

  async function applyZoom() {
    let hwZoom = 1;
    const range = hardwareZoom;
    if (track && range) {
      hwZoom = Math.min(range.max, Math.max(range.min, zoomLevel));
      try {
        await track.applyConstraints({ advanced: [{ zoom: hwZoom }] });
      } catch (e) {
        hwZoom = 1;
      }
    }
    digitalZoom = Math.max(1, zoomLevel / hwZoom);
  }

  $: if (scanning && zoomLevel) applyZoom();

  async function toggleTorch() {
    if (!track || !torchSupported) return;
    try {
      await track.applyConstraints({ advanced: [{ torch: !torchOn }] });
      torchOn = !torchOn;
    } catch (e) {
      torchSupported = false;
    }
  }

  async function loadCameras() {
    try {
      const devices = await navigator.mediaDevices.enumerateDevices();
      cameras = devices.filter((d) => d.kind === "videoinput");
    } catch (e) {
      cameras = [];
    }
  }

  function videoConstraints(useStoredCamera) {
    const size = RESOLUTIONS[resolution] || RESOLUTIONS.hd;
    const constraints = {
      width: { ideal: size.width },
      height: { ideal: size.height },
    };
    if (useStoredCamera && cameraId) {
      constraints.deviceId = { exact: cameraId };
    } else {
      constraints.facingMode = preferredCamera === "front" ? "user" : "environment";
    }
    return constraints;
  }

  async function startCamera() {
    if (!formContext || inBuilder || starting) return;
    unlockAudio();
    stopCamera();
    errorMessage = "";
    success = false;
    qrCodeValue = "";
    lastScanValue = null;
    fieldApi?.setValue(qrCodeValue);

    if (!navigator.mediaDevices?.getUserMedia) {
      errorMessage =
        "Camera access is not available. The page must be served over HTTPS.";
      return;
    }

    starting = true;
    try {
      try {
        stream = await navigator.mediaDevices.getUserMedia({
          video: videoConstraints(true),
          audio: false,
        });
      } catch (e) {
        // The remembered camera may be gone - fall back to the preferred one
        if (!cameraId || e?.name !== "OverconstrainedError") throw e;
        cameraId = "";
        stream = await navigator.mediaDevices.getUserMedia({
          video: videoConstraints(false),
          audio: false,
        });
      }

      track = stream.getVideoTracks()[0];
      const capabilities = track?.getCapabilities?.() || {};
      hardwareZoom = capabilities.zoom?.max > capabilities.zoom?.min ? capabilities.zoom : null;
      torchSupported = !!capabilities.torch;
      torchOn = false;

      scanning = true;
      await tick();
      video.srcObject = stream;
      await video.play();

      const activeId = track?.getSettings?.().deviceId;
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
    stream?.getTracks().forEach((t) => t.stop());
    stream = null;
    track = null;
    torchOn = false;
    if (video) video.srcObject = null;
  }

  function onCameraChange(e) {
    cameraId = e.target.value;
    storeCamera(cameraId);
    if (scanning) startCamera();
  }

  function onZoomInput(e) {
    zoomLevel = clampZoom(e.target.value);
  }

  function chooseFile() {
    unlockAudio();
    fileInput.click();
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
    if (!continuousScan) stopCamera();
    qrCodeValue = decodedText;
    fieldApi?.setValue(qrCodeValue);
    success = true;
    if (soundOnScan) playScanSound();
    if (vibrateOnScan) navigator.vibrate?.(100);
    onScan?.({ value: decodedText });
  }

  onMount(() => {
    if (autoStartCamera) startCamera();
  });

  onDestroy(() => {
    stopCamera();
    audioContext?.close?.();
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
            <div
              class="video-wrapper"
              bind:this={videoWrapper}
              style={`aspect-ratio: ${videoAspect}; max-height: ${videoHeightLimit}vh;`}
            >
              <!-- Digital zoom enlarges the video box instead of using a CSS
                   transform, which iOS Safari does not clip to the wrapper -->
              <!-- svelte-ignore a11y-media-has-caption -->
              <video
                bind:this={video}
                muted
                playsinline
                webkit-playsinline
                on:loadedmetadata={updateVideoAspect}
                on:resize={updateVideoAspect}
                style={`width: ${digitalZoom * 100}%; height: ${digitalZoom * 100}%; left: ${((1 - digitalZoom) / 2) * 100}%; top: ${((1 - digitalZoom) / 2) * 100}%;`}
              ></video>
              {#if scannerBox}
                <div
                  class="scanner-box"
                  style={`width: ${scannerBoxWidth || 250}px; height: ${scannerBoxHeight || 250}px;`}
                ></div>
              {/if}
            </div>
            {#if showZoomControl}
              <label class="zoom">
                Zoom
                <input
                  type="range"
                  min="1"
                  max={MAX_ZOOM}
                  step="0.1"
                  value={zoomLevel}
                  on:input={onZoomInput}
                />
                <span>{zoomLevel.toFixed(1)}x</span>
              </label>
            {/if}
          {/if}

          {#if success && showResult !== false}
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
              {#if showTorchButton && torchSupported}
                <button type="button" on:click={toggleTorch}>
                  {torchOn ? "Flashlight Off" : "Flashlight On"}
                </button>
              {/if}
              <button type="button" on:click={stopCamera}>Stop Scanning</button>
            {:else}
              <button type="button" on:click={startCamera} disabled={starting}>
                {success ? "Rescan?" : "Start Scanning"}
              </button>
            {/if}
            {#if allowFileScan}
              <button type="button" on:click={chooseFile}>
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
    /* Clip the video and scanner box shade inside the wrapper; clip-path also
       clips the separately composited video layer on iOS Safari */
    clip-path: inset(0);
    isolation: isolate;
    background: black;
  }
  video {
    position: absolute;
    display: block;
    max-width: none;
    max-height: none;
    object-fit: cover;
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
  .zoom,
  .controls,
  .result,
  .error {
    position: relative;
    z-index: 1;
  }
  .zoom {
    display: flex;
    align-items: center;
    gap: 8px;
    width: 100%;
    max-width: 400px;
  }
  .zoom input {
    flex: 1;
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
