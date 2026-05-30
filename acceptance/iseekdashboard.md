# Acceptance Checklist: iSeek Monitoring Dashboard

> Target: `https://iseekdashboard.intemotech.com/`
> Technical background: `decisions/03-iseekdashboard-recon.md`.
>
> This is the **objective definition of "runs perfectly."** A cold agent that built a
> browser must go through every item below and **actually test** it, not assume.
> Every box ticked = the docs successfully reproduced a working browser for this target.

---

## Prerequisites

- The user provides valid login credentials, OR an already-logged-in session.
- The agent does **not** enter passwords on the user's behalf — direct the user to log
  in themselves (this is a safety rule, not a doc limitation).

## Checklist

### A. Foundation

- [ ] **A1** The browser launches and loads the SPA (`#/login` then redirects into the
  admin app).
- [ ] **A2** After the user logs in, `localStorage.serviceToken` is written.
- [ ] **A3** `serviceToken` **persists across a full browser restart** — the user is
  still logged in after relaunch (does not bounce back to `#/login`).

### B. Core: Smart Monitoring (the heavy part)

- [ ] **B1** The Smart Monitoring page (`#/admin/cctvlist`) loads the camera grid.
- [ ] **B2** **All visible MJPEG channels play simultaneously** and update smoothly —
  not just the first one. Scroll the grid and confirm lower channels also stream.
- [ ] **B3** AI detection boxes (e.g. the green bounding box on the PPE/forklift channel)
  **overlay correctly on the canvas**, aligned with the video.
- [ ] **B4** Parallel connections across multiple `iseekmjpeg*` subdomains all succeed —
  no channels stuck black due to a connection-limit choke.
- [ ] **B5** Leave the page open for a few minutes — streams stay alive, no progressive
  freezing or runaway memory growth.

### C. REST features

- [ ] **C1** Subscription / My Plan pages load data via the API gateway.
- [ ] **C2** Notifications / Alert Query pages load and render records.
- [ ] **C3** Developer Zone (API key) page loads.

### D. General browsing (it must still be a real browser)

- [ ] **D1** Navigate to a general site (e.g. google.com) — loads and renders normally.
- [ ] **D2** A media-rich site (e.g. youtube.com) — page loads and the UI is usable.
      (Note: H.264 playback may depend on the build's codec support — see
      `decisions/02-licensing.md`. Record actual behavior here.)
- [ ] **D3** Basic navigation works: back/forward, reload, address entry.

## Pass / fail recording

When a cold agent runs this, record the result per item. Any failure is a **doc defect**:
note where it got stuck and what was missing, then feed it back per `AGENT.md` Step 5.

| Item | Result | Notes (what was missing if failed) |
|------|--------|------|
| A1 | ⬜ | |
| A2 | ⬜ | |
| A3 | ⬜ | |
| B1 | ⬜ | |
| B2 | ⬜ | |
| B3 | ⬜ | |
| B4 | ⬜ | |
| B5 | ⬜ | |
| C1 | ⬜ | |
| C2 | ⬜ | |
| C3 | ⬜ | |
| D1 | ⬜ | |
| D2 | ⬜ | |
| D3 | ⬜ | |
