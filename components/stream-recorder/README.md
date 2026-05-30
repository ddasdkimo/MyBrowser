# Component: Stream Recorder (canvas / video → MP4)

> Detect which video streams are on a page, and record one or more of them to MP4 —
> including unattended background recording. Built for canvas-rendered streams (iSeek
> MJPEG wall) and `<video>` elements (e.g. YouTube).

---

## Purpose

Many web apps draw live video onto `<canvas>` (to overlay detection boxes/timestamps);
others use `<video>` (MSE). This component lets the browser shell:

1. Enumerate active canvas + video streams on the current page and their labels.
2. Record one **or several concurrently**, each to its own continuous MP4 file.
3. Keep recording **in the background** when the window is hidden/minimized, without the
   user babysitting it, and prevent the machine from sleeping mid-recording.

Captured frames are whatever is rendered, so **on-canvas overlays (AI boxes) are
recorded**. `<video>` capture includes **audio**.

## Applies to which approaches

- **Electron** — this doc (uses a `<webview>` preload).
- CEF — same idea via a render-process script + IPC; not yet written.

## Architecture (3 pieces + IPC)

```
[webview preload]  enumerate + MediaRecorder per stream  ──save bytes──▶ [main]  write file
   (page context)  ◀──start/stop/fill commands── [host renderer UI]  ◀──saved──┘
```

**1. Webview preload (`webview-preload.js`)** — injected into the browsed page via the
host window's `will-attach-webview` (host needs `webviewTag: true`):

- **Enumerate**:
  - canvases: `document.querySelectorAll('canvas')`, skip `w<120||h<120`; label by
    climbing parents matching `/【[^】]+】/` (iSeek channel titles).
  - videos: `document.querySelectorAll('video')`, require `videoWidth>=120` (i.e. actually
    playing); label from `title` / `aria-label` / nearest `h1` / `document.title`.
  - tag each element with a stable `data-mb-id`; return `{id, kind, label, w, h}`.
- **Record** (support MANY at once — keep a `Map<id, {recorder, chunks, label}>`, NOT a
  single global recorder):
  - canvas → `el.captureStream(15)`; video → `el.captureStream()` (includes audio;
    if `getVideoTracks().length === 0`, it's DRM/EME → report an error, don't proceed).
  - `new MediaRecorder(stream, { mimeType })`; `recorder.start(2000)` (timeslice so a
    crash loses ≤2 s).
  - on `stop`: build `Blob`, `arrayBuffer()` → `Uint8Array`, send to main with the `id`.
- **Commands from host** (via `ipcRenderer.on`): `record:start {id,label}`,
  `record:stop {id}`, `record:stopAll`, `streams:query`.
- **To host** (via `ipcRenderer.sendToHost`, read on the `<webview>`'s `ipc-message`):
  `streams:list {items, recording:[ids]}`, `record:started {id,mime}`, `record:error {id,message}`.

**2. Main process**:
- `ipcMain.on('recording:save', ...)` → `fs.writeFileSync` into
  `~/Downloads/MyBrowser-Recordings/<safeLabel>_<ISO-timestamp>.<ext>`
  (ext = `mp4` if mime contains `mp4`, else `webm`); echo `recording:saved {id,file}` back.
- **Background recording**: set `backgroundThrottling: false` on BOTH the host window's
  `webPreferences` AND each webview (in `will-attach-webview`). Without this, Chromium
  throttles hidden pages and the canvas stops painting → frozen recording.
- **Prevent sleep**: while any recording is active, hold a
  `powerSaveBlocker.start('prevent-app-suspension')`; release it when none active. The
  renderer signals active/idle via an IPC (`recording:active <bool>`).

**3. Host renderer UI** — a side panel listing streams, each with its own Record/Stop
(concurrent), a "Stop all", and a status line. Sends commands to the active webview via
`view.send(...)`; receives lists/started/errors via the `<webview>` `ipc-message` event;
receives the saved-file notice from main via a contextBridge in the host preload.

### MIME selection

```js
const prefer = ['video/mp4;codecs=avc1,mp4a.40.2','video/mp4;codecs=avc1','video/mp4',
                'video/webm;codecs=vp9,opus','video/webm;codecs=vp8,opus','video/webm'];
const mime = prefer.find(m => MediaRecorder.isTypeSupported(m)) || '';
```

> **Verified fact**: on **macOS + Electron 33 (Chromium 130)**, `video/mp4;codecs=avc1`
> is supported (VideoToolbox hardware H.264) → `MediaRecorder` writes real `.mp4`
> directly. **No ffmpeg needed on macOS.** (`file` reports `ISO Media, MP4 Base Media v1`.)

## Interface contract

- Host → webview (`view.send`): `streams:query`, `record:start {id,label}`,
  `record:stop {id}`, `record:stopAll`.
- Webview → host (`ipc-message`): `streams:list {items:[{id,kind,label,w,h}], recording:[id]}`,
  `record:started {id,mime}`, `record:error {id,message}`.
- Webview → main (`ipcRenderer.send`): `recording:save {id,label,mime,data:Uint8Array}`.
- Renderer → main: `recording:active <bool>`, `recording:reveal <file>`.
- Main → renderer (bridge): `recording:saved {id,file,ext,mime}` | `{id,error}`.

## Scope / Boundaries ★ (required — read before implementing)

- ✅ Records canvas streams (incl. on-canvas overlays) and `<video>` streams (incl. audio).
- ✅ Multiple concurrent recordings, each its own continuous file.
- ✅ Background/unattended recording (throttling off) + no-sleep while recording.
- ✅ Real MP4 on macOS with zero extra dependencies.
- ❌ **DRM/EME `<video>`** (e.g. paid movies) yields no frames — cannot and must not be
  circumvented. Detect via empty `getVideoTracks()` and report.
- ❌ Overlays in a **separate DOM layer** above a canvas are not captured (iSeek draws on
  the canvas, so it's fine).
- ⚠️ **Legal boundary**: recording is a general capture capability (like screen
  recording). For **third-party platforms (e.g. YouTube)** the user must only record
  content they have the right to (own/CC/fair-use); recording copyrighted streams can
  violate the platform's ToS and copyright. The user's OWN sites (e.g. iSeek monitoring)
  have no such limitation. **Do not build a stream/segment downloader** for offline
  ripping of third-party protected media — that crosses from "capture" into circumvention.
- ⚠️ MP4 is **not** guaranteed off macOS — Linux/Windows builds without an H.264 encoder
  fall back to WebM; add the ffmpeg transcode (below) to still deliver `.mp4`.
- ⚠️ With multiple tabs, the same canvas `id` (e.g. `mb-canvas-0`) can appear in two tabs.
  Either namespace recording keys by tab, or accept that the panel acts on the active tab
  (a known POC simplification — log it, don't hide it).

## Dependencies

- **None required on macOS.**
- *(Fallback, only if MP4 unsupported on target OS)* `ffmpeg-static` — in main, transcode
  WebM → MP4 (`ffmpeg -i in.webm -c:v libx264 -pix_fmt yuv420p out.mp4`), then delete the
  WebM. libx264 is GPL and H.264 carries patents — see `decisions/02-licensing.md`.

## Changelog

- `2026-05-30` — Architect (HOT) build verified on macOS / Electron 33.4.11: recorded the
  iSeek wall (channels `ch3`, `前端測試工安的PPE`) and a `<video>` to valid MP4, multiple
  at once, in background. Spec written for cold-agent reproduction. (Cold validation pending.)
