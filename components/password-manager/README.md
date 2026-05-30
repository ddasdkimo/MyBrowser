# Component: Password Manager (credential save / autofill)

> Let the browser remember logins and offer to fill them next time — stored encrypted via
> the OS keychain. A core browser feature, implemented within strict safety boundaries.

---

## Purpose

When the user logs in to a site, offer to save the credentials; on a later visit, offer to
fill them. The user stays in control: they submit the login themselves; the browser only
remembers and fills.

## Safety boundaries ★ (read FIRST — these are non-negotiable)

- **Never store plaintext.** Encrypt with Electron `safeStorage` (backed by the macOS
  Keychain / OS secret store). If `safeStorage.isEncryptionAvailable()` is false, **refuse
  to save** and tell the user — do not fall back to plaintext.
- **Capture only on a user-initiated submit.** Read the password from a form the user
  themselves submitted. Never scrape password fields proactively, and never read the
  user's existing browser/Chrome saved passwords.
- **Saving requires explicit user confirmation** in the shell UI ("Save password for X?").
- **Autofill requires an explicit user click** ("Fill"), and the browser **must not
  auto-submit** — the user presses login. (Filling a field ≠ logging in on their behalf.)
- **Password never passes through the host renderer UI.** It flows page→main (to encrypt)
  and main→page (to fill). The host UI only ever sees the host/username, never the secret.
- Scope credentials by **origin/host**; only offer fill on a matching host.

## How (implementation)

**Store (main process)** — file at `app.getPath('userData')/credentials.json`, mapping
`host → { username, password: <base64 of safeStorage.encryptString(pw)> }`:
- `ipcMain.on('cred:capture', {host,username,password})` → hold as `pendingCapture`, and
  `win.webContents.send('cred:prompt', {host,username})` (NO password to the renderer).
- `ipcMain.on('cred:save')` → if `safeStorage.isEncryptionAvailable()`, encrypt
  `pendingCapture.password`, write the record; else notify "encryption unavailable".
- `ipcMain.on('cred:dismiss')` → drop `pendingCapture`.
- `ipcMain.handle('cred:has', {host})` → `{username}` or `null` (no password).
- `ipcMain.handle('cred:get', {host})` → `{username, password}` decrypted — returned only
  to the **webview preload** that will fill the fields.

**Capture (webview preload, page context)**:
- Listen `window.addEventListener('submit', …, true)` (capture phase). If the submitted
  form has `input[type=password]` with a value, grab the password and a best-guess username
  (`input[autocomplete=username]`, `[type=email]`, `[type=text]`, `[name*=user i]`,
  `[name*=account i]`). `ipcRenderer.send('cred:capture', {host: location.host, username, password})`.
- *(Known gap)* Pure SPA logins (fetch on a button click, no real form submit) won't fire
  `submit`. iSeek is such an SPA — for it, login persistence already covers the need. A
  SPA-friendly hook (intercept the login button / fetch) is a future extension; record it
  as a gap rather than silently missing it.

**Autofill (webview preload)**:
- On load (and a few delayed retries for SPAs), if `input[type=password]` exists,
  `await ipcRenderer.invoke('cred:has', {host})`; if found,
  `ipcRenderer.sendToHost('cred:available', {host, username})`.
- On `cred:fill` from host: `await ipcRenderer.invoke('cred:get', {host})`, then set the
  username + password fields using a **native setter + input/change events** so SPA
  frameworks (Vue/React) register the value:
  ```js
  function setNativeValue(el, value) {
    const d = Object.getOwnPropertyDescriptor(Object.getPrototypeOf(el), 'value');
    d.set.call(el, value);
    el.dispatchEvent(new Event('input', { bubbles: true }));
    el.dispatchEvent(new Event('change', { bubbles: true }));
  }
  ```
  Do **not** submit the form.

**Host UI**: a thin credential bar.
- `cred:prompt` (from main, via host preload bridge) → "Save password for X? [Save][No]"
  → `cred:save` / `cred:dismiss`.
- `cred:available` (from the active webview's `ipc-message`) → "Saved login found.
  [Fill] [Dismiss]" → `view.send('cred:fill')` / just hide the bar. (Always give a way to
  dismiss the bar without filling.)

## Interface contract

- Page → main: `cred:capture {host,username,password}`; invoke `cred:has {host}`,
  `cred:get {host}`.
- Main → host (bridge): `cred:prompt {host,username}`.
- Host → main: `cred:save`, `cred:dismiss`.
- Host → page: `cred:fill`. Page → host: `cred:available {host,username}`.

## Scope / Boundaries ★

- ✅ Save/fill for standard HTML login forms, encrypted at rest via OS keychain.
- ❌ Does not read existing Chrome/Safari/Firefox saved passwords (out of scope by design).
- ❌ Does not auto-submit / log in on the user's behalf.
- ⚠️ SPA logins without a real form submit are not captured yet (see gap above).
- ⚠️ This is local convenience storage, not a hardened vault — no master password / sync in
  v1. Don't oversell it; `safeStorage` ties decryption to the OS user account.

## Dependencies

- Electron `safeStorage` (built in). On Linux it may need a working secret service
  (gnome-keyring/kwallet); if unavailable, saving is refused (by design).

## Changelog

- `2026-05-30` — Spec written for cold-agent reproduction. (Not yet built or validated.)
