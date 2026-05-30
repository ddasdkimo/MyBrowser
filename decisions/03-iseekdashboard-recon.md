# Technical Recon: iSeek Monitoring Dashboard

> Recorded live technical recon of the first acceptance target
> `https://iseekdashboard.intemotech.com/`.
> The matching checkable acceptance list is in `acceptance/iseekdashboard.md`.
> Recon date: 2026-05-30.

---

## One-line summary

This is a **Vue SPA multi-channel CCTV live-monitoring platform**: video uses
**MJPEG → Canvas 2D** rendering, AI detection is done on the backend
(NVIDIA DeepStream), and the login state lives in localStorage.
**Technically very "well-behaved" — any Chromium shell runs it perfectly, with no
codec patent issues.**

## Tech stack

| Layer | Recon result | Meaning for the shell |
|------|---------|------|
| Frontend framework | Vue SPA + Vite build (single `index-[hash].js`, hash routing `#/`, mounts on `#app`) | Pure standard frontend, any Chromium shell runs it |
| Login state | JWT stored in `localStorage.serviceToken` | Shell only needs to preserve localStorage to keep login |
| API | REST via the `apigatewayiseek.intemotech.com` gateway, Bearer token | Standard fetch/XHR |
| Video transport | **MJPEG**, from multiple edge servers `iseekmjpeg3 / 7 / 8.intemotech.com` | See "Key architecture detail" below |
| Video rendering | Fetch MJPEG frames → draw to `<canvas>` 2D (N channels = N canvases) | Only needs Canvas 2D, universal |
| AI detection | Done on backend NVIDIA DeepStream (`/deepstream_manage/...`); frontend only receives results and overlays boxes | Zero AI compute load on the frontend |
| Codec | No H.264 / HLS / WebRTC main path (WebCodecs available but not required) | **Completely avoids codec patents** (JPEG is patent-free) |

## Main feature areas (nav bar)

- Smart Monitoring (`#/admin/cctvlist`) — core, multi-channel CCTV wall + AI detection boxes
- Subscription / My Plan / Notifications / Alert Query / Developer Zone (API key) /
  Analysis / Account / User Guide

## Key architecture detail (must factor into implementation)

### Multiple MJPEG subdomains = bypassing the browser connection limit

The monitoring wall opens a dozen-plus MJPEG channels at once. **MJPEG is a long-lived
connection** (each channel holds one HTTP connection open). Chromium defaults to **at
most 6 concurrent connections per host**. This site deliberately spreads streams across
multiple subdomains `iseekmjpeg3 / 7 / 8...` precisely to bypass that 6-connection limit.

> **Meaning for the shell**:
> - If the shell keeps Chromium's default connection policy → fine, the site already
>   shards.
> - But if the shell adds aggressive connection limits or domain-merging proxies →
>   it may choke the streams; watch out.
> - Acceptance must actually test "are all N channels alive simultaneously," not just
>   the first one.

### Login-state fragility

Login relies on `localStorage.serviceToken`. If the shell clears localStorage on close,
or uses an isolated storage partition, the user must re-login every time. **Acceptance
must confirm the token persists across launches.**

## Impact on approach choice

- Need to fork Chromium? → **No.** No engine-internal changes required.
- Desktop approach? → **Electron or CEF** (Chromium core, most stable compatibility).
- Codec / Google service licensing? → **Neither is used in this scenario**, license
  risk is low.
- Only compatibility concern → concurrency and memory of many long-lived MJPEG connections.

## Recon method (for future reproduction)

Run via browser automation in an already-logged-in tab:
- `performance.getEntriesByType('resource')` to get origins → discovered MJPEG
  subdomains and the API gateway.
- DOM queries: 0 `<video>`, 12 `<canvas>` (2D context, with drawn pixels) → confirmed
  canvas rendering.
- `localStorage` keys → discovered the `serviceToken` login mechanism.
