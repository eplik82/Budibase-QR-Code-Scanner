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
| Label | What the field is called.|
| Code Colors | Which code colors to look for: both dark on light and light on dark (default), dark on light only (slightly faster), or light on dark first.|
| FPS | The frame rate of the scanner.|
| Auto Start Camera | Start the camera as soon as the page loads (the browser asks for permission the first time).|
| Allow Scanning From Image File | Show a button to scan a QR code from an image file.|
| Scanner Box | Whether to draw a box in the center of the screen that users will have to align QR codes inside of.|
| Scanner Box Width | The width of the scanner box.|
| Scanner Box Height | The height of the scanner box.|

## Instructions

The camera only works over HTTPS (or on localhost) and is disabled in the builder preview.

To build the plugin run the following in your Budibase CLI:
```
budi plugins --build
```

You can also re-build everytime you make a change to your plugin with the command:
```
budi plugins --watch
```

