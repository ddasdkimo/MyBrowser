# Component: Tabs (multi-tab shell)

> Turn the single-view shell into a multi-tab browser: a tab strip, new/close/switch,
> each tab its own `<webview>`. Composes with the stream recorder (background tabs keep
> recording).

---

## Purpose

A real browser has tabs. This upgrades `components/shell-ui` from one `<webview>` to many,
with a tab strip and the address bar / nav / panels operating on the active tab.

## Applies to which approaches

- **Electron** (with `<webview>`) — this doc. Builds directly on `components/shell-ui`.

## How (implementation)

**Model**: keep an array `tabs = [{ id, view, title, url }]` and an `activeId`.
`activeView()` returns the active tab's `<webview>`.

**DOM**: a `#tabsbar` strip (with a `＋` new-tab button) above the toolbar; all `<webview>`
elements live stacked in `#stage`; only the active one is visible
(`view.style.display = isActive ? '' : 'none'`) — **do not destroy/recreate** hidden
views, just hide them (so background tabs keep running, incl. recording).

**Create a tab**:
- `document.createElement('webview')`, set `partition="persist:main"` (shared session →
  login shared across tabs, like a normal browser), `allowpopups`.
- Append to `#stage`, push to `tabs`, wire per-view listeners (below), activate it.
- Load the URL (start URL for the first tab; new tabs can open a blank/start page).

**Per-view listeners (wire each view, not just the active one)** — because background
tabs still emit events (e.g. recording):
- `page-title-updated` → update that tab's title + re-render the strip.
- `did-navigate` / `did-navigate-in-page` → update that tab's `url`; if it's active, set
  the address bar; refresh back/forward enabled state.
- `did-start/stop-loading` → update the status text only when active.
- `ipc-message` → the stream-recorder messages (`streams:list`, `record:started`,
  `record:error`). Bind with the tab id in closure so you know which tab emitted it.
- `dom-ready` → if `src` is still `about:blank`, load the intended URL.

**Activate / close**:
- Activate: set `activeId`, toggle visibility, sync address bar + nav buttons, re-render
  strip, and if the streams panel is open re-query the now-active view.
- Close: remove the view element, splice from `tabs`; if it was active, activate a
  neighbour; if no tabs remain, create a fresh start tab (never leave zero tabs).

**Toolbar/panel rewire**: every place that referenced the single `view` now calls
`activeView()` (navigate, back/forward/reload, `streams:query`, `record:*`, `cred:fill`).

## Interface contract

- Exposes `activeView()` to the rest of the renderer; all view-directed actions go through it.
- Expects `components/shell-ui` (window, persistent partition, webview preload wiring) and,
  if present, the recorder/credential message channels — it just routes them per active view.

## Scope / Boundaries ★ (required — read before implementing)

- ✅ Standard tab UX: open/close/switch, per-tab title/url, shared login across tabs.
- ✅ Background tabs keep running (hidden, not destroyed) → recording continues when you
  switch away.
- ❌ No tab drag-reorder / pinning / tab restore in v1 (extensions, not core).
- ⚠️ Recorder + tabs interaction: the same canvas `id` can exist in two tabs. v1 keeps the
  streams panel acting on the **active tab** and tracks recordings globally by stream id —
  a known simplification (see stream-recorder boundaries). Stop a background tab's
  recording by switching to that tab. Log this limit; don't hide it.
- ⚠️ Many tabs each bundling a Chromium view is memory-heavy — fine for a few tabs, not
  hundreds.

## Dependencies

- None beyond Electron + `components/shell-ui`.

## Changelog

- `2026-05-30` — Spec written for cold-agent reproduction. (Not yet cold-validated.)
