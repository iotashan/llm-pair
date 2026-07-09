---
name: llm-pair
version: 1.1.1
description: >-
  Use for ALL pairing with peer LLMs. Triggers: the user says "pair with codex",
  "pair with gemini", "pair with the LLM", "llm-pair", "three-way", or
  "co-develop with a second model" — OR proactively during substantial defined
  task/ticket work at the plan, work-item-review, and blocker boundaries. Runs up
  to TWO read-only peers — Codex / GPT-5.x (`codex exec`) and Google Gemini
  (`agy`, the Antigravity CLI) — calibrating each peer's model + reasoning effort
  to the task, and fans out to BOTH for complex work (planning + `big`) so the
  coordinating Claude reconciles a three-way consensus. Redacts sensitive data to
  stable placeholders before anything reaches a third-party peer. Self-throttles
  via a `skip` verdict on trivial work. Falls back to previous-gen Opus only when
  BOTH external peers are unavailable.
---

# llm-pair — calibrated pairing with peer LLMs

Pair the main agent (the **coordinator**, this Claude) with one or two **peer
LLMs** to raise quality through independent drafting, adversarial review, and
consensus — **without** spending the most expensive model + maximum reasoning on
every change, and **without** leaking sensitive data to third parties.

Two peer backends:

- **Codex** (GPT-5.x) via the `codex` CLI (`codex exec`) — the primary peer.
- **Gemini** via the `agy` CLI (Google Antigravity) — joins for **complex** work
  (planning + `big`) to give a genuinely different model's perspective.

The coordinator runs ALL the back-and-forth and is the **executive synthesizer**:
peers advise; Claude reconciles, decides, and integrates. (The name "llm-pair" is
now a mild misnomer — complex work is a three-way panel — but the name stays.)

## The problems this solves

1. **Cost calibration.** Running everything at the ceiling (top model + `max`) gives a
   one-line fix the same super-review as a risky migration; the usage window
   evaporates. This skill picks a **(model, effort) tier** sized to the task, per
   backend, and only fans out to the second peer when the stakes justify it.
2. **Reliability.** Pin peers to **read-only**, a fixed **working directory**,
   **structured output**, and **verified non-empty results** — and **never
   confabulate** a result a peer didn't actually return.
3. **Privacy.** Codex (OpenAI) and Gemini (Google) are **third parties**. Redact
   sensitive data to stable placeholders **before** building their prompts, keep
   the map coordinator-local, and re-expand on return. (The Opus fallback is
   Anthropic — same trust boundary as the coordinator — and is exempt.)
4. **Model diversity.** A second, architecturally different model catches what one
   model (and the coordinator) are blind to — the whole point of a panel.

---

## Non-negotiable safety rules

1. **Peers are read-only advisors.** They never write your files. Codex is pinned
   with `-s read-only`; Gemini has no such flag, so its safety comes from inlining
   the artifact + a headless "use no tools" instruction + omitting
   `--dangerously-skip-permissions` + **no `--add-dir`**. Give a peer write
   capability only if the user explicitly asks it to patch something this turn.
