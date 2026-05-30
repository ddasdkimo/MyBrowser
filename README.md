# MyBrowser — Let Anyone Own Their Own Browser

> 🌐 **English** | [繁體中文](#繁體中文)

> This repo contains **no browser source code** — only a set of *architecture
> specification documents*.
> The actual browser is **built on the spot by an AI agent that reads these docs
> and tailors a browser to your needs.**

---

## What is this

MyBrowser is a **"spec-as-product, agent-as-compiler"** project.

The traditional way is to write a browser, compile it, and ship an installer.
But everyone needs a different browser — some want extreme privacy, some want deep
AI integration, some just want one internal system to run flawlessly.
Instead of one bloated browser that tries to please everyone, we do this:

1. **Capture the knowledge of "how to build a browser" as structured,
   machine-executable documents.**
2. You clone this repo and hand it to an agent (e.g. Claude Code).
3. The agent **asks you a few questions first** (platform? purpose? modify the engine?),
   then picks the right path and components from the docs and
   **assembles a browser that is uniquely yours.**

In short: **the docs are the seed, the agent is the soil, the browser that grows is yours.**

## Why this is feasible

The foundation of modern browsers (Chromium) is open source (BSD license);
Edge, Brave, Opera, and Arc are all built on it. The hard part was never
"is it allowed" — it's that **a browser is too large for anyone to assemble from memory.**

That is exactly what agents are good at: if the decision logic and assembly steps are
written clearly, an agent can reproduce them. The core of this project is to
**distill that knowledge into docs a cold-start agent can build from.**

## How to use

```bash
git clone https://github.com/ddasdkimo/MyBrowser.git
cd MyBrowser
# Then hand the folder to your agent and say:
#   "Read AGENT.md and build me a browser by following its workflow."
```

The agent entry point is [`AGENT.md`](./AGENT.md) — it drives the whole flow.

## Repo structure

```
README.md            ← what you're reading (for humans)
AGENT.md             ← the agent's entry point and workflow (for agents) ★ core
decisions/           ← decision layer: converges requirements to the right approach
  00-platform-matrix.md   platform × engine feasibility (incl. iOS=WebKit hard rule)
  01-approach-tree.md     decision tree → Fork / CEF / Electron / WebView
  02-licensing.md         license & patent obligations checklist
  03-iseekdashboard-recon.md  technical recon of the first acceptance target
acceptance/          ← acceptance layer: turns "runs perfectly" into a checkable list
components/          ← implementation layer: one self-contained recipe per component
  shell-ui/               Electron desktop shell (window + webview + persistent login)
  tabs/                   multi-tab strip on top of the shell
  stream-recorder/        record canvas/<video> streams to MP4 (multi + background)
  password-manager/       save/fill logins, encrypted via OS keychain
recipes/             ← recipe layer: full assembly paths for common vertical scenarios
```

## How document quality is guaranteed — the cold-start reproduction test

These docs are **iteratively refined**, and correctness is validated by an
executable test:

> **Can a "cold agent" — one that has read only these docs, with zero conversation
> context — build a browser that passes the checklist in [`acceptance/`](./acceptance/)?**
>
> - Yes → that part of the docs is "complete."
> - No → the knowledge is still stuck in some past conversation and never made it
>   into the docs. Add the missing docs and re-test.

This is isomorphic to software TDD: **document completeness is itself a test that
passes or fails.**

## Current status

- [x] Methodology and repo structure established
- [x] Technical recon of the first acceptance target
      ([iSeek monitoring dashboard](./acceptance/iseekdashboard.md))
- [x] **v1 architect build (HOT) works** — a desktop Electron shell + stream recorder,
      built by the architect (with full context) runs the iSeek dashboard and records MP4
      on macOS (`electron@33.4.11`). This proves the *design* works, but is **not** the
      cold-start validation — it was not built by a context-free agent.
- [ ] **★ Cold-start validation (the real test)** — have a context-free agent build from
      these docs alone and pass the acceptance checklist. THIS is what validates the docs.
- [ ] Expand more components and recipes

## License

The documents themselves are MIT licensed. The license of a browser *assembled*
from these docs depends on the components chosen — see
[`decisions/02-licensing.md`](./decisions/02-licensing.md).

---
---

## 繁體中文

> [English](#mybrowser--let-anyone-own-their-own-browser) | 🌐 **繁體中文**

> 這個 repo 裡**沒有瀏覽器的程式碼**,只有一套「架構規格文件」。
> 真正的瀏覽器,是由**智能體(AI agent)讀完這份文件後,依照你的需求現場打造出來的**。

### 這是什麼

MyBrowser 是一個**「規格即產品、智能體即編譯器」**的專案。

傳統做法是把瀏覽器寫好、編譯好、發佈成安裝檔給你。但每個人需要的瀏覽器不一樣 ——
有人要極致隱私、有人要深度整合 AI、有人只要讓某個內部系統跑得完美。
與其做一個討好所有人的肥大瀏覽器,我們改成:

1. **把「怎麼蓋一個瀏覽器」的知識,寫成結構化、可被機器執行的文件。**
2. 你 clone 這個 repo,把它交給一個智能體(如 Claude Code)。
3. 智能體**先問你幾個問題**(平台?用途?要不要改引擎?),
   再依答案從文件裡挑出對的路徑與元件,**現場組裝出專屬於你的瀏覽器**。

換句話說:**文件是種子,智能體是土壤,長出來的瀏覽器是你的。**

### 為什麼這樣做可行

現代瀏覽器底層(Chromium)是開源的(BSD 授權),市面上 Edge、Brave、Opera、Arc 都是
基於它打造的。困難從來不在「能不能用」,而在於 ——
**瀏覽器太大,沒有人能憑空記住所有組裝細節。**

而這正是智能體擅長的事:只要把決策邏輯與組裝步驟寫清楚,智能體就能照著重現。
本專案的核心,就是把這些知識**沉澱成「冷啟動智能體也能照著做出來」的文件**。

### 怎麼使用

```bash
git clone https://github.com/ddasdkimo/MyBrowser.git
cd MyBrowser
# 然後把這個資料夾交給你的智能體,並對它說:
#   「請閱讀 AGENT.md,照著流程幫我打造一個瀏覽器。」
```

智能體的進入點是 [`AGENT.md`](./AGENT.md) —— 它會引導整個流程。
**注意:除 README 外,所有文件皆為英文撰寫**,以利各種智能體一致解析。

### 文件的品質如何保證 —— 冷啟動重現測試

這套文件採**滾動式精煉**,正確性靠一個可執行的測試來驗證:

> **一個只讀過這份文件、沒有任何對話脈絡的「冷智能體」,
> 能不能照文件做出一個能通過 [`acceptance/`](./acceptance/) 驗收清單的瀏覽器?**
>
> - 能 → 這部分文件「完整」。
> - 不能 → 代表知識還卡在某次對話裡、沒落到文件上,補文件後重測。

這跟軟體的 TDD 同構:**文件的完整性,本身就是一個會通過或失敗的測試。**

### 授權

文件本身採 MIT 授權。由本文件「組裝」出的瀏覽器,其授權取決於所選用的元件 ——
詳見 [`decisions/02-licensing.md`](./decisions/02-licensing.md)。
