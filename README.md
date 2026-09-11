# Capture Attendance

A lightweight, mobile-friendly attendance photo capture page built with plain **HTML, CSS, and JavaScript**.

The page opens the device camera, allows the user to capture an attendance photo, preview/retake it, or upload an existing image from the device. Once confirmed, the image is sent back to the parent attendance portal using `window.postMessage()`.

## Features

- 📷 Live camera access using the browser `MediaDevices.getUserMedia()` API
- 🤳 Front-camera and rear-camera switching
- ⚡ Photo capture directly from the camera
- 👀 Captured-photo preview
- ↩ Retake captured photo
- 📁 Upload an existing image from the device/gallery
- 🔒 Camera stream is stopped after capture to reduce battery usage
- ⏳ Loading states while starting the camera and processing/submitting images
- ❌ Camera-permission/error state with upload fallback
- ✅ Success state after the photo is submitted
- 📱 Responsive/mobile-friendly UI
- 🔄 Communication with the parent attendance portal through `postMessage`
- 💾 `sessionStorage` fallback when the parent window/opener is unavailable

## Technology

This project does not require a build system or framework.

- HTML5
- CSS3
- Vanilla JavaScript
- Browser Camera API (`navigator.mediaDevices.getUserMedia`)
- HTML Canvas API
- `FileReader` API
- `window.postMessage`
- `sessionStorage`

## Project Structure

```text
capture-attendance/
└── index.html
```

The complete application is contained in a single HTML file.

## How It Works

### 1. Camera Initialization

When the page loads, the application calls:

```javascript
navigator.mediaDevices.getUserMedia({
  video: {
    facingMode: facingMode,
    width: { ideal: 1280 },
    height: { ideal: 960 }
  },
  audio: false
});
```

The default camera mode is the front-facing camera.

```javascript
let facingMode = 'user';
```

### 2. Switch Camera

The **Switch Camera** button changes between:

- `user` → Front camera
- `environment` → Rear camera

The current camera stream is stopped before a new stream is started.

### 3. Capture Photo

When the user clicks **Capture Photo**:

1. The current video frame is drawn onto a hidden `<canvas>`.
2. The canvas is converted into a JPEG data URL.
3. The camera stream is stopped.
4. The captured image is displayed in the preview screen.

The image is generated using:

```javascript
canvas.toDataURL('image/jpeg', 0.88);
```

### 4. Retake

Selecting **Retake** clears the previously captured image and starts the camera again.

### 5. Gallery Upload

If camera access is unavailable, or the user prefers an existing image, they can upload an image from the device.

The current implementation accepts:

```html
<input type="file" accept="image/*">
```

A maximum file size of **5 MB** is enforced.

### 6. Submit Photo

When the user clicks **Use This Photo**, the application creates the following message:

```javascript
{
  type: 'FP_ATTENDANCE_PHOTO',
  image: capturedData,
  ts: Date.now()
}
```

The message is sent to the parent/opener window:

```javascript
window.opener.postMessage(message, allowedOrigin);
```

This allows the main attendance portal to receive and process the captured image.

## Parent Portal Integration

The application is designed to be opened from another attendance application, such as a Google Apps Script web app.

The parent page should listen for the message:

```javascript
window.addEventListener('message', (event) => {
  if (event.data?.type === 'FP_ATTENDANCE_PHOTO') {
    const image = event.data.image;

    // Process the attendance photo here
  }
});
```

The received object contains:

| Property | Description |
|---|---|
| `type` | Identifies the attendance photo message |
| `image` | Base64/Data URL representation of the captured image |
| `ts` | Timestamp generated when the image is submitted |

## Allowed Origin

The capture page supports an `origin` query parameter.

Example:

```text
index.html?origin=https%3A%2F%2Fexample.com
```

The page reads the parameter and uses it as the target origin for `postMessage()`.

```javascript
const params = new URLSearchParams(window.location.search);

if (params.get('origin')) {
  allowedOrigin = decodeURIComponent(params.get('origin'));
}
```

### Important Security Note

The current source defaults to:

```javascript
let allowedOrigin = '*';
```

For production use, the parent application should always provide a trusted origin, and the parent listener should validate:

```javascript
event.origin
```

before accepting the image.

## Fallback Behavior

If the capture page was not opened by another window, the application attempts to store the submitted message in `sessionStorage`:

```javascript
sessionStorage.setItem(
  'fp_attendance_photo',
  JSON.stringify(message)
);
```

The parent attendance application can poll/read this value when applicable.

## Browser Permissions

Camera access requires browser permission.

If permission is denied or camera access fails, the application displays:

> Camera Access Denied

The user can then upload a photo from the device instead.

For camera access, the page should normally be served from a **secure context**, such as:

- HTTPS
- `localhost` during development

## Running Locally

Because this is a standalone HTML page, there is no `npm install` or build step.

You can open the HTML file directly for basic UI testing:

```text
index.html
```

For camera testing, it is recommended to serve the project through a local HTTP server.

For example, with Python:

```bash
python -m http.server 8000
```

Then open:

```text
http://localhost:8000
```

## Deployment

The application can be hosted as a static HTML page on any platform that supports static files.

Examples include:

- GitHub Pages
- Netlify
- Vercel
- Cloudflare Pages
- Any HTTPS web server

The deployed page URL can then be opened by the main attendance application.

## Image Processing

Camera images are captured at the video's available resolution, with an intended camera resolution of:

```text
1280 × 960
```

The captured image is encoded as:

```text
JPEG
Quality: 0.88
```

The image remains in memory as a Base64/Data URL until it is submitted.

## UI States

The application has four primary states:

### Camera

Displays the live camera feed and capture controls.

### Preview

Displays the captured photo with:

- Retake
- Use This Photo

### Error

Displayed when camera access fails.

Provides an option to upload a photo manually.

### Success

Displayed after the image has been sent to the parent application.

The page then attempts to close itself after approximately 2 seconds.

## Important Implementation Details

### Camera Stream Cleanup

The application stops active media tracks when switching cameras or after capturing:

```javascript
stream.getTracks().forEach(t => t.stop());
```

This prevents the camera from remaining active unnecessarily.

### Front Camera Mirroring

Photos captured using the front camera are horizontally mirrored on the canvas so that the result appears natural to the user.

### File Size Validation

Uploaded images larger than 5 MB are rejected.

## Production Recommendations

Before using this in a production attendance system, consider adding:

1. Strict `postMessage` origin validation.
2. Validation of `event.origin` on the parent page.
3. Server-side image validation.
4. Authentication/authorization for attendance submissions.
5. Server-side timestamp validation if the timestamp is used for attendance records.
6. Face detection/verification if biometric attendance is required.
7. Image compression/resizing before transmission if large images cause payload issues.
8. HTTPS deployment.
9. Clear privacy/consent messaging for employee photographs.
10. Backend-side protection against duplicate or replayed attendance submissions.

## Current Limitations

- No backend is included in this page.
- No face detection or face recognition is implemented.
- The captured image is transferred as a Base64 data URL.
- Attendance database storage must be handled by the parent application/backend.
- Camera behavior depends on browser/device permissions and capabilities.
- `sessionStorage` fallback is limited to the same browser origin/context.

## License

Add the project's applicable license here if this application is intended for distribution or external use.
