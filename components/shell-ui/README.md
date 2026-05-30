# Component: Desktop Shell (Electron)

> The minimal desktop browser shell. This is the foundation every desktop vertical
> scenario builds on. Default approach: **Electron** (see
> `decisions/01-approach-tree.md` for why).

---

## Purpose

Provide a window with a Chromium-backed web view plus the minimum chrome a browser needs
(address bar, back/forward/reload, persistent profile). This is the smallest thing that
is recognizably "a browser."

## Applies to which approaches

- **Electron** — this doc.
- CEF — same concepts, different host language; a separate component doc covers it.
- Not for WebView/Tauri (those have their own shell docs).

## How (implementation)

Minimal Electron structure:

```
my-browser/
  package.json        # "main": "main.js", electron as devDependency
  main.js             # main process: creates BrowserWindow, owns the profile
  preload.js          # safe bridge (contextIsolation on)
  ui/                 # optional: your own toolbar UI (address bar, nav buttons)
```

Core of `main.js`:
- Create a `BrowserWindow` with a `BrowserView` (or `<webview>`) that loads the target.
- Set a **persistent `partition`** (e.g. `persist:main`) on the session so cookies and
  **localStorage survive restart** — critical for token-based logins.
- Keep Chromium defaults for networking unless you have a reason to change them.

Navigation: wire toolbar buttons to `view.webContents.goBack()` / `goForward()` /
`reload()` / `loadURL()`.

Minimal acceptance target: launch → load a URL → log in → restart → still logged in →
general sites browse normally.

## Interface contract

- **Exposes**: a `loadURL(url)` entry, a persistent session partition name, and hooks
  for other components (e.g. network-intercept attaches to `session.webRequest`).
- **Expects**: the chosen target URL(s) and an acceptance checklist to verify against.

## Scope / Boundaries ★ (required — read before implementing)

- ✅ Runs any standard web app with full Chromium compatibility (Canvas, WebSocket,
  fetch, localStorage, many concurrent connections).
- ✅ Persists login state across restarts via a persistent partition.
- ❌ Does **not** modify engine internals (packet-layer interception, fingerprint
  spoofing). If the user needs that, this component is the wrong layer — escalate to a
  fork decision.
- ❌ Does **not** by itself guarantee H.264/H.265 playback — that depends on the Electron
  build's bundled codecs. Verify per `decisions/02-licensing.md` if the target needs it.
- ⚠️ Do not add aggressive per-host connection caps or a domain-merging proxy — it can
  choke multi-subdomain MJPEG walls like `acceptance/iseekdashboard.md` (see B4).

## Dependencies

- `electron` — the runtime + bundled Chromium. Pin a specific major version for
  reproducibility. Source: npm `electron`. Scope: desktop only (Mac/Win/Linux).
  - **Verified: `electron@33.4.11`** (range `^33.0.0`) on macOS — passed the
    iseekdashboard core acceptance (multi-channel MJPEG wall + AI overlay rendered
    and ran normally). Node 22.17.1 / npm 10.9.2.

## Minimal file layout that passed (reference)

```
package.json   # "main": "main.js", devDependency electron ^33
main.js        # BrowserWindow with webviewTag:true; touch session.fromPartition('persist:main')
preload.js     # contextIsolation bridge (no privileged API needed for POC)
ui/index.html  # toolbar (back/forward/reload + address) + <webview partition="persist:main">
ui/renderer.js # wires toolbar to webview nav events; URL/search normalization
```

Key points that mattered:
- `<webview>` with `partition="persist:main"` → login (localStorage `serviceToken`)
  persists; do not clear storage on close.
- `webviewTag: true` in the host window's `webPreferences` is required to use `<webview>`.
- Left default Chromium networking → multi-subdomain MJPEG wall streamed fine.

## Changelog

- `2026-05-30` — Initial Electron desktop shell component, written to support the
  iseekdashboard POC.
- `2026-05-30` — **Architect (HOT) build verified on macOS** with `electron@33.4.11`:
  the iSeek dashboard runs with normal function. NOTE: this was built by the architect
  with full conversation context, **not** a context-free cold agent — so it validates
  the *design*, not yet the *doc completeness*. Cold-start validation is pending.
