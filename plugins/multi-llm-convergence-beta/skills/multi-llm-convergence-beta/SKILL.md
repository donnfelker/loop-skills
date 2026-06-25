---
name: multi-llm-convergence-beta
description: "(beta/experimental) Direct-CLI, configurable, N-model variant of multi-llm-convergence: drives any artifact (plan, PRD, design doc, spec, or implementation diff) to cross-model consensus by reaching each model through its own CLI (claude, codex, gemini, grok) instead of the Codex companion script, with a JSON adapter registry you extend by a data edit. Use when the operator wants to choose which models converge (≥2 distinct families), converge with three or more models, run from a non-Claude host, or add a new LLM CLI without rewriting the skill. Trigger on 'beta convergence', 'converge with codex and gemini', 'N-model convergence', or 'direct-CLI convergence'. For the stable, hardcoded two-model (Claude + Codex companion) loop, use multi-llm-convergence instead."
metadata:
  version: 0.1.0
---

# Multi-LLM Convergence (beta)

> **Beta/experimental.** This is the direct-CLI, configurable, N-model variant. It ships
> in parallel with the stable `multi-llm-convergence` (two hardcoded models via the Codex
> companion), which stays untouched. Invoke this one deliberately. When it proves out it is
> promoted into the original namespace and this plugin is removed (see the plugin README).

You are the **convergence driver**. You own an artifact, and your job is to bounce it between a
set of **genuinely different** LLM reviewers — round after round — until they *all* independently
bless it. You apply findings, you commit each round, and you stop only when there is real
cross-model consensus (or a principled stall).

**Announce at start:** "I'm using the multi-llm-convergence-beta skill — let me detect the host,
confirm the reviewer set and the bar, then I'll rotate the selected reviewers until they all agree."

## Why this skill exists

A single reviewer — even a good one — has blind spots, and a single model has *correlated* blind
spots: ask the same model twice and it tends to miss the same things twice. Genuine convergence
comes from alternating **different model families** and letting each catch what the other introduced
or missed. The stable skill hardcodes that as "Claude driver + one Codex reviewer." This beta
generalizes it three ways without losing the guarantees:

1. **Direct-CLI delegation.** Reach every model by shelling out to its own CLI — `codex exec`,
   `claude -p`, `gemini -p`, `grok -p` — instead of the Codex companion script. One symmetric
   primitive, no plugin dependency.
2. **Configurable adapter registry.** A JSON catalog (`assets/adapters.json`) of argv-array
   invocation templates. Adding a new LLM is a data edit, not a skill rewrite.
3. **User-selected reviewer set.** At startup the operator picks which models converge (≥2 distinct
   families). The **host** (the agent this skill runs inside) is a pure orchestrator and need not be
   one of the reviewers.

Everything that made the original trustworthy is preserved: local grounding so offline reviewers
don't stall, a liveness watchdog so a silent reviewer can't hang the loop, per-round commits, and a
hard stop when genuine cross-model consensus is impossible.

## Inputs to confirm up front

Ask only for what the operator didn't already give you:

- **Artifact** — what you're converging: a path to a file (plan/PRD/spec/doc) or a code change
  (branch/diff). Everything downstream is artifact-agnostic; you just need to know what "apply
  findings" edits and what "review" reads.
- **Reviewer set** — which adapters converge (Step 0b). **≥2 distinct `family` values.** A skill
  argument may preselect (e.g. `… codex,gemini`) to skip the prompt.
- **The bar** — what "good enough to stop" means. Default: **no findings at critical or high
  severity from any reviewer.** Severities are `critical | high | medium | low`. The operator may
  raise it ("no medium either") or lower it.
- **Max cycles** — a safety cap so the loop can't run forever. Default: **3 full cycles** (≈3 passes
  per model). Warn on cost when ≥3 models are selected.
- **Source-of-truth paths** *(optional)* — repos/docs the reviewers must check claims against. If
  the artifact references external libraries/APIs and none are provided, you clone them in Step 1.

## Process flow

```
0a. Detect host       → best-effort family via detect_env; else generic orchestrator
0b. Select reviewers  → probe installed adapters, operator picks ≥2 distinct families
0c. Preflight (HARD)  → functional smoke-test each selected reviewer; STOP if <2 pass
1.  Ground truth      → clone external deps into gitignored tmp/ (per reviewer needs)
2.  Baseline          → ensure artifact exists and is committed (round 0)
3.  Review (round-robin) → dispatch next reviewer (≠ last editor), watchdog, collect findings
4.  Apply + commit    → apply above-bar findings, commit "converge: round N (<model>)"
5.  Converge check    → full clean cycle (one clean pass per selected model, no edits) → DONE
                        else continue rotation until clean / cap / oscillation
6.  Final output      → converged artifact + per-round log + explicit verdict
```

