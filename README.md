# llm-pair

A Claude Code Agent Skill for **calibrated pairing with peer LLMs**. It pairs your
main coding agent (the *coordinator*) with up to two read-only advisor models —
[Codex](https://github.com/openai/codex) / GPT-5.x via the standalone `codex` CLI,
and Google **Gemini** via the `agy` (Antigravity) CLI — and spends compute in
proportion to the task: cheap, fast models for small changes; maximum reasoning for
plans and risky work. For complex work (planning and high-risk reviews) it fans out
to **both** peers and the coordinator reconciles a **three-way consensus**.

## Why

Pairing a coding agent with a strong second model raises quality (independent
drafting, adversarial review, consensus). But running peers at "best model + max
reasoning" on *every* change drains usage windows fast. `llm-pair` keeps quality
where it matters and throttles it where it doesn't — including a `skip` verdict that
pairs *nothing* on trivial work, and a "complex-only" fan-out so the second
(rate-limited) peer is reserved for planning and big reviews.

It also **hardens** the peer calls and **protects your data**:

- Advisors run **read-only** (Codex sandboxed; Gemini inlined + headless, no tools,
  no `--add-dir`) — no surprise file edits.
- Pinned **working directory**, structured **JSON output** (Codex via
  `--output-schema`; Gemini via defensive parsing), and **never confabulated** — if
  a peer returns nothing usable, the skill says so instead of inventing an opinion.
- **Sensitive-data redaction.** Codex (OpenAI) and Gemini (Google) are third
  parties, so PII/secrets are replaced with stable placeholders **before** their
  prompts are built; the coordinator keeps the map and re-expands on return. The
  Opus fallback (Anthropic) is exempt.

## How it works

A cheap classifier sizes each task to a **(model, effort) tier**, per backend:

| verdict   | Codex                 | effort | Gemini (`agy`)              |
|-----------|-----------------------|--------|-----------------------------|
| `skip`    | — (no pairing)        | —      | —                           |
| `trivial` | `gpt-5.3-codex-spark` | low    | `Gemini 3.5 Flash (Low)`    |
| `small`   | `gpt-5.4-mini`        | low    | `Gemini 3.5 Flash (Medium)` |
| `normal`  | `gpt-5.6-terra`       | medium | `Gemini 3.1 Pro (Low)`      |
| `big`     | `gpt-5.6-sol`         | max    | `Gemini 3.1 Pro (High)`     |

…with fixed rules: **plans always run at max**, **blockers floor at `small`**, and
any risk signal (auth, migrations, money, shared libs, infra, public API, security,
concurrency) blocks `skip`/`trivial` and biases upward.

**Fan-out (default: complex-only).** Codex handles trivial/small/normal alone;
`big` reviews/blockers and **all planning** fan out to **Codex + Gemini** for a
three-way consensus. (Gemini's lower tiers exist only as a substitute when Codex is
down.) The matrix is one editable block in `SKILL.md`.

Four contexts, two collaboration patterns:

- **Plans / implementation plans** → *parallel-draft-then-converge*: the coordinator
  and both peers each draft independently (no anchoring), then reconcile to
  consensus.
- **Implementation review** & **blocker diagnosis** → *draft → advisory review →
  integrate*: the coordinator produces the work, peers review/diagnose read-only,
  the coordinator integrates.

Pairing triggers at the **work-item boundary** (e.g. each item in a ticket), not per
edit; blocker pairing is reactive (an error that survives 2+ fixes or cascades). It
can fire **proactively** during defined task work — see Setup.

**Consensus** is bounded: the coordinator is the *executive synthesizer*, runs at
most 2 peer rounds (targeted disagreement briefs, not full transcript replay), then
decides unilaterally and records unresolved trade-offs.

**Fallback:** previous-generation Opus is used as the peer **only when BOTH Codex
and Gemini are unavailable**. If one peer is down, the skill proceeds with the other
peer + coordinator (no Opus). On a single-peer tier, a downed Codex is substituted
by Gemini at the equivalent rung.

See [`SKILL.md`](./SKILL.md) for the full operating contract.

## Install

Via [`npx skills`](https://github.com/vercel-labs/skills):

```bash
npx skills add iotashan-llc/llm-pair
```

Or manually — drop `llm-pair/` into your shared `~/.agents/skills/` store and
symlink it into each agent's skills dir (e.g. `~/.claude/skills/llm-pair →
../../.agents/skills/llm-pair`).

**Prerequisites:**

- The standalone [`codex` CLI](https://github.com/openai/codex) on your PATH
  (`brew install codex`) and authenticated. No Claude Code Codex *plugin* is
  required — `llm-pair` shells out to `codex exec` directly.
- *(Optional, for the second peer)* the `agy` (Antigravity) CLI on your PATH and
  authenticated (`agy models` should list models). Without it, the skill runs
  Codex-only.
- An Opus-capable agent for the both-peers-down fallback.

## Setup — wire it into your CLAUDE.md (required)

`npx skills` does **not** display post-install messages, so this step is manual (the
skill will also offer to do it for you the first time it runs). Add this block to
your global `CLAUDE.md` (or a project one) so the agent reaches for the skill
automatically:

```markdown
## Pairing with peer LLMs — the `llm-pair` skill
- **Use the `llm-pair` skill for all peer-LLM pairing.** Triggers: the user says
  "pair with codex" / "pair with gemini" / "pair with the LLM" / "llm-pair" /
  "three-way" / "co-develop with a second model" — OR you are doing substantial
  **defined task work** (a ticket, a feature, "do XYZ-123") and reach a pairing
  boundary.
- **Proactive triggering (don't wait to be asked):** during defined task/ticket
  work, invoke `llm-pair` at these boundaries —
  - **Planning** — before forming an overall or implementation plan (parallel-draft
    with the peers; don't anchor on your draft).
  - **Work-item review** — when a work item / sub-task reaches a complete state.
  - **Blocker** — when an error survives 2+ fix attempts or cascades.
- **Granularity:** pair at work-item boundaries, NOT per edit. The classifier +
  `skip` verdict keep small/trivial work cheap or unpaired, so erring toward
  invoking it is safe.
- **What the skill owns (don't re-derive):** sizing each peer's model + effort;
  running peers read-only (Codex via `codex exec`, Gemini via `agy`) with structured
  output; the complex-only fan-out to BOTH peers for three-way consensus; redacting
  sensitive data to placeholders before any third-party peer; cwd-pinning; output
  verification (no confabulation); the Opus fallback used ONLY when both external
  peers are down; and bounded consensus with a "what each peer caught / what we
  rejected" note.
- Don't accept a peer blindly; don't dismiss it reflexively. The reconciled position
  is the deliverable.
```

If you want **explicit-only** pairing (no proactive firing), drop the "Proactive
triggering" bullet and keep only the first trigger line.

## Configure

Edit the **CONFIG** block at the top of `SKILL.md`: the **Codex ladder**, the
**Gemini ladder**, and the **fan-out matrix** (who's invoked per verdict). Verify
model IDs against your accounts (`codex` / `agy models`). To change when the second
peer joins (e.g. "normal and up", or "always both"), edit the fan-out matrix —
nothing else changes.

## Dependencies & bindings

`llm-pair` is backend-agnostic by design, with these bindings:

- **Classifier** — Claude Code's REPL `haiku()` (any fast model with structured
  output works; keep the schema, swap the call).
- **Peer backends** — the `codex` CLI (`codex exec`) and the `agy` CLI, each a
  concrete *adapter* (command template, output mode, read-only enforcement,
  availability detection) over a shared zellij-pane dispatch engine.
- **Fallback** — any Opus-capable agent, used only when both external peers are
  unavailable.
- **Prompt templates** — the peer-prompt skeleton, role lines, task output contracts,
  the simplicity LENS, and the tier nudge are **vendored** from the
  `prompt-engineering-patterns` skill (instruction-hierarchy, task-schema, grounding,
  progressive-disclosure patterns), baked into `SKILL.md` as static text — the hot
  path loads no prompt skill and runs no external code, so the skill stays
  self-contained.

To add another peer backend, write a new adapter + its CONFIG ladder; the contexts,
classifier, redaction, consensus, and fallback are unchanged.

## License

MIT (intended).
