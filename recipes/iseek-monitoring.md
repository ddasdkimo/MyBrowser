# Recipe: MJPEG CCTV Monitoring Browser

> Scenario: a desktop browser that runs a multi-channel MJPEG CCTV monitoring dashboard
> perfectly (live walls + AI overlay), while remaining a usable general-purpose browser.
> Reference target: the iSeek dashboard. Verified by `acceptance/iseekdashboard.md`.

---

## Chosen approach

**Electron desktop shell.** Rationale: the target is a standard Vue SPA streaming MJPEG
into Canvas 2D — no engine-internal changes needed, and a full bundled Chromium gives the
most predictable compatibility for many concurrent long-lived connections. No fork, no
system WebView. (See `decisions/01-approach-tree.md` and
`decisions/03-iseekdashboard-recon.md`.)

## Components to assemble (in order)

1. **`components/shell-ui/` (Electron desktop shell)**
   - Persistent session partition (`persist:main`) so `localStorage.serviceToken`
     survives restart.
   - Keep Chromium default networking — the dashboard already shards streams across
     `iseekmjpeg*` subdomains to bypass the per-host 6-connection limit.

2. **`components/stream-recorder/` (canvas → MP4)** *(optional feature)*
   - Detects the canvas-rendered MJPEG channels and records a chosen one.
   - On macOS, `MediaRecorder` outputs real MP4 directly (no ffmpeg). Recorded video
     includes the AI overlay boxes since they're drawn on the canvas.

3. *(optional, only if needed)* network / header tweaks — **avoid** unless an acceptance
   item fails. Aggressive connection caps or a merging proxy can choke the MJPEG wall.

## Key gotchas (from recon)

- **Login persistence** depends entirely on localStorage surviving restart → use a
  persistent partition, never clear storage on close. (Acceptance A2/A3.)
- **All channels must stream**, not just the first → test scrolled-in channels too.
  (Acceptance B2/B4.)
- **No codec licensing concern** for the dashboard itself (MJPEG/JPEG is patent-free).
  General-web H.264 playback is a separate question — see `decisions/02-licensing.md`.

## Verify

Run every item in [`acceptance/iseekdashboard.md`](../acceptance/iseekdashboard.md).
Any failure → diagnose, fix, and **write the fix back into the relevant component doc**
(`AGENT.md` Step 5) so the next cold agent inherits it.