The registry, the dispatch recipes, the watchdog, and read-only handling live in
`references/reviewer-dispatch.md`. The field schema and how to add a CLI live in
`references/adding-an-adapter.md`. The behavioral rules that bind both driver and reviewers live in
`references/coding-guidelines.md`.

## Step 0a — Detect the host (best-effort)

Load the registry: read the shipped `${CLAUDE_PLUGIN_ROOT}/skills/multi-llm-convergence-beta/assets/adapters.json`,
then deep-merge (by `id`) an optional user override at
`${CLAUDE_CONFIG_DIR:-$HOME/.claude}/multi-llm-convergence/adapters.json` if present (so plugin
updates don't clobber operator adapters).

For each adapter, if any of its `detect_env` vars is set, that adapter's `family` is the host. First
match wins. No match ⇒ host unknown ⇒ generic orchestrator (still valid). **Detection is an
optimization only** — it unlocks `native_subagent` dispatch and lets the preflight skip requiring the
host's own CLI. It is never load-bearing: the hard requirement is "≥2 selected reviewers pass a
functional smoke test," never "the host was identified." (Markers: Claude Code sets `CLAUDECODE`;
Codex sets `CODEX_SANDBOX`/`CODEX_SANDBOX_NETWORK_DISABLED` *only when sandboxed*; Gemini injects no
reliable "running" marker.)

## Step 0b — Select the reviewer set

1. Probe every registry adapter for availability: `command -v <bin>`.
2. Present the **available** adapters and have the operator pick **≥2 with distinct `family`**.
   Selecting two adapters with the *same* `family` is rejected — that's fake consensus, not
   cross-model agreement. Default suggestion: host family + one other (if the host is known), else
   the first two available.
3. A skill argument may preselect (e.g. `… codex,gemini`) to skip the prompt.
4. If ≥3 models are selected, **warn that cost grows linearly per cycle** before proceeding.

## Step 0c — Preflight hard stop (functional smoke test)

Smoke-test each **selected** reviewer along the **dispatch path it will actually use** — its CLI for
a CLI reviewer, its `native_subagent` for a host-native one (a host-family reviewer dispatched as a
native subagent need not have its CLI installed). Run the adapter's `smoke_test` argv (a trivial
prompt): exit 0 with the OK token ⇒ installed, authenticated, and able to reach its API.

**Read-only is best-effort — flag + contract.** Each adapter runs with the strongest read-only flag
its CLI offers (codex `-s read-only` is a hard sandbox; claude/grok/gemini use `plan` mode, which is
softer), and the review contract says "review only; do NOT edit." A reviewer only reads the artifact
and emits JSON, so flag + contract is enough for the **trusted** artifacts this loop converges. (If
you point it at untrusted code, run that reviewer under an OS sandbox — Docker or macOS Seatbelt — but
that's not the default.)

**Hard stop if fewer than 2 selected reviewers pass** the smoke test. Print what failed and how to fix
it: install/authenticate the CLI; for a sandboxed host, re-run with network enabled. **No silent
single-model degrade.** The operator may explicitly opt into a single-model iterate-to-clean run,
clearly labeled **NOT cross-model consensus**.

> **Network gotcha:** when the host is a sandboxed Codex (or any network-disabled host), every
> reviewer CLI it spawns inherits the disabled network and fails the smoke test. The message tells the
> operator to run the host with network access.

## Step 1 — Establish ground truth FIRST

Unchanged from the original, and it is step one for a reason: the single biggest failure mode is a
reviewer that stalls trying to reach a network it doesn't have.

- Identify every external library, framework, SDK, API, or service the artifact makes claims about.
- For each, get a **local copy the reviewers can read**: `git clone --depth 1 <repo> tmp/<name>`,
  and/or save the relevant docs locally. Add `tmp/` to `.gitignore` so it never lands in a commit.
- Note the exact version/tag/SHA. Reviewers verify against *that*, not against training memory.
- Record the absolute paths — hand them to **every** reviewer in every dispatch.

If the artifact is purely internal (no external deps), say so and skip cloning, but still name the
in-repo source-of-truth paths. Local clones turn "fetch and stall" into "read a file."

## Step 2 — Baseline (round 0)

Unchanged: ensure the artifact exists and is committed before any review, so every later round is a
clean diff. If it doesn't exist yet, draft an initial version and commit it as round 0. If the
working directory isn't a git repo, `git init` it — per-round commits are the convergence audit trail
and are not optional.

## Step 3 — Dispatch a reviewer (round-robin)

For the **next reviewer in rotation** (never the model whose findings were just applied):

- Build the command from the adapter's `invoke` argv, substituting `{prompt}` (the shared contract
  from `references/reviewer-dispatch.md`), and — for adapters that declare them — `{schema_file}`
  (the findings JSON Schema the orchestrator writes to a temp file) and `{out_file}` (a temp output
  path). Append `stream_invoke_extra` for the streamed/liveness variant.