2. **Redact sensitive data before it reaches a third-party peer** (Codex, Gemini).
   See [Sensitive-data redaction](#sensitive-data-redaction-protocol). This is a
   hard gate, not best-effort.
3. **Never confabulate.** If a peer returns nothing usable, report that — do not
   invent its opinion or pass your own analysis off as the peer's.

---

## CONFIG — tier ladders + fan-out (edit this block to tune)

The classifier returns a **verdict**; these tables map verdict → (model, effort)
per backend, and verdict → which peers to invoke. **This block is the single
source of truth — exact model strings live here, never scattered in prose.**
Verify model IDs against each CLI (`codex` / `agy models`); if a model is rejected,
fall back one rung toward the stronger model and note it.

### Codex ladder (`-m <model>` + `-c model_reasoning_effort=<effort>`)

| verdict     | model                  | effort   | when                                                         |
|-------------|------------------------|----------|-------------------------------------------------------------|
| `skip`      | — (do not call peers)  | —        | trivial **and** zero risk signals — not worth pairing       |
| `trivial`   | `gpt-5.3-codex-spark`  | `low`    | rename, comment, one-line tweak, isolated copy/config       |
| `small`     | `gpt-5.4-mini`         | `low`    | small, well-contained change, low blast radius              |
| `normal`    | `gpt-5.6-terra`        | `medium` | ordinary feature/fix, moderate scope                        |
| `big`       | `gpt-5.6-sol`          | `max`    | large, cross-cutting, or high breakage risk                 |

**Codex effort floor is `low`.** `none`/`minimal` are rejected by the current Codex
tool config (incompatible with the `web_search` tool) — never emit them.
**Codex effort ceiling is `max`, NOT `ultra`.** The effort enum ascends
`low < medium < high < xhigh < max < ultra`, but `ultra` = "maximum reasoning with
**automatic task delegation**" — it auto-spawns tool-using sub-agents, which breaks
the read-only advisory guarantee. Never emit `ultra`; `max` is the top rung.
**Coverage is intentional, not exhaustive** — the ladder optimizes review quality by
risk tier, not minimum cost or full-lineup coverage. `gpt-5.5` (frontier), `gpt-5.4`
(strong everyday), and `gpt-5.6-luna` (fast/affordable) are deliberately unmapped;
add them only if you introduce a cost/latency policy dimension.

### Gemini ladder (`--model "<name> (<effort>)"` — effort is baked into the name)

| verdict     | model string                | when                                            |
|-------------|-----------------------------|-------------------------------------------------|
| `trivial`   | `Gemini 3.5 Flash (Low)`    | substitute-only (see fan-out)                   |
| `small`     | `Gemini 3.5 Flash (Medium)` | substitute-only                                 |
| `normal`    | `Gemini 3.1 Pro (Low)`      | substitute-only                                 |
| `big`       | `Gemini 3.1 Pro (High)`     | fan-out peer                                    |
| `planning`  | `Gemini 3.1 Pro (High)`     | fan-out peer (fixed)                            |

`agy models` is the authority for valid strings. Note Pro has **no Medium** tier.
Under the default fan-out, Gemini is invoked only at `big`/planning; the lower rungs
exist **only** for substitution (when Codex is down) — don't over-optimize unused
paths, and don't silently upgrade a Codex outage on a small task to Pro High.

### Fan-out matrix — who gets invoked (default: complex-only)

| verdict     | peers invoked                       |
|-------------|-------------------------------------|
| `skip`      | none                                |
| `trivial`   | codex                               |
| `small`     | codex                               |
| `normal`    | codex                               |
| `big`       | **codex + gemini** (three-way)      |
| planning    | **codex + gemini** (three-way)      |
| blocker     | codex (gemini too if classified `big`) |

To change the policy (e.g. "normal and up", or "always both"), edit this matrix —
nothing else changes.

**Fixed overrides (never run the classifier for these):**

- **Planning** (overall plan, implementation plan, pre-work) → **both peers**, codex
  `gpt-5.6-sol @ max` + gemini `Gemini 3.1 Pro (High)`. Define "planning" narrowly:
  an explicit implementation/architecture plan or task decomposition requested
  before code changes — **not** every passing "I'll do X" thought, which would
  burn Gemini quota.
- **Blocker / error diagnosis** → classifier runs, floor `small`; gemini joins only
  if it classifies `big`.

**Risk signals** (any one blocks `skip`/`trivial` and biases the verdict upward,
often to `big` — which also pulls in Gemini): auth / authz / sessions, DB
migrations or schema, money / billing / payments, shared or widely-imported
libraries, infra / CI / deploy, public API or contract changes, security-sensitive
code, concurrency.

---

## Peer selection & substitution

1. Classify → verdict. Read intended peer set from the **fan-out matrix**.
2. Check each intended peer's [availability](#availability--fallback-state-machine).
   Drop the unavailable ones.
3. **If ≥1 third-party peer is available, proceed with those — no Opus.** A
   three-way that loses one peer becomes a two-way (peer + Claude); don't block.
4. **Substitution (single-peer tiers):** if the primary (Codex) is down but Gemini
   is up, substitute Gemini at its **tier-equivalent** rung (trivial→Flash Low,
   small→Flash Medium, normal→Pro Low) — a *substitute*, not a second peer. Don't
   re-invoke both if Codex recovers mid-flow.
5. **Opus fallback only if BOTH Codex and Gemini are unavailable** — see
   [the state machine](#availability--fallback-state-machine).

---

## When to pair — granularity (read this first)

**Pairing happens at the work-item boundary, not per edit.** Over-triggering burns
the window faster than the old always-max default.

- A ticket with 3–4 work items → pair ~3–4 times: **plan once** up front (three-way),
  then **review each work item** as it reaches a coherent, complete state.
- A single work item with a dozen edits → do **not** pair on each edit. Pair once,
  on the completed unit.
- **Blocker pairing is reactive** — only when an error **survives 2+ fix attempts**
  *or* **cascades into new errors**. Not the first failing test.

### Proactive triggering

You do **not** wait for the user to say "pair." During substantial **defined task
work** (a ticket, a feature, "do XYZ-123"), invoke this skill at the boundaries
above. The `skip` verdict + work-item granularity keep small work cheap or
unpaired, so erring toward invoking is safe. Don't pair on pure questions or
trivial one-off edits.

---

## Sensitive-data redaction protocol

Codex and Gemini are **third parties**. Before their prompts are **assembled**,
replace every sensitive value with a stable placeholder; keep the map
coordinator-local; re-expand on return. The Opus fallback and the coordinator are
Anthropic — they get **raw** content.

**Redact before assembly, not after.** Run redaction over each raw artifact
(diff, error text, code, requirements, logs, filenames, stack traces) *as you build
the peer prompt* — never as a final pass over a finished prompt string, or quoted
logs / paths / tracebacks slip through. The redacted text is what gets written to
the `/tmp` prompt file; the **raw text must never touch a peer-readable file**.

**What counts as sensitive (PII is one category):** people's names, emails, phones,
postal addresses; **secrets** — API keys, tokens, passwords, cookies, bearer
headers, private keys, DB connection strings; customer/account/user IDs; internal
hostnames, private URLs, IPs; proprietary client/brand identifiers; home-directory
paths that embed a username; anything the user marks sensitive. **When unsure,
redact.** For secrets, never send even surrounding context that reveals the value —
emit the placeholder only.

**Placeholder format — collision-resistant.** Use `__PII_<TYPE>_<N>__` (e.g.
`__PII_PERSON_1__`, `__PII_EMAIL_2__`, `__PII_SECRET_3__`, `__PII_CLIENT_1__`,
`__PII_HOST_1__`). Plain single brackets (`[PII_1]`) get mangled by models
(`[ PII_1 ]`, markdown-link interpretation) so re-expansion misses them and broken
code gets integrated — the double-underscore/alnum form survives tokenization
intact. Reserve the `__PII_…__` pattern: if it already appears in the input, escape
it first so user text can't collide. **Reuse the same placeholder for repeat
occurrences** of the same value so cross-references survive, and keep the mapping
**stable across all rounds** of the same pairing session.

**Map ownership & leakage surface.** The placeholder→real map lives in the
coordinator's working memory. Do **NOT** write it under `/tmp` (or any peer-staging
path): `/tmp` is world-readable, so a `pii-map.json` there would re-leak every secret
you just redacted. If persistence is unavoidable, use a coordinator-private dir
**outside** `/tmp/llmpair-*` (`mkdir -m700`; file `chmod 600`), never named in any peer
command, prompt, or `--add-dir`, and delete it when the session ends. Redacted content
is all that may appear in: `/tmp` prompt/output
files, stderr/`.err` files, parse-error messages. After
assembling a peer prompt, **grep it for known sensitive tokens** before dispatch
(belt-and-suspenders). Re-expand placeholders **only** in Claude's integrated
output, file edits, and user-facing summaries — never echo the map back into a peer
prompt. **Secrets** (keys/tokens/passwords) stay as placeholders even in user-facing
output unless the user explicitly needs the value.

**If a task has no sensitive data** (e.g. editing a generic open-source skill), say
so ("redaction pass: nothing to redact") and send raw. Don't over-redact to where
the peer can't reason — preserve structure and types.

---

## The classifier (cheap, structured)

Run one fast-model classification. In Claude Code, use the REPL `haiku()` shorthand
with the schema below. The classifier returns the **verdict**; map verdict →
model+effort and peer set via [CONFIG](#config--tier-ladders--fan-out-edit-this-block-to-tune)
— do **not** let the classifier invent a model ID.

```js
const SCHEMA = {
  type: 'object',
  properties: {
    reasoning:  { type: 'string' },   // FIRST: brief CoT that DRIVES the verdict
    verdict:    { type: 'string', enum: ['skip','trivial','small','normal','big'] },
    confidence: { type: 'string', enum: ['high','med','low'] },
    riskFlags:  { type: 'array', items: { type: 'string' } }
  },
  required: ['reasoning','verdict','confidence','riskFlags']
}
```

**Signal pack** — a compact summary, not the whole diff: the context (review vs
blocker) + one-line task summary; `git diff --stat`; files touched and ±lines;
visible risk flags. **Classify on redacted signals if the summary contains
sensitive data** (the classifier is local Haiku, so raw is acceptable, but stay
consistent).

**Rules:** any risk flag → `skip`/`trivial` off the table; **low `confidence` biases
the verdict UP one rung** (the conservative default on a borderline `normal`/`big`);
blocker → floor `small`; **explicit user override wins** ("pair on max", `--tier big`,
"three-way", a named model) → skip the classifier. Surface the verdict + chosen peers in one line before
dispatching ("Classified review as `big` → codex gpt-5.6-sol@max + gemini Pro High").

---

## The shared dispatch engine (DRY core)

Both peers run through the **same** mechanism — only the command line differs (see
[adapters](#peer-adapters)). Each peer call: stage a **redacted** prompt file, run
the peer CLI with its **stdout/stderr redirected to files**, then read those files.
**Reading the result from the file — not the call's return value — is the core
reliability guarantee:** a long `max`/Pro-High run can outlast a tool timeout and
return empty, but its output is already on disk.

> **⚠️ Stage with the file-write tool, never shell-string interpolation.** Write
> `$J.prompt.txt` (+ `$J.schema.json` for Codex) with your **file-write tool** — raw
> content, no escaping. Never interpolate a diff / backticks / `$` into a shell string
> or a REPL template literal. PREFLIGHT before launch: `$J.prompt.txt` exists &
> non-empty; (Codex) `$J.schema.json` is valid JSON; the peer binary is on PATH. A
> missing/empty prompt is a STAGING bug, not peer-unavailable.

```bash
J="$(mktemp -u /tmp/llmpair-<peer>-XXXXXX)"   # PER PEER + unique. In a 3-way, codex &
                                              # gemini MUST get DISTINCT $J or they
                                              # clobber each other's files (correctness
                                              # + cross-peer leak).
# 1. Write $J.prompt.txt (REDACTED for third-party peers) with the file-write tool.
#    For Codex also write $J.schema.json (its --output-schema needs a file).
# 2. Run the peer, redirecting BOTH streams to files + writing a done-sentinel:
<PEER COMMAND> > "$J.out" 2> "$J.err"; echo "EXIT=$?" > "$J.done"   # <PEER COMMAND>: see adapter
# 3. Read the RESULT FROM THE FILE (never the call's return value):
cat "$J.out"; cat "$J.done"    # + cat "$J.err" to classify a failure
```

Run step 2 as a single `sh()` call. A short peer run returns inline; a long one the
Claude Code harness **backgrounds** and delivers via a task-notification when it
finishes. **Either way, read `$J.out` from disk** once `$J.done` exists — do **not**
treat an empty call-return as "the peer said nothing." Reads of a just-written `$J.*`
can transiently report "no such file" (filesystem/tool lag) — **re-read after a 1–2 s
sleep** before concluding a peer produced nothing; a single failed read is NOT "peer
unavailable."

**Reliability notes:**

- **Read the file, not the return value.** Output is on disk the moment the peer
  exits; the `sh()` return value is unreliable for long runs. This is the whole reason
  for the `> $J.out` redirect + `$J.done` sentinel.
- **Clean up by explicit path, never a glob** that can match a still-needed file (`rm
  -f /tmp/llmpair-*` mid-run deletes a staged prompt). Prefer a per-peer `mktemp -d`
  dir and `rm -rf` it when done.

**For three-way work, run BOTH peers** (codex + gemini) with DISTINCT `$J`, then read
both `$J.out`. Partial completion is fine: if one errored or is still running past a
sane deadline, proceed with whoever returned (see availability).

**Output verification — never confabulate:** if a peer's output is empty / unusable,
retry once (fresh call); if it still fails, report honestly and proceed with the
remaining peer(s) or the fallback — never fabricate an opinion.

---

## Peer adapters

Each adapter is a concrete binding of the shared engine: `<PEER COMMAND>`, output
mode, read-only enforcement, and availability detection. Selection, redaction,
consensus, and fallback are shared and unchanged across adapters.

### Codex adapter (`codex exec`)

```bash
<PEER COMMAND> =
  cat "$J.prompt.txt" | codex exec \
    --skip-git-repo-check -s read-only \
    -C "<ABSOLUTE repo/worktree path>" \
    -m "<model from Codex ladder>" -c model_reasoning_effort="<effort>" \
    --output-schema "$J.schema.json" -
```

- **`-s read-only`** — sandboxes the peer's model-generated shell to read-only,
  killing "Codex edited my files." Constrains shell/file tools, **not** a separately
  write-capable MCP server — for a hard guarantee also restrict tools/config.
- **`-C <abs path>`** — pins the working root. Never omit.
- **`-m` + `-c model_reasoning_effort=`** — the calibrated tier. (No
  `--reasoning-effort` flag; effort is a config override.)
- **`--output-schema <file>`** — forces structured JSON you can parse. Use the
  review / diagnosis / plan schema. (Drop it for free-form turns — plan
  reconciliation, consensus follow-ups — and read the clean final message.)
- **`--skip-git-repo-check`** — lets the peer run on non-git content (docs/skill
  dir, worktree). Trade-off: removes the fail-fast "not a repo" guard, so a typo'd
  `-C` won't fail fast — double-check it.
- **prompt via stdin + `-`** — avoids quoting hell for large diffs.
- Codex puts reasoning/progress on **stderr** (`$J.err`); the
  clean final JSON is on **stdout** (`$J.out`). Don't use `--json` (noisy event
  stream).
- **Availability:** unavailable if `codex` not on PATH; or `$J.done` exit ≠ 0; or
  `$J.out` empty/invalid after one retry; or `$J.err` matches
  `rate.?limit|quota|usage limit|429|401|403|not logged in|unauthorized`.

### Gemini adapter (`agy`, the Antigravity CLI)

```bash
<PEER COMMAND> =
  cat "$J.prompt.txt" | agy -p - \
    --model "<model string from Gemini ladder, e.g. Gemini 3.1 Pro (High)>" \
    --print-timeout 10m
```

- **`-p -`** — `--print` (non-interactive single shot) reading the prompt from
  **stdin** (bare `-p` errors "needs an argument"). Like Codex's `-`, this avoids
  quoting hell.
- **`--model "Name (Effort)"`** — model **and** effort in one string; effort is
  baked into the name (`agy models` lists valid ones).
- **No `--output-schema`.** Coax JSON in the prompt and **parse defensively**
  (Gemini wraps JSON in conversational filler): (1) strict `JSON.parse` of stdout →
  (2) first fenced ```json block → (3) first balanced `{...}` → (4) else preserve
  raw as `unstructured_advice`, mark `schema_valid:false`, never invent fields.
- **No `-s read-only`; safety is by DISCIPLINE, not a sandbox** (treat Gemini as
  *no-filesystem-intended, not sandbox-enforced*):
  - Inline the entire artifact in the prompt; the peer needs no filesystem access.
    Where practical, launch `agy` from an empty throwaway dir so there's nothing local
    to read.
  - **Do NOT pass `--add-dir`** — it would let Gemini read beyond the redacted
    prompt, breaking the read-only/privacy guarantee.
  - **Do NOT pass `--dangerously-skip-permissions`.**
  - **Inject a hard headless instruction** (see [prompt shaping](#prompt-shaping)):
    `CRITICAL: headless, non-interactive. Do NOT use any tools or read any files.
    Output ONLY the requested JSON.` Without this, `agy` may try a tool and prompt
    for permission on **stdin — which is already at EOF** (consumed by `cat`) → it
    hangs or crashes. `--print-timeout 10m` bounds the hang.
- print-mode **stdout is clean** (no progress noise — unlike Codex), which makes the
  defensive parse easier; `$J.err` is usually empty on success.
- **Availability:** unavailable if `agy` not on PATH; or timed out (`--print-timeout`
  hit / no `$J.done` past deadline); or `$J.out` **empty** after one retry; or
  `$J.err`/`$J.out` matches `rate.?limit|quota|RESOURCE_EXHAUSTED|429|unauthenticated|permission`.
  A **non-empty but unparseable** `$J.out` is *available-but-malformed* (keep as
  `unstructured_advice`), **not** unavailable — don't trigger Opus on its account.
  **Note:** a rate-limited `agy` may exit **0** with empty/garbage stdout — so check
  stdout-emptiness AND grep stderr, don't trust the exit code alone.

### Opus fallback adapter (Anthropic — only if BOTH external peers are down)

- Spawn via the Agent tool / Workflow `agent()` with `model: 'opus'` and a
  **self-contained** prompt — problem statement + the concrete artifact only, **not**
  your conversation history or conclusions (fresh context = no rubber-stamping).
- **Same Anthropic trust boundary as the coordinator → send RAW content, no
  redaction.** Build a fresh raw prompt; do not reuse the redacted peer prompt
  (that would needlessly degrade Opus's input).
- **Caveat:** the `model` param exposes only the family (`opus`), resolving to the
  **current** latest; pinning the exact previous minor may not be selectable. Use
  `opus` + fresh context anyway, and flag that a real Codex/Gemini pass is worth
  redoing once limits reset.
- Same calibration (classifier tier) and read-only/advisor discipline apply.

---

## The contexts

| Context                         | Collaboration pattern              | Peers (default)            |
|---------------------------------|------------------------------------|----------------------------|
| Overall plan / impl plan / pre-work | **Parallel-draft → converge**  | **codex + gemini** (3-way) |
| Implementation review           | **Draft → advisory review → integrate** | classifier (gemini if `big`) |
| Blocker / error diagnosis       | **Advisory diagnosis**             | classifier, floor `small`  |

### Context A — Planning (parallel-draft → converge)

Do **not** draft a plan and hand it over for critique — that anchors the peers on
your framing. Instead:

1. **Dispatch both peers' independent drafts in parallel** with the **same
   source material** (ticket, requirements, relevant code — **redacted**) but **not
   your draft**. Build BOTH prompts from the skeleton with the **planning** role line
   and the **plan** contract — `--output-schema` for Codex, the SAME schema pasted
   inline for Gemini.
2. **While they run, draft your own plan** independently.
3. **Collect all three, diff the ideas** — agreements, real disagreements, anything
   one side caught that the others missed.
4. **Iterate to consensus** ([loop](#the-consensus-loop)) on real disagreements only.
5. Present the converged plan + a "what each peer caught / what we rejected" note.

### Context B — Implementation review (draft → advisory review → integrate)

Triggered when a **work item reaches a complete state**.

1. Build the signal pack and **classify**. If `skip`, say so and move on.
2. Dispatch the selected peer(s) **read-only** at the classified tier over the work
   item's **redacted** diff, using the **review** role line + **review** contract
   (file:line in the contract's "N" or "N-M" format). Explicit "no findings → say so."
3. **Integrate** correct findings (re-expand placeholders first); push back on wrong
   ones. Iterate to consensus; present findings, then "what we fixed / rejected."

### Context C — Blocker / error diagnosis (advisory diagnosis)

Triggered only by the reactive threshold (2+ failed fixes **or** cascading errors).

1. Classify (floor `small`); wide blast radius → `big` (pulls in Gemini).
2. Dispatch peer(s) **read-only** with the **redacted** error, what you've tried,
   and the relevant code, using the **diagnosis** role line + **diagnosis** contract.
   Ask for ranked root-cause hypotheses + a concrete next probe — not a blind patch.
3. Apply the fix yourself; if it fails, escalate a tier and re-pair.

---

## Prompt shaping — ONE skeleton, filled per peer

Assemble EVERY peer prompt from these labeled slots, in this order (the
prompt-engineering *instruction hierarchy*). Same skeleton for both peers and all
three contexts so outputs stay diffable; only the bracketed bits and the Gemini-only
line differ. Assemble **after redaction**. Keep each slot to one line; the artifact
is the only large block.

```text
ROLE: <pick by context — verbatim lines below>
MUST: Return findings/analysis ONLY; do NOT propose to run, edit, or execute
  anything. Output exactly one JSON object matching the contract — nothing before or
  after. Base every claim ONLY on the artifact below; do not assume code / imports /
  behavior you cannot see — if you lack context, list it under "insufficient_context"
  instead of guessing.
[Gemini only] MUST: You are headless and non-interactive. Use NO tools and read NO
  files — everything is inline. No prose outside the JSON object.
TASK: <Review this diff for correctness, then reuse/simplification. | Draft your OWN
  plan; do not assume an approach. | Give ranked root-cause hypotheses + one concrete
  next probe; do not patch blind.>
LENS (plan+review only): Prefer the laziest solution that holds (stdlib → native
  feature → installed dep → one line → minimum code). REVIEW: raise over-engineering as
  findings with category:simplification (speculative abstraction, needless dep,
  indirection for constants, scaffolding-for-later, a diff that could shrink/delete).
  PLANNING: prefer the smallest plan that works and record over-engineering you rejected
  under risks/openQuestions (the plan schema has no findings[]). MUST NOT simplify away
  trust-boundary validation, error handling, security, or explicitly-requested behavior.
CONTEXT: <repo / language / what the change or plan is meant to do — 1–2 lines>
ARTIFACT (redacted):
<diff | error+attempts+code | requirements>
OUTPUT CONTRACT: Return ONLY JSON matching the task schema below.
```

**Role lines (verbatim — pick by context):**

- **review:** You are an adversarial senior code reviewer acting as a read-only
  advisor. Find real correctness defects first, then reuse/simplification wins.
  Findings only; you never edit files.
- **planning:** You are an independent senior engineer drafting your OWN
  implementation plan from the requirements below. You have NOT seen any other plan —
  do not assume an approach; propose what you think is best, then name the risks you'd
  want a reviewer to challenge.
- **diagnosis:** You are a systematic debugger acting as a read-only advisor. From the
  error and what's been tried, produce ranked root-cause hypotheses and ONE concrete
  next probe — not a blind patch.

**Tier nudge (the ONLY richness scaling — progressive disclosure):** at `big`/planning
only, append ONE sentence to TASK, matched to the mode — **review/diagnosis:** *"Before
returning, drop any finding you cannot tie to a specific line and downgrade anything
that's a preference rather than a defect; reason internally before emitting the JSON."*
**planning:** *"Before returning, drop any step you're not confident in and reason
internally before emitting the JSON."* Nothing more — no examples, no
multi-stage verify loop. Codex's `-s read-only` + `--output-schema` make the
Gemini-only MUST line redundant for Codex; include it for Gemini only.

The skeleton, role lines, LENS, and tier nudge are **vendored** from the
`prompt-engineering-patterns` skill — see
[Portability](#portability--swapping-backends); don't shell out to it on the per-turn
hot path. (LENS mirrors the `ponytail` skill.)

**Output contract** (Codex: `--output-schema`; Gemini: paste the shape, parse
defensively, treat a missing field as *unknown*). Reasoning lives in the model's
reasoning channel (Codex stderr / Gemini's internal pass from the tier nudge), **not**
a JSON field — a free-text reasoning field is brittle under `additionalProperties:false`
plus the defensive parser.

```jsonc
// review
{ "type":"object","additionalProperties":false,
  "properties":{
    "findings":{"type":"array","items":{"type":"object","additionalProperties":false,
      "properties":{"severity":{"type":"string","enum":["critical","high","medium","low","nit"]},
        "category":{"type":"string","enum":["correctness","simplification","efficiency"]},
        "file":{"type":"string"},"line":{"type":"string"},
        "issue":{"type":"string"},"suggestion":{"type":"string"},
        "confidence":{"type":"string","enum":["high","med","low"]}},
      "required":["severity","category","file","line","issue","suggestion","confidence"]}},
    "summary":{"type":"string"},"residualRisk":{"type":"string"},
    "insufficient_context":{"type":"array","items":{"type":"string"}}},
  "required":["findings","summary","residualRisk","insufficient_context"] }
//   line = "N" or "N-M", no prose. insufficient_context REQUIRED (emit [] if none).

// diagnosis
{ "type":"object","additionalProperties":false,
  "properties":{
    "hypotheses":{"type":"array","items":{"type":"object","additionalProperties":false,
      "properties":{"cause":{"type":"string"},
        "likelihood":{"type":"string","enum":["high","medium","low"]},
        "evidence":{"type":"string"}},
      "required":["cause","likelihood","evidence"]}},
    "nextProbe":{"type":"string"},"summary":{"type":"string"},
    "insufficient_context":{"type":"array","items":{"type":"string"}}},
  "required":["hypotheses","nextProbe","summary","insufficient_context"] }
//   likelihood stays as the confidence signal. insufficient_context REQUIRED (emit []).

// plan
{ "type":"object","additionalProperties":false,
  "properties":{
    "goal":{"type":"string"},
    "steps":{"type":"array","items":{"type":"object","additionalProperties":false,
      "properties":{"n":{"type":"integer"},"action":{"type":"string"},
        "confidence":{"type":"string","enum":["high","med","low"]}},
      "required":["n","action","confidence"]}},
    "filesTouched":{"type":"array","items":{"type":"string"}},
    "risks":{"type":"array","items":{"type":"string"}},
    "openQuestions":{"type":"array","items":{"type":"string"}},
    "insufficient_context":{"type":"array","items":{"type":"string"}}},
  "required":["goal","steps","filesTouched","risks","openQuestions","insufficient_context"] }
```

**Merge note:** every property is in `required` on purpose — OpenAI's `--output-schema`
strict mode rejects a schema (with `additionalProperties:false`) unless `required` lists
*every* key, so `insufficient_context` is always emitted (`[]` when the peer saw enough).
Gemini isn't schema-enforced — when reconciling, treat a missing `confidence`/field as
*unknown* (not `low`/0). `insufficient_context` is the in-band grounding hatch: a peer
says "I can't see this" instead of degrading to `unstructured_advice`.

---

## The consensus loop

Pairing is **iteration to consensus**, not a single pass. With two peers, the
coordinator is the **executive synthesizer** — not a relay.

1. Collect each peer's structured output. Form your own view. Re-expand placeholders
   before reasoning about findings.
2. Build a quick agreement map: where all agree (high confidence), where a majority
   agrees, and lone-wolf findings (judge on merit).
3. Integrate correct points; push back, with reasoning, on wrong ones. **Don't
   launder one peer's unsupported claim into "consensus."**
4. **Second round only if a material disagreement remains** — send a **compact
   disagreement brief** (claim, evidence, why contested, the specific
   yes/no-answerable question) to the relevant peer(s) only. **Redact any
   freshly-quoted evidence first** — this ad-hoc mini-prompt bypasses the
   assembly-time redaction gate. Run round 2 **free-form (no `--output-schema`)** — the peer
   replies in the same finding/hypothesis shape **plus** `position:hold|change` and an
   updated `confidence` (extra fields a strict `additionalProperties:false` schema would
   reject, which is why this round drops schema enforcement), so a changed position +
   confidence delta are mechanically detectable. Do **not** replay full transcripts,
   and do **not** pipe peers' full outputs to each other (token + leakage + ping-pong
   cost).
5. **Hard cap: 2 peer rounds.** After round 2, the coordinator **decides
   unilaterally** and terminates the loop — record unresolved trade-offs for the
   user to arbitrate. Escalate a tier and re-pair only if the work proved bigger
   than first classified.
6. Present the converged result + a **"what Codex caught / what Gemini caught / what
   the coordinator accepted or rejected and why / residual risk"** note.

Don't accept a peer blindly; don't dismiss it reflexively. The reconciled position
is the deliverable.

---

## Availability & fallback state machine

```text
intended := peers from the fan-out matrix for this verdict
for each peer in intended: attempt once (bounded); classify result
  -> usable result        : keep
  -> unavailable           : drop (rate-limit / auth / timeout / not-on-PATH / empty)
  -> available but malformed: retry once; still malformed -> keep as cautious
                             residual risk; do NOT fall to Opus on its account

if any intended peer produced a usable result:
    proceed with those + the coordinator        # NO Opus, even if the other peer died
elif single-peer tier (intended = codex only) and codex unavailable:
    PROBE gemini now (it was NOT in the intended set); if up, substitute it at the
    tier-equivalent rung (a substitute, not a 2nd peer; don't re-add codex if it recovers).
    if gemini also unavailable -> fall through to both-down
elif BOTH codex AND gemini unavailable:
    spawn ONE previous-gen Opus agent (raw content, fresh context) as the sole peer
else:
    report that no peer was reachable; do not confabulate
```

**"Unavailable" is scoped to the current request** (one bounded attempt) — a
transient Gemini rate-limit must not poison later turns. Distinguish *unavailable*
(rate-limit/auth/timeout → fall back) from *available-but-malformed* (peer answered,
output unusable → retry, but do **not** jump to Opus if the other peer succeeded).
Per-CLI detection patterns live in the [adapters](#peer-adapters).

---

## First-run setup (one-time, per machine)

`llm-pair` triggers reliably only if the host CLAUDE.md tells the agent to use it.
On first invocation:

- Check whether the user's CLAUDE.md (global or project) contains an `llm-pair`
  pairing rule; if not, **offer** to add the wiring block from the README "Setup"
  section. **Do not edit CLAUDE.md without consent.**
- Confirm prerequisites:
  - `codex` CLI on PATH (`brew install codex`) and authenticated.
  - `agy` CLI (Antigravity) on PATH and authenticated (`agy models` should list
    models). Gemini is **optional** — if `agy` is absent, the skill runs Codex-only
    (that's not "both down", so no Opus unless Codex is also down).
  - An Opus-capable agent for the both-down fallback.

---

## Portability — swapping backends

Designed to be open-sourced (`iotashan-llc/llm-pair`). Bindings:

- **Classifier** — Claude Code's REPL `haiku()`. Any fast model with structured
  output works; keep the schema, swap the call.
- **Peer backends** — the `codex` CLI and the `agy` CLI, each a concrete
  [adapter](#peer-adapters) over the [shared dispatch engine](#the-shared-dispatch-engine-dry-core).
  To add/replace a backend, write a new adapter (command template, output mode,
  read-only enforcement, availability detection) + its CONFIG ladder; selection,
  redaction, consensus, and fallback are unchanged.
- **Fallback** — any Opus-capable agent (Claude Code's Agent/Workflow tools).
- **Fan-out policy** — the [fan-out matrix](#fan-out-matrix--who-gets-invoked-default-complex-only)
  is the one knob to change which/when peers are invoked.

- **Prompt templates** — the peer-prompt skeleton, the verbatim role lines, the task
  output contracts, the LENS block, and the tier nudge in
  [Prompt shaping](#prompt-shaping--one-skeleton-filled-per-peer) are **vendored** from
  the `prompt-engineering-patterns` skill (its instruction-hierarchy, task-schema,
  grounding, and progressive-disclosure patterns), baked in as static text. On the
  per-turn hot path the coordinator reads ONLY this file — it does **not** Skill-invoke
  `prompt-engineering-patterns`, read its `references/*.md`, or run its
  `optimize-prompt.py`. Every lookup (which role line, which contract, whether to add
  the tier nudge) is answerable from the vendored text with zero extra tool calls; a
  per-turn skill-load would tax the path that fires on EVERY pairing and defeat the
  cost-calibration mandate, and would break self-containment (a downstream user of
  `iotashan-llc/llm-pair` with no prompt skills installed loses nothing). To re-tune the vendored blocks, a maintainer consults the
  prompt skills (`prompt-engineer`, `enhance-prompt`, `llm-prompt-optimizer`,
  `prompt-engineering-patterns`) **offline** and pastes improved text back in — never a
  runtime call-out, and never running their code inside the read-only / redaction /
  no-confabulate boundary (`prompt-engineering-patterns` is `source:community /
  risk:unknown`).

Local install wiring (skill symlinks, CLAUDE.md edits) is **not** part of the
shippable skill — see the README "Setup" section.
