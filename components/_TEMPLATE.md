# Component: <NAME>

> Copy this template for every new component doc. Each component must be
> **self-contained** — readable on its own without chasing other docs. Connect to other
> components through explicit "interface contracts," not "go read that other file."

---

## Purpose

What problem this component solves, in 1–3 sentences. Why a browser needs it.

## Applies to which approaches

Which of Fork / CEF / Electron / Tauri / WebView this component is for. If it differs
per approach, say so here.

## How (implementation)

The concrete steps to build it. Be specific enough that a cold agent can reproduce it:
- Key APIs / config / files involved
- Minimal working example or pseudo-structure
- Order of operations / gotchas

## Interface contract

What this component exposes to others, and what it expects from others. Inputs/outputs,
events, config keys. This is how components connect without cross-referencing prose.

## Scope / Boundaries ★ (required — read before implementing)

- ✅ What this approach **can** do.
- ❌ What it **cannot** do.
- ⚠️ Under what conditions you must switch approach / escalate to the user.

> This section is the most valuable part of the doc. It stops the agent from leading the
> user into a dead end.

## Dependencies

Third-party packages / runtimes needed. For each: **why needed, version, source link,
scope/caveats.** (When a cold agent discovers a missing dependency during acceptance,
it writes it back here — see `AGENT.md` Step 5.)

## Changelog

- `YYYY-MM-DD` — what changed and why (so we can trace which addition made the cold
  agent pass).