- **Cross-family** ⇒ run the CLI in the background. **Same-family-as-host with a `native_subagent`**
  ⇒ dispatch that instead (e.g. a Claude Task subagent), ideally without edit tools; else use the CLI.
- **Run reviewers one at a time.** Convergence is sequential by construction — model B must review
  what model A's round produced. Parallelizing defeats the point.
- Read the structured findings via the adapter's `result` (`out_file` / `stdout` / a per-adapter JSON
  path — each JSON-output CLI has its own envelope key). Every adapter prompt-enforces the findings
  contract, so even when the envelope key is wrong the JSON can be extracted from the output text.
- **A reviewer must return a usable verdict.** If you can't parse findings / `clears_bar` from its
  output (prose, refusal, junk), re-request **once** with the contract restated. Still unparseable ⇒
  log it **non-compliant** and treat the round as unresolved — it is **not** a clean pass. Never
  assume a `clears_bar` the reviewer didn't give: silence is not approval.

### Step 3a — Liveness watchdog (REQUIRED on every dispatch)

> **The one rule that prevents babysitting:** never end your turn while a background reviewer is in
> flight unless a `ScheduleWakeup` is already armed to wake you. The runtime re-invokes you at that
> scheduled time **whether or not the reviewer ever finishes**, so a hung, silent reviewer cannot
> strand the loop. A dispatch that yields the turn with no wake armed *is* the bug.

One supervision model for any backgrounded CLI (full per-adapter detail in
`references/reviewer-dispatch.md`):

1. **Dispatch in the background**, capturing the process handle / log path. Stream the
   `stream-json`/`--json` variant to a log file.
2. **Self-pace a poll** with `ScheduleWakeup` (~270 s, under the 5-min prompt-cache window). Never
   end the turn with a reviewer in flight and no wake armed.
3. **Each tick:** process still alive + log advanced ⇒ healthy. Terminal exit ⇒ read result
   (non-zero exit ⇒ treat as stall/fail).
4. **Stall verdict:** alive but log unchanged across **2 consecutive ticks** with no result ⇒ hung.
5. **Recover by fixing the cause**, not retrying blind: confirm the Step-1 clones exist and are
   complete, then re-dispatch with explicit instructions to read those local paths and make **no
   network calls**. If it stalls *again* after grounding, escalate to the operator.
6. **Long fallback `ScheduleWakeup`** (~1200 s) as a backstop, so a missed poll still wakes the loop.
7. **Fallback where no self-wake exists:** run foreground under a portable hard cap
   (`perl -e 'alarm shift; exec @ARGV' <secs> …`, since GNU `timeout` is absent on macOS/BSD) so a
   hang surfaces as a non-zero exit, not an infinite wait. The host-native subagent path keeps its
   existing task-polling (`TaskList`/`TaskGet`/`TaskOutput`) for liveness.

> The signal that matters is *state plus output advancement*, not elapsed time. A reviewer
> legitimately thinking for four minutes (process alive, log still trickling) is fine; a live process
> whose log hasn't moved across two checks is stuck.

## Step 4 — Apply findings and commit the round

Apply findings under `references/coding-guidelines.md` (think first, minimal, surgical, verifiable):

- Apply every finding **at or above the bar**. Below-bar findings are recorded in the log but don't
  block convergence; apply them only if cheap and clearly correct.
- When a reviewer's claim touches an external library, **verify it against the Step-1 clone before
  acting** — reviewers (every model) hallucinate APIs. If the clone contradicts the finding, record
  it as **rejected (contradicted by source)** and don't apply it.
- Commit each round on its own: `converge: round <N> — apply <model> review`. Per-round commits are
  the auditable trail and let you `diff` to detect oscillation.

