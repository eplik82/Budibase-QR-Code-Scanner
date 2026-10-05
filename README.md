# QR-Code-Scanner
This is a readme for your new Budibase plugin.

# Description
A component that scans QR codes, including inverted (white on black) codes.

Version 2.0.0 is built for Svelte 5 (Budibase 3.x plugin format, `svelteMajor: 5`).
Scanning uses [jsQR](https://github.com/cozmo/jsQR) instead of html5-qrcode, because html5-qrcode cannot read inverted codes.

![Budibase QR Code Scanner Sample](https://github.com/chungchunwang/Budibase-QR-Code-Scanner/blob/0789bd49723bef58f673086af4ae88d702397f4c/assets/Budibase%20QR%20Code%20Scanner%20Plugin%20Demo%20GIF.gif)


Find out more about [Budibase](https://github.com/Budibase/budibase).

## Options
Option | Description |
|---|---|
| Field | The form field the scanned value is written to.|
| Label | What the field is called.|
| On Scan | Actions to run after each scan. The scanned text is available as the `Scanned Value` binding.|
| Code Colors | Which code colors to look for: both dark on light and light on dark (default), dark on light only (slightly faster), or light on dark first.|
| Auto Start Camera | Start the camera as soon as the page loads (the browser asks for permission the first time).|
| Continuous Scanning | Keep the camera running after a scan, so several codes can be scanned in a row.|
| Same Code Rescan Delay (ms) | In continuous mode, the same code is only scanned again after it has been out of view for this long.|
| Show Scanned Result | Show the scanned text under the camera.|
| FPS | How many frames per second are scanned.|
| Preferred Camera | Back or front camera. The camera can also be changed while scanning; the last one used is remembered.|
| Camera Resolution | Standard, HD or Full HD. Higher resolution helps with small codes.|
| Zoom Level | Starting zoom (1-8). Uses the camera's own zoom when available (mostly phones), otherwise digital zoom.|
| Show Zoom Slider | Let the user change the zoom while scanning.|
| Show Flashlight Button | Show a flashlight button when the camera supports it (mostly phones).|
| Play Sound On Scan | Play a sound after a successful scan.|
| Sound | Beep, double beep or chime.|
| Sound Volume (0-100) | Volume of the scan sound.|
| Vibrate On Scan | Vibrate after a successful scan (Android devices).|
| Allow Scanning From Image File | Show a button to scan a QR code from an image file.|
| Scanner Box | Draw a box in the middle of the camera view; only codes inside it are scanned (on by default). The box always stays inside the camera view.|
| Scanner Box Width | The width of the scanner box.|
| Scanner Box Height | The height of the scanner box.|
| Camera Height (% of screen) | Height of the camera view as a share of the screen height (default 60). The camera picture is cropped to fill it.|

For small codes, use Full HD resolution and a zoom level of 2-4.

The buttons and zoom slider are shown above the camera view. On Budibase 3 grid screens the camera view shrinks to fit the component's cell, so make the cell taller in the builder for a bigger camera view. In the builder the component shows a dashed placeholder as tall as the camera view, which shows how tall to make the cell.

## Instructions

**Svelte version:** plugins run on Budibase's own Svelte runtime, and Svelte's internal API changes between minor versions. The plugin must therefore be built with exactly the Svelte version Budibase ships (5.40.2 for Budibase 3.4x, see `svelte` in Budibase's root `package.json`). A mismatch can freeze the browser when the component renders. If Budibase updates Svelte, update the pinned version in `package.json` and rebuild.

The camera only works over HTTPS (or on localhost) and is disabled in the builder preview.

To build the plugin run the following in your Budibase CLI:
```
budi plugins --build
```

You can also re-build everytime you make a change to your plugin with the command:
```
budi plugins --watch
```

