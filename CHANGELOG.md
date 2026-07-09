# Changelog

All notable changes to the `llm-pair` skill are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.1.1] - 2026-07-09

### Changed

- **Dispatch engine is now backend-agnostic and zellij-free.** Peers run as direct
  read-only CLI calls with **stdout/stderr redirected to files**; the coordinator
  reads the result from disk (never the call's return value), so a long run that
  outlasts a tool timeout is not lost. Removed the zellij-pane orchestration
  (session detection, pane stacking, `--close-on-exit`, live-tail) — it added
  operational complexity and could silently stall. No change to redaction, structured
  output, availability detection, consensus, or the fallback.

## [1.1.0] - 2026-07-09

### Changed

- **Codex tier ladder updated for the codex-cli 0.144 model lineup** (verified via
  three-way peer consensus):
  - `normal`: `gpt-5.5` → `gpt-5.6-terra` (the purpose-built "balanced everyday" model).
  - `big` / `planning`: `gpt-5.5` → `gpt-5.6-sol` (the new frontier agentic top model),
    effort raised `xhigh` → `max`.
  - `trivial` (`gpt-5.3-codex-spark`) and `small` (`gpt-5.4-mini`) unchanged.
- Repository moved: `iotashan/llm-pair` → `iotashan-llc/llm-pair` (install command +
  in-doc references).

### Added

- **Effort-ceiling guardrail**: the ladder is capped at `max`, never `ultra` — `ultra`
  auto-delegates to tool-using sub-agents, which would break the read-only advisory
  guarantee.
- **Coverage note**: `gpt-5.5`, `gpt-5.4`, and `gpt-5.6-luna` are intentionally left
  unmapped — the ladder optimizes review quality by risk tier, not minimum cost or
  full-lineup coverage. Benchmark unmapped models as in-tier *replacements*, never as
  new verdict levels.
- `CLAUDE.md` — repo guidance for future Claude Code sessions (repo map, README/SKILL
  sync rule, prompt-vendoring rationale, model-drift expectations).

### Notes

- Considered adding new classifier verdict levels to exploit the richer lineup;
  rejected by unanimous three-way consensus. The verdict enum sizes risk/scope, not
  model coverage — adjacent rungs raise cheap-classifier misclassification for
  marginal gain. The `normal → big` double-jump (model strength + Gemini fan-out at
  once) is a known smell but is already guarded by the "low confidence biases the
  verdict up one rung" rule; no change until there is measured evidence.

## [1.0.0] - 2026-07-09

Baseline: the accumulated pre-versioning history of the skill.

- Initial release of the `llm-pair` skill — calibrated peer-LLM pairing with a
  cost-sizing classifier and a read-only Codex advisor.
- Added Gemini (`agy`, Antigravity CLI) as a second peer with complex-only fan-out
  and coordinator-reconciled three-way consensus.
- Sensitive-data redaction protocol (stable placeholders before any third-party peer).
- Hardened the shared zellij-pane dispatch engine (staging, sentinels, availability
  detection) and the peer prompt contracts.
- Vendored the prompt-engineering patterns (skeleton, role lines, LENS, tier nudge)
  as static text so the hot path stays self-contained; dropped the opt-in runtime
  prompt-help hook.
