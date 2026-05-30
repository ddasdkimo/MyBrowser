# Component: Stream Recorder (canvas → MP4)

> Detect which video streams are being watched on a page, and record a chosen one to a
> video file. Built for canvas-rendered streams like the iSeek MJPEG monitoring wall.
> Verified working in the POC (see Changelog).

---

## Purpose

Many monitoring/streaming web apps draw live video onto `<canvas>` (so they can overlay
detection boxes, timestamps, etc.) instead of using `<video>`. This component lets the
browser shell:

1. Enumerate the active canvas streams on the current page and their channel labels.
2. Capture a chosen stream and save it as a video file — **MP4 when the platform can
   encode H.264, otherwise WebM** (with an optional ffmpeg transcode to guarantee MP4).

The captured frames include whatever is drawn on the canvas, so **AI overlay boxes are
recorded too** — no extra compositing needed.

## Applies to which approaches

- **Electron** — this doc (uses a `<webview>` preload).
- CEF — same idea via a render-process script + IPC; not yet written.
- Not applicable as-is to system-WebView shells (no equivalent preload-into-page hook).

## How (implementation)

Three pieces, connected by IPC:

**1. A preload injected into the browsed page (`webview-preload.js`)** — runs in the
page's context, so it can touch the site's canvases:

- Attach it via the host window's `will-attach-webview`:
  ```js
  win.webContents.on('will-attach-webview', (_e, webPreferences) => {
    webPreferences.preload = path.join(__dirname, 'webview-preload.js');
  });
  ```
  (Host window needs `webPreferences.webviewTag: true`.)
- **Enumerate**: `document.querySelectorAll('canvas')`, skip tiny ones
  (`w < 120 || h < 120` filters out icons/sparklines), tag each with a stable
  `data-mb-id`, and find its label by climbing parents and matching the nearest
  channel title (for iSeek: regex `/【[^】]+】/`).
- **Record**: `canvas.captureStream(15)` → `new MediaRecorder(stream, { mimeType })`.
  Pick the mime by preference (see below). Collect `dataavailable` chunks; on `stop`,
  build a `Blob`, read `arrayBuffer()`, and `ipcRenderer.send('recording:save', {label, mime, data})`.
- Talk to the host UI with `ipcRenderer.sendToHost(...)` (list, started, error).

**2. The main process** — receives the bytes and writes the file:

- `ipcMain.on('recording:save', ...)` → `fs.writeFileSync` into
  `~/Downloads/MyBrowser-Recordings/<label>_<timestamp>.<ext>`. Ext = `mp4` if the mime
  contains `mp4`, else `webm`.
- Optionally reveal in Finder via `shell.showItemInFolder`.

**3. The toolbar UI (host renderer)** — a panel listing streams with Record/Stop per row;
sends `record:start` / `record:stop` to the webview via `view.send(...)`, receives the
list via the `<webview>`'s `ipc-message` event, and the saved-file notice via a
contextBridge from the host preload.

### MIME selection (the MP4 question)

```js
const prefer = ['video/mp4;codecs=avc1','video/mp4',
                'video/webm;codecs=vp9','video/webm;codecs=vp8','video/webm'];
const mime = prefer.find(m => MediaRecorder.isTypeSupported(m)) || '';
```

> **Verified fact**: on **macOS + Electron 33 (Chromium 130)**, `video/mp4;codecs=avc1`
> **is supported** — Chromium uses the OS VideoToolbox hardware H.264 encoder, so
> `MediaRecorder` writes a real `.mp4` directly. **No ffmpeg needed on macOS.**

## Interface contract

- **Host UI → webview preload** (`view.send`): `streams:query`,
  `record:start {id,label}`, `record:stop`.
- **webview preload → host UI** (`sendToHost`, read via `ipc-message`):
  `streams:list [{id,label,w,h}]`, `record:started {id,mime}`, `record:error {id,message}`.
- **webview preload → main** (`ipcRenderer.send`): `recording:save {label,mime,data:Uint8Array}`.
- **main → host UI**: `recording:saved {file,ext,mime}` or `{error}`.

## Scope / Boundaries ★ (required — read before implementing)

- ✅ Records any **canvas-rendered** stream, including its on-canvas overlays
  (AI boxes, timestamps).
- ✅ Outputs real MP4 on macOS with zero extra dependencies.
- ✅ Works regardless of the underlying transport (MJPEG/WebSocket/etc.) — it captures
  the rendered canvas, not the network stream.
- ❌ Does **not** capture `<video>`-element streams (HLS/WebRTC/MP4 `<video>`). Those
  need a different path: capture via `video.captureStream()` instead of canvas, or grab
  the manifest. Detect by also enumerating `<video>` and branching.
- ❌ Does **not** capture overlays that live in a **separate DOM layer** above the canvas
  (e.g. HTML-positioned boxes). For iSeek this is fine (boxes are drawn on the canvas).
- ⚠️ **MP4 is not guaranteed off macOS.** On Linux/Windows Electron builds without an
  H.264 encoder, `isTypeSupported('video/mp4')` is false → it records WebM. To still
  deliver `.mp4`, add the ffmpeg fallback below.
- ⚠️ One recording at a time in this version (single `MediaRecorder` global). Multi-stream
  concurrent recording is a future extension.

## Dependencies

- **None required on macOS** — MP4 comes from the bundled Chromium + OS encoder.
- *(Fallback, only if MP4 unsupported on the target OS)* `ffmpeg-static` —
  spawn it in the main process to transcode the recorded WebM to MP4
  (`ffmpeg -i in.webm -c:v libx264 -pix_fmt yuv420p out.mp4`), then delete the WebM.
  Note libx264 is GPL and H.264 carries patents — see `decisions/02-licensing.md`.
  *(Cold agents: if you had to add this, record the exact version + OS here.)*

## Changelog

- `2026-05-30` — Initial implementation, verified on macOS / Electron 33.4.11. Recorded
  the iSeek monitoring wall (channels `ch3`, `前端測試工安的PPE`) to valid MP4
  (`ISO Media, MP4 Base Media v1`) directly via `MediaRecorder`, with AI overlay boxes
  included. No ffmpeg needed on macOS.
