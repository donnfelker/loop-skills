---
name: multi-llm-convergence
description: "Drive any artifact to convergence by alternating two genuinely different LLM reviewers — a Codex reviewer and an independent Claude review subagent — applying each round's findings and looping until BOTH independently agree it clears the bar (default: no critical/high findings). Artifact-agnostic: works on a plan, PRD, design doc, spec, or implementation diff. Use whenever the user says 'have two LLMs converge on this', 'loop Claude and Codex until they agree', 'review and revise until consensus', 'iterate this plan/PRD/spec until there are no blockers', or wants cross-model agreement rather than one reviewer's opinion. Prefer this over a single review pass whenever the user wants different models to actually agree on an end result. NOT pr-autopilot (one GitHub PR's comment threads) — this is the artifact-agnostic, alternating-different-models, iterate-to-consensus loop."
metadata:
  version: 0.1.0
---

# Multi-LLM Convergence

You are the **convergence driver**. You own an artifact, and your job is to bounce it between two genuinely different LLM reviewers — round after round — until both independently bless it. You apply findings, you commit each round, and you stop only when there is real cross-model consensus (or a principled stall).

**Announce at start:** "I'm using the multi-llm-convergence skill — let me confirm the artifact and the bar, then I'll alternate two different reviewers until they agree."

## Why this skill exists

A single reviewer — even a good one — has blind spots, and a single model has *correlated* blind spots: ask the same model twice and it tends to miss the same things twice. Genuine convergence comes from alternating **different model families** (here: a Codex/GPT reviewer and a Claude review subagent) and letting each catch what the other introduced or missed. This is not theater. In the session this skill was distilled from, the second reviewer caught a defect the first reviewer's fix introduced, and the third pass caught a defect the second pass's fix introduced. Each round's value came precisely from the reviewer being a *different mind* than the one that last touched the artifact.

Three things make or break this loop, and all three are baked into the steps below:

1. **Ground the reviewers in local source-of-truth.** Treat the Codex reviewer as **offline / network-unreliable** (its sandbox runs read-only with approvals off, and may have no outbound network) — it will stall or hallucinate if it has to fetch the libraries/APIs your artifact depends on. Clone them locally *first*.
2. **Never let a reviewer go silent.** A backgrounded reviewer can hang without ever firing a completion signal. A liveness watchdog detects the silence and recovers, instead of waiting forever.
3. **Stop on real consensus, not the first "looks good."** Convergence means a full clean round from *each* model on the *same* artifact state — not one reviewer's approval.

## Inputs to confirm up front

