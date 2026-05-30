# Platform × Engine Feasibility Matrix

> This is the **hard-rule layer**. The agent must read it before choosing an approach;
> violating these limits makes the whole project unbuildable.

---

## Core hard rule: no single engine covers all platforms

If the user wants "all platforms," it **must be a "one engine for desktop + one engine
for mobile" combination** — you cannot use one Chromium across all five platforms.
The reason is iOS.

### The iOS hard rule (most often misjudged)

> **Apple App Store rule 2.5.6 forces all iOS browsers to use the system's built-in
> WebKit (WKWebView) — bundling your own engine is not allowed.**

- Even **Google Chrome on iOS is WebKit underneath, not Chromium/Blink**.
- So you **cannot** run a modified Chromium on iOS. If an agent tries, it's wasted effort.
- Exception: the EU DMA (since 2024) allows third-party engines, but **only in the EU,
  requires extra Apple authorization, and has very few real deployments.** Always plan
  as "iOS = WebKit."

## Feasibility matrix

| Approach | macOS | Windows | Linux | Android | iOS | One-line positioning |
|------|:----:|:----:|:----:|:----:|:----:|------|
| **Fork Chromium (self-build)** | ✅ | ✅ | ✅ | ✅ | ❌ | Max control, extreme maintenance cost |
| **CEF** (Chromium Embedded Framework) | ✅ | ✅ | ✅ | ❌ | ❌ | Embeddable Chromium, deep customization |
| **Electron** | ✅ | ✅ | ✅ | ❌ | ❌ | Desktop, JS/frontend-led, most mature ecosystem |
| **Tauri / Wails** (system WebView) | ✅ | ✅ | ✅ | ✅ | ✅ | Lightest, but inconsistent cross-platform rendering |
| **Native WebView shell** | ✅ WKWebView | ✅ WebView2 | ✅ WebKitGTK | ✅ Android WebView | ✅ WKWebView | Each platform uses its own system component |

Legend: ✅ feasible　❌ not supported

## Per-platform underlying engine (relevant to shell approaches)

| Platform | System WebView | Underlying engine | Note |
|------|------|------|------|
| macOS | WKWebView | WebKit | Updates with the OS |
| Windows | WebView2 | Chromium (Edge) | Needs Runtime; most Win11 ship it |
| Linux | WebKitGTK | WebKit | Large distro variance, must test |
| Android | Android WebView | Chromium | Updates via OS/Play |
| iOS | WKWebView | WebKit | **Forced, no alternative** |

## Cross-platform rendering consistency warning

When using a "system WebView" approach (Tauri / native shell), the **three desktop
platforms run different engines** (Mac=WebKit, Win=Chromium, Linux=WebKit), causing
rendering inconsistency, CSS behavior differences, and JS API support differences.

> **Decision rule**: if the target site/app needs "pixel-level cross-platform
> consistency" or uses newer Web APIs, the desktop side should choose a **Chromium-core
> approach (Electron / CEF)** rather than system WebView. Conversely, if it only runs
> your own controlled content, the lightweight advantage of system WebView is worth more.

## Decision flow for the agent

1. User target includes iOS? → plan mobile as WKWebView, **do not touch Chromium**.
2. User wants desktop + mobile from one codebase? → only Tauri/system WebView works,
   and accept the rendering-inconsistency risk.
3. User wants desktop only and values compatibility? → Electron or CEF (Chromium core).
4. Then proceed to `01-approach-tree.md` for final convergence.
