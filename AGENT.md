# AGENT.md — Agent Entry Point & Workflow

> Hello, agent. If you're reading this, someone wants to use this repo to build a
> tailored browser.
> **This document is your SOP. Follow it strictly — do not skip steps, and do not
> start writing code from prior assumptions.**

---

## Your role

You are not here to "freely build a browser." You are a **compiler**:
the input is "the user's requirements" + "this repo's docs", and the output is
"a browser that passes the acceptance checklist."

Your goal is **not** to build the most impressive browser, but to build one that
**just satisfies the user's needs and can be objectively verified as runnable.**
The more you narrow scope, the better.

## ⚠️ Hard rules (violating these causes outright failure — read all before acting)

1. **Converge first, implement second.** Before writing any code, you MUST complete
   the questions in *Step 1* below and get explicit answers from the user.
2. **iOS cannot bundle its own engine.** If the target platforms include iOS, the
   mobile side is forced to use the system WebKit (WKWebView). **Do not try to put
   Chromium on iOS** — it's wasted effort and cannot ship. See
   `decisions/00-platform-matrix.md`.
3. **For vertical scenarios, do not fork Chromium by default.** The only valid reason
   to fork is "must modify engine-internal behavior." Otherwise always use a shell
   approach (Electron / CEF / WebView). Fork maintenance cost will crush a small team.
4. **Do not carry the whole browser at once.** Per *Step 3*, assemble only the
   components the chosen path needs.
5. **The acceptance checklist is the only definition of "done."** When finished, you
   must go through the checklist in `acceptance/` item by item — never declare "done"
   by gut feeling.

## Workflow

### Step 1 — Converge requirements (mandatory, never skip)

Before doing anything, ask the user the following and wait for answers.
**Do not invent answers.**

1. **Target platforms?** (desktop macOS / Windows / Linux, mobile iOS / Android —
   multiple allowed)
2. **Core purpose?** (privacy / de-Google, AI automation integration, a specific
   vertical scenario, learning/exploration)
3. **Modify engine internals?** (e.g. intercept/rewrite network traffic at packet
   level, anti-fingerprinting, change rendering behavior — if the user is unsure,
   default to "no")
4. **Any specific site or system that must run perfectly?** (If yes, find the matching
   checklist under `acceptance/`; if none exists, build one together with the user.)

### Step 2 — Choose the approach

With Step 1's answers, read `decisions/00-platform-matrix.md` and
`decisions/01-approach-tree.md`, and walk the decision tree to a definite approach
(Fork / CEF / Electron / WebView / Tauri).

**Report your chosen approach and a one-line rationale to the user for confirmation
before continuing.**

### Step 3 — Assemble

1. If `recipes/` already has a recipe for the vertical scenario → follow it; it has
   pre-selected the components.
2. Otherwise → take the component docs needed for the approach from `components/` and
   implement per each component's "How" section.
3. Every component doc has a **"Scope / Boundaries"** section — **read it first.** It
   tells you what this path cannot do and under what conditions to switch approach.
   When you hit a boundary, report to the user — do not force it.

### Step 4 — Acceptance

1. Open the matching checklist in `acceptance/` and **test and check each item.**
2. Any item fails → go to Step 5.
3. All pass → done. Report to the user with the checked results.

### Step 5 — When you hit a gap (the heart of iterative refinement)

When you find the docs insufficient to pass an acceptance item (e.g. a package needs
an extra third-party dependency, or a config file is missing):

1. **Diagnose first**: what exactly is missing? A dependency, a setup step, or an
   undocumented gotcha?
2. **Explain to the user** and obtain the missing info.
3. **Write the info back into the docs** — into the relevant component's
   `dependencies.md` or a new section, noting "why needed, version, source, scope."
   > This step is the lifeblood of the project: knowledge must be "deposited" from
   > conversation back into the docs, or the next cold agent will get stuck at the
   > same place.
4. Continue acceptance once filled in.

## Self-check: am I a "good enough" output?

When done, ask yourself:

- [ ] Did I complete the Step 1 questions before acting?
- [ ] Does my chosen approach violate any *Hard rule*?
- [ ] Was every `acceptance/` item actually *tested*, not assumed?
- [ ] Did I write back every gap I hit, so the next cold agent won't get stuck again?

## Special note for "verifier" agents

If your assigned task is to **verify the completeness of these docs** (the cold-start
reproduction test):

- You may rely **only** on this repo's contents. Do not use any "I remember from
  before…" context.
- Walk Steps 1–4 strictly.
- If any item in Step 4 fails, **that is a defect in the docs** — clearly record
  "where it got stuck, what was missing." That gap report is itself the most valuable
  output for this project.
