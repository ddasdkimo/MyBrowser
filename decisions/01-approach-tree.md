# Approach Decision Tree

> Carrying the requirements converged in *Step 1*, walk this tree to a definite approach.
> Read `00-platform-matrix.md` for the platform hard rules first, then use this doc
> for the final choice.

---

## Decision tree

```
START
 │
 ├─ Q1. Need to "modify engine-internal behavior"?
 │     (intercept/rewrite traffic at the packet layer, anti-fingerprinting,
 │      change the render pipeline, change JS engine behavior)
 │
 │   ├─ Yes, and the change is deep ─────────► [Fork Chromium (self-build)]
 │   │                                          Only when truly necessary. Confirm the
 │   │                                          team can bear tens of GB of source +
 │   │                                          multi-hour builds + ongoing upstream rebase.
 │   │
 │   └─ No / only "request-layer" interception (edit headers, block domains, inject scripts)
 │         │  ── a shell can do this; continue down ──
 │         ▼
 │   Q2. Is mobile (Android/iOS) a required target?
 │     │
 │     ├─ Yes, and must share one codebase with desktop ─► [Tauri / system WebView]
 │     │                                          Accept cross-platform rendering
 │     │                                          inconsistency (see matrix warning).
 │     │                                          iOS automatically lands on WKWebView.
 │     │
 │     └─ No / mobile can be done separately ──► go to Q3 (desktop choice)
 │           ▼
 │   Q3. Desktop: what tech writes the UI?
 │     │
 │     ├─ Pure frontend (HTML/CSS/JS) for fast UI,
 │     │   want the richest ecosystem/examples ──► [Electron]
 │     │                                          Most mature, best-documented, fewest
 │     │                                          pitfalls. Default first choice for the
 │     │                                          POC and most vertical scenarios.
 │     │
 │     └─ Native language (C++/C#/other) leading,
 │         Chromium just as an embedded component ─► [CEF]
 │                                          Closer to native, smaller footprint possible,
 │                                          but more integration work than Electron.
```

## Approach quick reference

| Approach | When to pick | Biggest pro | Biggest cost |
|------|---------|---------|---------|
| **Electron** | Desktop + frontend UI + must be stable | Mature ecosystem, fast dev, good Chromium compatibility | Large size (bundles Chromium), memory-heavy |
| **CEF** | Desktop + native-language-led | Lighter, closer to native, deep customization | Complex integration, fewer docs |
| **Tauri** | Desktop+mobile shared code, extreme lightness | Installer as small as a few MB | Cross-platform rendering inconsistency |
| **Native WebView shell** | Only runs your own controlled content | Most resource-frugal, updates with OS | Built separately per platform, engine capability limited |
| **Fork Chromium** | Must modify engine internals | Full control | Extreme maintenance: build/patent/upstream burden |

## Default recommendation for vertical scenarios

For "make one specific system run perfectly + also browse general web" type
**vertical scenarios**:

> **Default: desktop uses Electron (or CEF). Do not fork. Do not use system WebView.**
>
> Rationale: a vertical scenario wants "the specific site runs perfectly" + "general
> web doesn't break," which needs a **stable, high-compatibility Chromium core.**
> Electron bundles a full Chromium directly, giving the most predictable compatibility;
> system WebView's cross-platform differences would explode debugging cost.

## After choosing

Report "approach + one-line rationale" to the user for confirmation, then:
- If `recipes/` has a matching recipe → follow it.
- Otherwise → assemble component by component from `components/`.
- Throughout, remember to read each component's "Scope / Boundaries."
