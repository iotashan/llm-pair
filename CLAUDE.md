# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A single **Claude Code Agent Skill** (`llm-pair`) — no application code, no build, no
tests. The product is prose: an operating contract a coordinating agent reads and
follows. Two files matter:

- **`SKILL.md`** — the deliverable. ~41KB operating contract loaded by the agent at
  runtime. Everything the skill *does* lives here as instructions + shell/dispatch
  recipes, not executable source.
- **`README.md`** — user-facing docs (why/how/install/setup).

There is nothing to compile, lint, or run. "Testing" a change means reasoning through
the contract and, ideally, dogfooding it in a live Claude Code session.

## The skill in one paragraph

The coordinator agent (Claude) pairs with up to two **read-only** peer LLMs — Codex
(`codex exec`) and Gemini (`agy`) — sizing each peer's `(model, effort)` to the task
via a cheap classifier, fanning out to BOTH only for complex work (planning + `big`)
to reconcile a three-way consensus. Peers run as read-only CLI calls with file-redirected output; PII/
secrets are redacted to stable placeholders before any third-party peer sees them.

## Where to edit — SKILL.md map

Section headers are stable anchors. The parts you'll touch most:

- **CONFIG block** (the primary tuning surface): the **Codex ladder**, **Gemini
  ladder**, and **fan-out matrix** (who's invoked per verdict). Changing *when the
  second peer joins* = edit the fan-out matrix, nothing else.
- **Non-negotiable safety rules** — read-only enforcement, cwd-pinning, redaction,
  no confabulation. Don't weaken these casually.
- **The shared dispatch engine** — the file-redirected run + read-back recipe all
  adapters share (write prompt file → job script → new pane → poll sentinel).
- **Peer adapters** — Codex / Gemini / Opus-fallback, each a command template +
  output mode + availability check over the shared engine.
- **The contexts** (A planning, B review, C blocker) and **consensus loop**.

## Editing rules specific to this repo

- **README and SKILL.md must stay in sync.** The tier table, trigger list, and
  fallback rules appear in both — change one, change the other.
- **Prompt templates are vendored, deliberately.** Role lines, task contracts, the
  simplicity LENS, tier nudge are baked as static text in `SKILL.md` (sourced from
  `prompt-engineering-patterns`). This keeps the hot path self-contained — it loads
  no other skill and runs no external code. Don't replace them with a runtime skill
  load.
- **Model IDs are account-specific and go stale.** The ladders name concrete model
  IDs (`gpt-5.6-sol`, `Gemini 3.1 Pro (High)`, etc.). Verify against `codex` / `agy
  models` before trusting them; treat drift as expected.
- **The setup block in README is copy-paste'd into users' CLAUDE.md.** Keep it a
  self-contained markdown block that reads correctly out of context.

## Constraints inherited from global rules

- **No AI attribution** in commits/PRs/artifacts.
- The Opus fallback is used **only when BOTH** Codex and Gemini are down — one peer
  down = proceed with the other, no Opus.