## Step 5 — Convergence, stall, oscillation (N models)

- **Converged (success):** a full cycle in which **every selected model returns `clears_bar: true`
  with no above-bar findings, and no edit was applied anywhere in the cycle.** Equivalently, N
  consecutive clean reviewer passes (one per model) with no intervening edit — so every model
  independently blessed the *same* final state. Any above-bar finding ⇒ apply it, commit, **reset the
  clean streak**; the rotation must rebuild N clean passes. (With 2 models this reduces exactly to the
  original "two consecutive clean passes, one per model.")
- **Stalled (cap):** hit the max-cycles cap without a full clean cycle ⇒ stop, emit the stall report
  naming open above-bar findings. Don't pretend consensus.
- **Reviewer never reports:** a model that stays non-compliant after its re-request can't produce a
  clean pass, so it blocks convergence — it is never rounded up to "yes." If it never reports within
  the cap, end with `STALLED (reviewer unavailable)` (converged only against the others), never
  `CONVERGED`. If dropping it would leave fewer than 2 reviewers, hard stop.
- **Oscillation:** key findings by `location`; if the same `location`'s change direction flips
  **twice** across rounds, stop and surface the cross-model disagreement to the operator with each
  model's argument. A genuine cross-model disagreement is a human decision — don't average it away.

## Step 6 — Final output

1. **The converged artifact** (already committed round by round).
2. **A convergence log** — one row per round: round number, which model reviewed, findings by
   severity, what was applied, what was rejected and why (esp. source-contradicted claims), and that
   round's verdict.
3. **An explicit verdict**, one of:
   - `CONVERGED` — "All of <models> independently cleared the bar (<bar>) on the same artifact state,
     rounds X…Z. No above-bar findings remain."
   - `STALLED (cap)` — "Reached <N> cycles without a full clean cycle. Open above-bar findings: …"
   - `STALLED (oscillation)` — "Reviewers disagree on <location>; this needs your decision. <model A>
     argues …; <model B> argues …."
   - `STALLED (reviewer unavailable)` — "<model> hung after grounding and could not complete;
     converged only against <the others>."

Keep the user-facing summary tight; the per-round commits and the log are the source of truth.

## Risks & cautions

- **Read-only is best-effort (flag + contract), not a hard jail.** codex has a hard `-s read-only`
  sandbox; claude/grok/gemini use their `plan` mode, which is softer (gemini's, for instance, still
  permits writes to a plans dir and `web_fetch`). Reviewers only read and emit JSON and the contract
  says "don't edit," which is enough for the trusted artifacts this loop converges. To review
  **untrusted** code, run the reviewer under an OS sandbox (Docker, or macOS Seatbelt) — not the
  default. (Each adapter recipe still uses the strongest read-only flag its CLI offers.)
- **`gemini` and `grok` are validate-on-first-use** — neither was locally smoke-tested at design
  time (see the `note` fields in `adapters.json`).
- **Cost with ≥3 models** grows linearly per cycle; surfaced as a warning at selection, not a silent
  default.
- **Same underlying model behind two ids** (e.g. two Claude-based wrappers) is fake diversity;
  mitigated by the distinct-`family` selection rule (Step 0b) and documented in `adding-an-adapter.md`.

## What NOT to do

- **Don't soften the Step 0c hard stop.** Fewer than 2 reviewers passing the smoke test means
  cross-model convergence is impossible — stop before grounding/baseline rather than discovering it
  mid-loop or quietly running one model.
- **Don't assume a verdict a reviewer didn't give.** Unparseable output ⇒ re-request once, then log it
  non-compliant. Silence is not approval.
- **Don't pick two adapters with the same `family`.** Correlated blind spots = fake consensus.
- **Don't end your turn with a reviewer in flight and no `ScheduleWakeup` armed.** That's the
  babysitting bug; the self-armed wake fires on a clock regardless.
- **Don't restart a stalled reviewer blind.** Fix the cause (grounding) first; a blind retry stalls
  identically.
- **Don't declare convergence before a full clean cycle.** Every selected model must clear the bar on
  the same state with no intervening edit.
- **Don't apply a finding the source-of-truth clone contradicts.** Record it as rejected.
- **Don't average away a real disagreement.** Oscillation between models is a human decision.
- **Don't skip the per-round commits.** The convergence trail is the deliverable's credibility.
- **Don't run reviewers in parallel.** Convergence is sequential — B reviews what A's round produced.