Before looping, get these settled (ask only for what the user didn't already give you):

- **Artifact** — what you're converging. A path to a file (plan/PRD/spec/doc) or a code change (branch/diff). Everything downstream is artifact-agnostic; you just need to know what "apply findings" edits and what "review" reads.
- **The bar** — what "good enough to stop" means. Default: **no findings at critical or high severity from either reviewer.** (Severities are `critical | high | medium | low` — matching the Codex plugin's vocabulary so a real Codex finding classifies cleanly.) The user may raise it ("no medium either") or lower it.
- **Max rounds** — a safety cap so the loop can't run forever. Default: **6 reviewer passes** (≈3 per model).
- **Source-of-truth paths** *(optional)* — repos/docs the reviewers must check claims against. If the artifact references external libraries/APIs and none are provided, you will clone them in Step 1.

## Process flow

```
1. Ground truth      → clone/fetch external deps into gitignored tmp/ so offline reviewers don't stall
2. Baseline          → ensure the artifact exists and is committed (round 0)
3. Review (model A)  → dispatch reviewer, RUN THE LIVENESS WATCHDOG, collect findings
4. Apply + commit    → apply findings above-bar, commit "converge: round N (<model> review)"
5. Review (model B)  → the OTHER model reviews the updated artifact (watchdog again)
6. Apply + commit
7. Converge check    → two consecutive clean passes (one per model, no edits) → DONE
                       else loop to step 3, alternating models, until clean or cap/oscillation
8. Final output      → converged artifact + round-by-round convergence log + explicit verdict
```

## Step 1 — Establish ground truth FIRST

This is step one, not an afterthought. The single biggest failure mode of this loop is a reviewer that stalls trying to reach the network it doesn't have.

- Identify every external library, framework, SDK, API, or service the artifact makes claims about.
- For each, get a **local copy the reviewers can read**: `git clone --depth 1 <repo> tmp/<name>` into a `tmp/` directory, and/or save the relevant docs locally. Add `tmp/` to `.gitignore` so it never lands in a commit.
- Note the exact version/tag/SHA you cloned. Reviewers must verify claims against *that*, not against their training memory.
- Record the absolute paths — you will hand them to **both** reviewers in every dispatch.

If the artifact is purely internal (no external dependencies to verify), say so and skip the cloning, but still tell each reviewer which in-repo paths are the source of truth.

> Why this matters: a reviewer asked to verify "does library X's function still exist" will `curl` GitHub, fail (no network → repeated `curl` exit 6), and go quiet — looking exactly like a hang. Local clones turn "fetch and stall" into "read a file."

## Step 2 — Baseline (round 0)

Make sure the artifact exists and is committed before any review, so every later round is a clean diff.

- If the user is converging an existing file/branch, confirm it's committed (commit it if not).
- If the artifact doesn't exist yet (e.g. "converge on a PRD for X"), draft an initial version yourself, then commit it as round 0.
- If the working directory isn't a git repo, `git init` it — per-round commits are the convergence audit trail and are not optional. (Uncommitted work is fragile: a reset or re-clone erases it entirely.)

## Steps 3 & 5 — Dispatch a reviewer (alternating models)

Reviewers alternate every round. The rule is simple: **the model that reviews next is never the model that last edited the artifact.** Concretely:

- **Reviewer A = Codex** (a non-Claude model). Run it **read-only** through the free-form `codex-companion task` path — *not* the `review`/`adversarial-review` commands (those are git-diff-bound and don't fit a plan/PRD/doc), and *not* the `codex:codex-rescue` agent (it defaults to a write-capable run and only forwards, exposing no job-control for the watchdog). The `task` path lets you send the review contract and gives you native `status`/`result` job-control for liveness. See `references/reviewer-dispatch.md` for the exact command, flags, and script-path resolution.
- **Reviewer B = an independent Claude review subagent** (`subagent_type: "general-purpose"`), started fresh with no prior context. Same contract, different model family.

Each reviewer gets the **same** review contract (full template in `references/reviewer-dispatch.md`): the artifact, the bar, the source-of-truth paths from Step 1, and a required output shape — a JSON list of findings, each with `severity` (critical | high | medium | low), `location`, `claim`, `verified_against`, and `suggested_fix`, plus a final `verdict` (`clears_bar: true|false`). Demand the structured output so you can mechanically decide convergence instead of parsing prose.

**Run reviewers one at a time, not in parallel.** Unlike a single-pass multi-reviewer sweep, convergence is inherently sequential: model B must review what model A's findings produced. Parallelizing defeats the point.

### Steps 3a & 5a — The liveness watchdog (REQUIRED on every dispatch)

Tool names in this section (`ScheduleWakeup`, `Agent(...)`, `TaskList`/`TaskGet`/`TaskOutput`) are illustrative capability names — map them to whatever the running host exposes for scheduling a self-wake, dispatching a background subagent, and polling a background task respectively.

> **The one rule that prevents babysitting:** never end your turn while a background reviewer is in flight unless a `ScheduleWakeup` is already armed to wake you. The runtime re-invokes you at that scheduled time **whether or not the reviewer ever finishes** — so a hung, silent reviewer cannot strand the loop. A dispatch that yields the turn with no wake armed *is* the bug. Arm the wake in the same turn you dispatch, every time.

A backgrounded reviewer — Codex especially — can stall and **never fire a completion notification**. Passively awaiting that notification is how the loop hangs forever — because the only thing that would wake you is a signal that never comes. The self-armed `ScheduleWakeup` is what makes the loop autonomous: it fires on a clock, not on the reviewer's cooperation. So every dispatch is supervised. The exact liveness signal depends on the path:

- **Codex (`codex-companion task --background`):** the command returns a **job id**. Poll `node <companion> status <job-id> --json` each tick — read the authoritative `job.status`, whose values are `queued | running | completed | failed | cancelled` (note `done` is a *phase*, not a status), not a guess from file mtime. Fetch the final output with `result <job-id> --json`. (`status --json` reads instantaneous state and returns immediately; it is independent of the companion's own internal `--wait` timeout, so your tick cadence is yours to choose.)
- **Claude subagent (`Agent(run_in_background: true)`):** capture the returned agent id and poll `TaskList`/`TaskGet`/`TaskOutput` for liveness and output growth.

The supervision loop, either path:

1. **Dispatch in the background**, capturing the job/agent id.
2. **Self-pace a poll** with `ScheduleWakeup`, waking about **every 270 seconds** — under the 5-minute prompt-cache window so each check is cheap and cache-warm. This self-paced heartbeat is what keeps a silent reviewer from hanging the loop; drive it with `ScheduleWakeup` directly (do not invoke the user-facing `/loop` command — that's for a human setting up a recurring task, not for pacing inside a running skill). **If `ScheduleWakeup` isn't available in this host, fall back** to a *foreground* `task` run under a hard time cap (see references) so a hang surfaces as a non-zero exit instead of an infinite wait — you lose "still thinking vs stuck" discrimination but keep the no-hang guarantee. Use a portable cap: GNU `timeout` is absent on macOS/BSD, so prefer `perl -e 'alarm shift; exec @ARGV' <secs> …` (or `gtimeout` from coreutils) there.
3. **Each tick, read job state + output growth.** Any terminal `job.status` (`completed`/`failed`/`cancelled`) means proceed (read the result). A still-active job (`queued`/`running`) whose **output has advanced** is healthy — let it keep going.
4. **Stall verdict:** `job.status` is still `queued`/`running` AND output unchanged across **2 consecutive ticks** with no result → declare it hung. Don't keep waiting; don't blindly re-dispatch the same way (it stalls identically).
5. **Recover by fixing the cause, not retrying blind:** confirm the Step-1 `tmp/` clones exist and are complete, then re-dispatch with explicit instructions to read those local paths and make **no network calls**. If it stalls *again* after grounding, stop and escalate to the user with what you observed.
6. **Fallback wakeup:** also set one long `ScheduleWakeup` (~1200 s) as a backstop, so even if a completion notification never arrives and a poll is missed, the loop still wakes and re-checks rather than dying silently.

> The detection signal that matters is job *state plus output advancement*, not elapsed time. A reviewer legitimately thinking for four minutes (job `running`, output still trickling) is fine; a `running` job whose output hasn't moved across two checks is stuck. Prefer `status --json` over raw file mtime precisely so a quiet-but-thinking job isn't misjudged as hung.

## Steps 4 & 6 — Apply findings and commit the round

- Apply every finding **at or above the bar**. Findings below the bar (e.g. `medium`/`low` when the bar is `high`) are recorded in the log but don't block convergence; apply them only if cheap and clearly correct.
- When a reviewer's claim touches an external library, verify it against the Step-1 local clone before acting — reviewers (both models) hallucinate APIs. If the clone contradicts the finding, record it as **rejected (contradicted by source)** in the log and don't apply it.
- Commit each round on its own: `converge: round <N> — apply <model> review`. Per-round commits are the auditable trail; they also let you `diff` to detect oscillation (Step 7).

## Step 7 — Convergence, stall, and oscillation

**Converged (success):** the artifact has reached convergence when **two consecutive reviewer passes — one from each model — both return `clears_bar: true` with no above-bar findings, AND neither pass required an edit.** That guarantees both model families independently bless the *same* artifact state. If a pass produces any above-bar finding, you apply it and the clean streak resets — the *other* model must then re-bless the edited version.

**Stalled (cap reached):** if you hit the max-rounds cap without two consecutive clean passes, stop. Don't pretend consensus. Emit the stall report (below) naming the findings still in dispute.

**Oscillation:** if the two reviewers flip-flop — model A wants change X, you apply it, model B wants it reverted, model A wants it back — you are not converging, you are ringing. **Concrete trigger:** key findings by their `location`; if the *same* `location` gets changed and then reverted (its change direction flips) **twice** across rounds, that's oscillation. When you hit it, **stop and surface the disagreement to the user** rather than burning rounds — quote what each model argues about that location. A genuine cross-model disagreement is a human decision, not something to average away. This is a signal, not a failure of the loop.

## Step 8 — Final output

Whatever the outcome, produce:

1. **The converged artifact** (already committed round-by-round).
2. **A convergence log** — one row per round: round number, which model reviewed, count of findings by severity, what you applied, what you rejected and why (esp. source-contradicted claims), and that round's verdict. This is the proof of *how* consensus was reached, and it's where the cross-model catches show up.
3. **An explicit verdict**, one of:
   - `CONVERGED` — "Both <model A> and <model B> independently cleared the bar (<bar>) on the same artifact state, rounds X and Y. No above-bar findings remain."
   - `STALLED (cap)` — "Reached <N> rounds without consecutive clean passes. Open above-bar findings: …"
   - `STALLED (oscillation)` — "Reviewers disagree on <point>; this needs your decision. <model A> argues …; <model B> argues …."
   - `STALLED (reviewer unavailable)` — "<model> hung after grounding and could not complete; converged only against <the other model>."

Keep the user-facing summary tight; the per-round commits and the log are the source of truth.

## Edge cases & failure modes

- **Only one reviewer is available** (e.g. Codex plugin not installed). You cannot do cross-model convergence with one model. Tell the user, and offer either (a) install the Codex plugin from https://github.com/openai/codex-plugin-cc (then run `/codex:setup` to authenticate) and re-run, or (b) proceed as a single-model iterate-to-clean loop, clearly labeled as NOT cross-model consensus. Don't silently degrade.
- **Reviewer returns prose instead of the JSON contract.** Re-request once with the schema restated. If it still won't comply, parse conservatively and note the degraded parsing in the log — never invent a `clears_bar` verdict the reviewer didn't give.
- **A finding can't be verified against any source** (no local clone, claim depends on runtime data). Treat it like an unsubstantiated claim: record it, don't auto-apply a risky change on its basis, and flag it for the user.
- **Artifact has no external deps.** Skip cloning; still name the in-repo source-of-truth paths so reviewers verify against code, not memory.
- **The loop wakes from a `ScheduleWakeup` and the reviewer already finished.** Good — read the result, cancel the redundant fallback wakeup, and proceed. A wasted wake is cheap; a missed hang is not.

## What NOT to do

- **Don't use the same model for both reviewer slots.** Correlated blind spots = fake consensus. Different families is the entire mechanism.
- **Don't end your turn with a reviewer in flight and no `ScheduleWakeup` armed.** That is *the* babysitting bug: you'd be waiting on a completion notification that a hung reviewer never sends. The self-armed wake fires on a clock regardless — arm it every dispatch, no exceptions.
- **Don't passively await a background reviewer without the watchdog.** That's the exact way the loop hangs. Every dispatch gets supervised.
- **Don't restart a stalled reviewer blind.** Fix the cause (grounding) first; a blind retry stalls identically.
- **Don't declare convergence on a single "looks good."** It takes two consecutive clean passes, one per model, with no edits.
- **Don't apply a finding that the source-of-truth clone contradicts.** Record it as rejected; reviewers hallucinate library APIs.
- **Don't average away a real disagreement.** Oscillation between models is a human decision — surface it, don't paper over it.
- **Don't skip the per-round commits.** The convergence trail is the deliverable's credibility.
- **Don't run the two reviewers in parallel.** Convergence is sequential by construction — B reviews what A's round produced.
