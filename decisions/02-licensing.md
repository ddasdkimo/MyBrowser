# License & Patent Obligations Checklist

> The agent must walk this checklist before the "ship/package" stage.
> Most vertical scenarios won't hit landmines, but "include the license notices" is
> almost always required.

---

## Core concept: Chromium is not a "single license"

"Chromium is BSD" is only half true. The browser you actually package is a
**collection of hundreds of components**:

- **Chromium's own code** → BSD 3-Clause (very permissive, closed-source commercial OK).
- **Third-party dependencies** (`third_party/`) → licenses are all over the map: BSD,
  MIT, Apache 2.0, LGPL, MPL, a few GPL. This is what must be reviewed one by one.

Shell approaches (Electron / CEF) ship a pre-packaged, compliant Chromium for you,
so **most obligations become "correctly include the existing license notices" rather
than auditing hundreds of components yourself.**

## License obligations reference

| License type | Obligation | Risk | Common in |
|---------|------|:----:|------|
| BSD / MIT / Apache 2.0 | Retain copyright & license text | Low | Chromium core, most components |
| LGPL | Dynamic linking can stay closed-source; must let users replace the component | Med | Some audio/video libs |
| MPL 2.0 | Modifying that file requires open-sourcing it; does not infect your own files | Med | A few components |
| GPL | Infectious | High | Very few build tools (usually excludable at build time) |

## Checklist

### Always do

- [ ] **Include license notices.** Electron uses `electron-license` / built-in credits;
      self-built Chromium uses the `about:credits` generator. Put the output in your
      "About" page or installer.
- [ ] **Rename and replace the logo.** `"Chrome"`, `"Google"`, and the Chrome logo are
      **trademarks** and cannot be used. This is why a plain build is called Chromium,
      not Chrome. Give your browser its own name and icon.

### Conditionally

- [ ] **Codec patents**: H.264 / H.265 / AAC carry patent royalties (MPEG-LA / Via LA).
      - If your scenario uses **only JPEG / MJPEG / VP8 / VP9 / AV1** (patent-free) → fine.
      - If you need to play H.264/H.265 video → assess licensing cost, or switch to a
        patent-free codec.
      > Example: the `acceptance/iseekdashboard.md` scenario is pure MJPEG, **completely
      > avoiding this issue.**
- [ ] **Google cloud services**: Chromium's built-in sync, geolocation, voice,
      Safe Browsing, etc. need a Google API key, and **Google does not license these
      for third-party browser commercial use.**
      - Vertical scenarios usually don't need these → just disable them.
      - If needed → you must build your own backend replacement (Brave/Edge do this).
- [ ] **GPL components**: confirm the final artifact does not link to GPL components;
      if it does, exclude them at build time.

## Decision for the agent

1. Shell approach (Electron/CEF) + pure frontend content + no H.264 playback?
   → usually only needs "include license notices + rename/replace logo," low risk,
   proceed with confidence.
2. Need to play H.264/H.265, or use Google account sync etc.?
   → stop and explain the patent/licensing cost to the user; let them decide.
3. Any plan to "sell closed-source commercially"? → remind the user this is not legal
   advice and to consult a lawyer for complex cases.

> This document is general guidance, not legal advice. For commercial use with any
> doubt, consult a qualified lawyer.
