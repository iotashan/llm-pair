# llm-pair

A Claude Code Agent Skill for **calibrated pairing with a peer LLM**. It pairs
your main coding agent with a second model (currently [Codex](https://github.com/openai/codex)
/ GPT‑5.x, via the standalone `codex` CLI) as a **read‑only advisor**, and spends
compute in proportion to the task: cheap, fast models for small changes; maximum
reasoning for plans and risky work — instead of burning the top model on every
one‑line fix.

## Why

Pairing a coding agent with a strong second model raises quality (independent
drafting, adversarial review, consensus). But running the peer at "best model +
max reasoning" on *every* change drains usage windows fast. `llm-pair` keeps the
quality where it matters and throttles it where it doesn't — including a `skip`
verdict that pairs *nothing* on trivial work.

It also hardens the peer call: the advisor runs **read‑only** (sandboxed shell — no surprise file edits),
in a **pinned working directory** (no wrong‑worktree reads), returns **structured
JSON** (`--output-schema`), and is **never confabulated** — if the peer returns
nothing usable, the skill reports that instead of inventing an opinion.

## How it works

A cheap classifier sizes each task to a **(model, reasoning‑effort) tier**:

| verdict   | model                 | effort   |
|-----------|-----------------------|----------|
| `skip`    | — (no pairing)        | —        |
| `trivial` | `gpt-5.3-codex-spark` | low      |
| `small`   | `gpt-5.4-mini`        | low      |
| `normal`  | `gpt-5.5`             | medium   |
| `big`     | `gpt-5.5`             | xhigh    |

…with two fixed rules: **plans always run at max**, and **blockers floor at
`small`**. Any risk signal (auth, migrations, money, shared libs, infra, public
API, security, concurrency) blocks `skip`/`trivial` and biases upward.

Four contexts, two collaboration patterns:

- **Plans / implementation plans** → *parallel‑draft‑then‑converge*: the agent and
  the peer each draft independently (no anchoring), then reconcile to consensus.
- **Implementation review** & **blocker diagnosis** → *draft → advisory review →
  integrate*: the agent produces the work, the peer reviews/diagnoses read‑only,
  the agent integrates.

Pairing triggers at the **work‑item boundary** (e.g. each item in a ticket), not
per edit; blocker pairing is reactive (an error that survives 2+ fixes or
cascades). It can fire **proactively** during defined task work — see Setup.

See [`SKILL.md`](./SKILL.md) for the full operating contract.

## Install

Via [`npx skills`](https://github.com/vercel-labs/skills):

```bash
npx skills add iotashan/llm-pair
```

Or manually — drop `llm-pair/` into your shared `~/.agents/skills/` store and
symlink it into each agent's skills dir (e.g. `~/.claude/skills/llm-pair →
../../.agents/skills/llm-pair`).

**Prerequisites:** the standalone [`codex` CLI](https://github.com/openai/codex)
on your PATH (`brew install codex`) and authenticated. No Claude Code Codex
*plugin* is required — `llm-pair` shells out to `codex exec` directly.

## Setup — wire it into your CLAUDE.md (required)

`npx skills` does **not** display post‑install messages, so this step is manual
(the skill will also offer to do it for you the first time it runs). Add this
block to your global `CLAUDE.md` (or a project one) so the agent reaches for the
skill automatically:

```markdown
## Pairing with a peer LLM — the `llm-pair` skill
- **Use the `llm-pair` skill for all peer‑LLM pairing.** Triggers: the user says
  "pair with codex" / "pair up with codex" / "pair with the LLM" / "llm-pair" /
  "co‑develop with a second model" — OR you are doing substantial **defined task
  work** (a ticket, a feature, "do XYZ‑123") and reach a pairing boundary.
- **Proactive triggering (don't wait to be asked):** during defined task/ticket
  work, invoke `llm-pair` at these boundaries —
  - **Planning** — before forming an overall or implementation plan (parallel‑draft
    with the peer; don't anchor it on your draft).
  - **Work‑item review** — when a work item / sub‑task reaches a complete state
    (review the unit, not each edit).
  - **Blocker** — when an error survives 2+ fix attempts or cascades into new errors.
- **Granularity:** pair at work‑item boundaries, NOT per edit. The skill's
  classifier + `skip` verdict keep small/trivial work cheap or unpaired, so erring
  toward invoking it is safe.
- **What the skill owns (don't re‑derive):** sizing the peer's model + reasoning
  effort, running it read‑only via `codex exec` with structured JSON output,
  cwd‑pinning, output verification (no confabulation), the Opus fallback when
  Codex is unavailable, and iterating to consensus with a "what the peer caught /
  what we rejected" note.
- Don't accept the peer blindly; don't dismiss it reflexively. The reconciled
  position is the deliverable.
```

If you want **explicit‑only** pairing (no proactive firing), drop the "Proactive
triggering" bullet and keep only the first trigger line.

## Configure

Edit the **CONFIG** tier ladder at the top of `SKILL.md`. Verify the model IDs
against your account; the defaults target the Codex GPT‑5.x family. The effort
floor is `low` (`none`/`minimal` are rejected when Codex's `web_search` tool is
enabled). If a model ID is rejected, the skill falls back toward the stronger model.

## Dependencies & bindings

`llm-pair` is backend‑agnostic in design, with these bindings:

- **Classifier** — Claude Code's REPL `haiku()` (any fast model with structured
  output works; keep the schema, swap the call).
- **Peer backend** — the standalone `codex` CLI via `codex exec`
  (`-s read-only`, `-C`, `-m`, `-c model_reasoning_effort=`, `--output-schema`).
- **Fallback** — any Opus‑capable agent, used when Codex is rate‑limited/unavailable.

To add another peer backend, replace the invocation contract and model IDs; the
contexts, classifier, fallback, and consensus loop are unchanged.

## License

MIT (intended).
