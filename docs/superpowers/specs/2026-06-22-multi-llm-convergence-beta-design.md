# Design: `multi-llm-convergence-beta`

**Date:** 2026-06-22
**Status:** Approved design — pending implementation plan
**Branch:** `donnfelker/dual-agent-delegation`

## Summary

Re-architect the `multi-llm-convergence` skill from a hardcoded two-model loop
(Claude driver delegating to a Codex reviewer via the Codex Claude Code plugin)
into a **host-agnostic, configurable, N-model convergence loop** that delegates
to any agent CLI directly.

Ships as a **new, parallel plugin** `multi-llm-convergence-beta` so the existing
`multi-llm-convergence` (v0.3.0) stays untouched and both can be installed and
exercised side by side. When the beta proves out, it is promoted into the
original namespace and the beta plugin is removed (see [Promotion path](#promotion-path)).

Three changes drive the rewrite:

1. **Direct-CLI delegation (Approach B).** Reach every model by shelling out to
   its own CLI — `codex exec`, `claude -p`, `gemini -p`, `grok -p` — instead of
   the Codex companion script. One symmetric primitive, no plugin dependency.
2. **Configurable adapter registry.** A JSON catalog of LLM adapters with
   argv-array invocation templates. Built-ins for `claude`, `codex`, `gemini`,
   `grok`; a documented schema to add any other CLI (e.g. a future tool) without
   touching the skill body.
3. **User-selected reviewer set.** At startup the skill asks which LLMs should
   converge (≥2 distinct families). The host is a pure orchestrator and need not
   be one of the reviewers.

## Goals / Non-goals

**Goals**
- Run from any host agent (Claude Code, Codex, Gemini CLI, …) — not Claude-only.
- Let the operator pick which models converge, from a configurable set.
- Keep the original loop's guarantees: local grounding, a liveness watchdog so a
  silent reviewer can't hang the loop, per-round commits, and a hard stop when
  genuine cross-model consensus is impossible.
- Make adding a new LLM a data edit, not a skill rewrite.

**Non-goals**
- Replacing or modifying the existing `multi-llm-convergence` plugin (untouched
  until promotion).
- Building a "Claude companion" or any per-host wrapper script. Delegation is
  always "exec the other CLI."
- Guaranteeing every listed CLI works on every machine. Correctness rests on a
  per-reviewer functional smoke test, not on assumptions about flags.

## Grounded facts (verified, not assumed)

All flags below were confirmed to *exist* (names, enums) from live `--help` on the
dev machine and/or official docs via context7; detection markers likewise. Flag
existence is not proof of read-only *behavior* — that is verified per reviewer by
the preflight negative probe (Step 0c), not assumed from `--help`.

| Adapter | Reviewer invocation (read-only, structured) | Source |
|---------|---------------------------------------------|--------|
| `claude` | `claude -p --output-format json --permission-mode plan` (`plan` = read-only; a bare `--disallowedTools "Edit Write NotebookEdit"` still leaves `Bash` write-capable) | live `--help` |
| `codex` | `codex exec -s read-only --skip-git-repo-check --output-schema <schema> -o <out> "<prompt>"` | live `--help` + context7 `/openai/codex` |
| `gemini` | `gemini --approval-mode plan --output-format json -p "<prompt>"` (`plan` = read-only, enforced by the built-in Tier-1 policy) | context7 + cloned `/google-gemini/gemini-cli` docs |
| `grok` | `grok -p "<prompt>" --output-format json --permission-mode plan` (`plan` = read-only; no `read-only` value — enum is `default, acceptEdits, auto, dontAsk, bypassPermissions, plan`) | live `--help` |

Codex sandbox modes are exactly `read-only | workspace-write | danger-full-access`.
Gemini structured output: `--output-format json` (parse `.response`) or
`stream-json` for streaming/liveness. Grok exposes
`--output-format plain|json|streaming-json` (note: `streaming-json`, not `stream-json`).

**Host-detection markers (best-effort — see [Detection](#step-0a--detect-the-host-best-effort)):**

| Host | Marker | Reliability |
|------|--------|-------------|
| Claude Code | `CLAUDECODE` set | Reliable (verified live in-process) |
| Codex | `CODEX_SANDBOX=seatbelt` / `CODEX_SANDBOX_NETWORK_DISABLED=1` | Set **only when sandboxed** (the default). Absent under `danger-full-access`; `seatbelt` value is macOS-specific. Per codex `AGENTS.md`. |
| Gemini | — | **No auto-injected marker.** `GEMINI_SANDBOX` is a user toggle to *enable* sandboxing, not a "running" signal. Not reliable. |
| Other / unknown | — | Treated as a generic orchestrator. |

**Consequence:** host detection cannot be load-bearing. The loop's hard
requirement is "**≥2 selected reviewers pass a functional smoke test**," never
"the host was identified." Detection is an optimization only (native
same-family subagent; skip requiring the host's own CLI; tailored messages).

## Architecture

### Host vs. reviewers (decoupled)

- **Host** = the agent the skill is running inside, detected best-effort. Acts as
  the **orchestrator**: confirms inputs, grounds dependencies, applies findings,
  commits per round, runs the watchdog. The host does **not** have to review.
- **Reviewer set** = the ≥2 adapters the operator selects. Each runs as a fresh,
  isolated context. A reviewer whose family matches the host *may* be dispatched
  via the host's native fresh-context mechanism (e.g. a Claude Task subagent)
  instead of its CLI; every other reviewer is dispatched via its CLI.

### Adapter registry (JSON)

Canonical data file shipped with the plugin:
`skills/multi-llm-convergence-beta/assets/adapters.json`.

Each adapter is a record:

```json
{
  "id": "codex",
  "family": "codex",
  "bin": "codex",
  "detect_env": ["CODEX_SANDBOX", "CODEX_SANDBOX_NETWORK_DISABLED"],
  "invoke": ["exec", "-s", "read-only", "--skip-git-repo-check",
             "--output-schema", "{schema_file}", "-o", "{out_file}", "{prompt}"],
  "stream_invoke_extra": ["--json"],
  "result": { "from": "out_file" },
  "smoke_test": ["exec", "-s", "read-only", "--skip-git-repo-check",
                 "Reply with the single token: OK"],
  "native_subagent": null
}
```

Field contract:

| Field | Meaning |
|-------|---------|
| `id` | Unique adapter id; selecting two with the same `family` is rejected (no fake consensus). |
| `family` | Model family for the "genuinely different model" guarantee. Defaults to `id`. |
| `bin` | Binary checked with `command -v`. |
| `detect_env` | Env vars that, if set, mark this adapter as the host. Empty = never auto-detected as host. |
| `invoke` | **argv array** (no shell string). Placeholders `{prompt}`, `{schema_file}`, `{out_file}`, all substituted at dispatch: the multi-line review contract is injected as the single `{prompt}` element (no shell-escaping), and the orchestrator writes the findings-contract JSON Schema to a temp `{schema_file}` and picks a temp `{out_file}`. An adapter whose CLI has no schema flag simply omits those two placeholders. |
| `stream_invoke_extra` | Extra argv appended for the streamed/liveness variant (e.g. `--json`, `--output-format stream-json`). |
| `result` | Where the final structured text is read from: `out_file`, `stdout`, or a JSON path. Each JSON-output CLI has its own envelope key, so the path is per-adapter — claude `.result`, gemini `.response`. |
| `smoke_test` | A cheap argv that proves installed + authed + network-reachable (exit 0 and output contains the token ⇒ reachability pass). Preflight pairs it with a **negative read-only probe** that targets a sentinel inside the artifact worktree and must be blocked at enforcement level (a model that merely *declines* is inconclusive, not a pass; a native subagent is exempt only when dispatched with an enforced read-only tool set). Both must pass. |
| `native_subagent` | Optional host-native dispatch when this adapter is also the host (e.g. `"claude_task"`). The host **must** dispatch it with an enforced read-only tool set (allowlist of read/search tools, not the default all-tools subagent); if it can't, use the CLI instead. Else null ⇒ use the CLI. |

**Uniformity rule (extensibility backbone):** *every* adapter prompt-enforces
the findings-JSON contract (below). A CLI's own `--output-schema` /
`--output-format json` is a **bonus** parse path, not required to add a new
adapter (built-ins like `codex` include it because the orchestrator always has
the findings schema to hand). Adding an LLM therefore needs only: a binary, a
non-interactive read-only invocation, and the ability to return its final text.

**Layered config (persistence across updates):** the orchestrator loads the
shipped `adapters.json`, then deep-merges (by `id`) an optional user override at
`${CLAUDE_CONFIG_DIR:-$HOME/.claude}/multi-llm-convergence/adapters.json` if
present. Operators add/override adapters there so plugin updates don't clobber
them. Built-ins: `claude`, `codex`, `gemini`, `grok` (`gemini` and `grok` carry a
"validate on first use" note — neither was locally smoke-tested at design time).
`gemini`'s read-only control is `--approval-mode plan` (a read-only mode enforced
by the built-in Tier-1 policy), with the headless caveat in Risks below.

### Findings contract (unchanged in shape)

The shared review contract and its JSON schema (severity
`critical|high|medium|low`, `location`, `claim`, `verified_against`,
`suggested_fix`, `clears_bar`) carry over verbatim from the original
`reviewer-dispatch.md`. It is prompt-enforced for all adapters.

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

### Step 0a — Detect the host (best-effort)

For each adapter, if any `detect_env` var is set, that adapter's `family` is the
host. First match wins. No match ⇒ host unknown ⇒ generic orchestrator (still
valid). Detection only unlocks `native_subagent` dispatch and lets the preflight
skip requiring the host's own CLI.

### Step 0b — Select the reviewer set

1. Probe every registry adapter: `command -v bin`.
2. Present the **available** adapters; operator picks **≥2 with distinct
   `family`**. Default suggestion: host family + one other (if host known), else
   the first two available.
3. Optional skill argument preselects (e.g. `… codex,gemini`) to skip the prompt.

### Step 0c — Preflight hard stop (both/all ends)

Smoke-test each **selected** reviewer along the **dispatch path it will actually
use** — its CLI for a CLI reviewer, its `native_subagent` for a host-native one —
not "its CLI" unconditionally (a host-family reviewer dispatched as a native
subagent need not have its CLI installed). Each smoke test has **two parts**:

1. **Reachability:** a trivial prompt returns the OK token ⇒ installed,
   authenticated, and able to reach its API.
2. **Read-only:** the reviewer must be *unable* to write — proven, not promised. A
   native subagent is read-only only if the host **dispatches it with an enforced
   read-only tool set** — an allowlist of read/search tools (or a read-only agent
   type), *not* the default subagent, which may inherit `Edit`/`Write`/`Bash`. (The
   original Claude `general-purpose` subagent carries all tools, so "by
   construction" is the *enforced dispatch boundary*, never an assertion.) If the
   host can't enforce a read-only native dispatch, route that reviewer through its
   CLI and run the probe below instead. A CLI reviewer's read-only-ness rests on a
   *flag*, so it gets a **negative probe**: the orchestrator drops a sentinel file
   **inside the artifact's own worktree** (the exact surface the reviewer reads —
   *not* an arbitrary temp path, because read-only policies are path-specific, e.g.
   gemini plan mode *allows* writes under `.gemini/tmp/.../plans/*.md` while denying
   others), tells the reviewer to modify that sentinel, and grades three ways:
   - **pass** — the write was *attempted* and *blocked at enforcement level* (a
     sandbox/policy denial) **and** the sentinel's checksum is verified unchanged;
   - **inconclusive** — the model *declined without attempting* ("I can't write in
     plan mode"). This does **not** count as read-only verified: a polite decline at
     preflight says nothing about what a prompt injection in the reviewed artifact
     could later induce. Treat as fail/abstain;
   - **fail** — the sentinel was modified.

   Nothing short of an observed enforcement-level denial *on the artifact surface*
   certifies the adapter — `--help` only shows a flag *exists*, not that
   `Bash`/shell/edit are blocked, and a soft-read-only adapter (e.g. gemini, whose
   headless policy auto-allows `exit_plan_mode` → YOLO) can otherwise escape to
   write.

**Hard stop if fewer than 2 selected reviewers pass both parts** (reachability
*and* an enforcement-level read-only pass) — print exactly what failed and how to
fix it (install/authenticate the CLI; for a sandboxed host, re-run with network
enabled; for a reviewer that **wrote** or only **declined** the probe, give it a
hard read-only sandbox or drop it — a decline is not certification). No silent
single-model degrade; the operator may explicitly opt into a single-model
iterate-to-clean run, clearly labeled **NOT cross-model consensus**.

**Network gotcha (generalized):** when the host is a sandboxed Codex (or any
network-disabled host), every reviewer CLI it spawns inherits the disabled
network and fails the reachability probe. The smoke test catches this
deterministically and the hard-stop message tells the operator to run the host
with network access.

### Step 1 — Ground truth

Unchanged from the original: identify external libraries/APIs the artifact makes
claims about, `git clone --depth 1 <repo> tmp/<name>` (gitignored), record exact
versions, and hand the local paths to every reviewer. Reviewers run read-only and
may be network-restricted, so local clones turn "fetch and stall" into "read a
file." Purely-internal artifacts skip cloning but still name in-repo
source-of-truth paths.

### Step 2 — Baseline (round 0)

Unchanged: ensure the artifact exists and is committed; `git init` if needed;
draft an initial version and commit it as round 0 if the artifact doesn't exist.

### Step 3 — Dispatch a reviewer (round-robin)

For the next reviewer in rotation (never the model whose findings were just
applied):

- Build the command from the adapter's `invoke` argv, substituting `{prompt}`
  (the shared contract), `{schema_file}`, `{out_file}`. Append
  `stream_invoke_extra` for the streamed variant.
- Cross-family ⇒ run the CLI in the background. Same-family-as-host with a
  `native_subagent` ⇒ dispatch that instead (e.g. Claude Task subagent).
- **Run reviewers one at a time.** Convergence is sequential by construction —
  model B must review what model A's round produced.
- Read the structured findings via `result` (`out_file` / `stdout` / JSON path).

#### Step 3a — Liveness watchdog (unified, REQUIRED every dispatch)

One supervision model for any backgrounded CLI:

1. Dispatch in the background; capture the process handle / log path. Stream the
   `stream-json`/`--json` variant to a log file.
2. Self-pace a poll with `ScheduleWakeup` (~270 s, under the 5-min cache window).
   Never end the turn with a reviewer in flight and no wake armed.
3. Each tick: process still alive + log advanced ⇒ healthy. Terminal exit ⇒ read
   result (non-zero ⇒ treat as stall/fail).
4. Stall verdict: alive but log unchanged across 2 consecutive ticks ⇒ hung.
5. Recover by fixing the cause (confirm Step-1 clones, re-dispatch read-local /
   no-network); escalate to the operator if it stalls again.
6. Long fallback `ScheduleWakeup` (~1200 s) as a backstop.
7. **Fallback where no self-wake exists:** foreground under a portable hard cap
   (`perl -e 'alarm shift; exec @ARGV' <secs> …`, since GNU `timeout` is absent
   on macOS/BSD) so a hang surfaces as a non-zero exit, not an infinite wait.

The host-native subagent path keeps its existing task-polling
(`TaskList`/`TaskGet`/`TaskOutput`) for liveness.

### Step 4 — Apply findings and commit

Unchanged: apply every finding **at or above the bar** under the
`coding-guidelines.md` rules (think first, minimal, surgical, verifiable). Verify
any external-library claim against the Step-1 clone before acting; record
source-contradicted claims as **rejected** and don't apply them. Commit each
round: `converge: round <N> — apply <model> review`.

### Step 5 — Convergence, stall, oscillation (N models)

- **Converged:** a full cycle in which **every selected model returns
  `clears_bar: true` with no above-bar findings, and no edit was applied anywhere
  in the cycle.** Equivalently, N consecutive clean reviewer passes (one per
  model) with no intervening edit — so every model independently blessed the
  *same* final state. Any above-bar finding ⇒ apply it, commit, reset the clean
  streak; the rotation must rebuild N clean passes. (With 2 models this reduces
  exactly to the original "two consecutive clean passes, one per model.")
- **Stalled (cap):** hit max cycles without a full clean cycle ⇒ stop, emit the
  stall report naming open above-bar findings. **Default cap: 3 full cycles**
  (≈3 passes per model, matching the original's ~3-per-model budget); warn on
  cost when ≥3 models are selected.
- **Oscillation:** key findings by `location`; if the same `location`'s change
  direction flips twice across rounds, stop and surface the cross-model
  disagreement to the operator with each model's argument. Don't average it away.

### Step 6 — Final output

- The converged artifact (committed round by round).
- A convergence log: one row per round — round number, which model reviewed,
  findings by severity, what was applied, what was rejected and why, that round's
  verdict.
- An explicit verdict: `CONVERGED` (naming **all** selected models and the final
  state), `STALLED (cap)`, `STALLED (oscillation)`, or
  `STALLED (reviewer unavailable)`.

## Packaging

```
plugins/multi-llm-convergence-beta/
├── README.md                                  # beta notice + promotion path
├── .claude-plugin/plugin.json                 # name, version 0.1.0, license MIT
└── skills/multi-llm-convergence-beta/
    ├── SKILL.md                               # <500 lines; the loop above
    ├── assets/
    │   └── adapters.json                      # canonical registry (claude, codex, gemini, grok)
    └── references/
        ├── reviewer-dispatch.md               # contract + per-adapter dispatch + watchdog
        ├── adding-an-adapter.md               # field schema + worked example + override path
        └── coding-guidelines.md               # copied verbatim from the original
```

- **Skill name / triggers:** `multi-llm-convergence-beta` (valid: single hyphens,
  ≤64 chars). Description leads with "**(beta/experimental)** direct-CLI,
  configurable N-model variant" plus a distinct trigger so that, with both
  plugins installed, the operator invokes the beta deliberately rather than the
  two firing ambiguously.
- **Registration:** add to `.claude-plugin/marketplace.json`, the root
  `README.md` plugins table (inside `<!-- PLUGINS:START/END -->`), and a dated
  `CHANGELOG.md` entry plus version-table row (`multi-llm-convergence-beta | 0.1.0`).
- **Validation:** `./validate-skills.sh plugins/*/skills/` must pass; description
  is a single line ≤1024 chars; `SKILL.md` < 500 lines.

### Promotion path

When the beta is accepted: copy `SKILL.md`, `assets/`, and `references/` into the
original `multi-llm-convergence` skill (renamed to the original skill name),
bump the original plugin's version, update its `marketplace.json` / README /
CHANGELOG, then delete `plugins/multi-llm-convergence-beta/` and its
registrations. The README beta notice documents this so the intent is explicit.

## Risks & open items

- **Gemini read-only is soft in headless mode.** `--approval-mode plan` is a
  current, documented read-only mode whose built-in Tier-1 policy blocks writes to
  anything but the plans directory — but in non-interactive runs that same policy
  **auto-allows `exit_plan_mode`** (verified in `plan.toml`: `decision = "allow"`,
  `interactive = false`), and exiting plan mode switches to YOLO (auto-approve
  all). So plan mode is **not** a hard boundary headless: a model mistake or a
  prompt injection in the reviewed artifact could `exit_plan_mode` and gain
  write/shell. The "do not exit plan mode" contract is advisory, **not**
  enforcement. The Step 0c probe only certifies an adapter when a write is
  *attempted and blocked at enforcement level*; a gemini that merely declines is
  **inconclusive**, not certified. So for untrusted artifacts gemini must run under
  a hard OS/sandbox layer or be marked **unsafe-for-untrusted-artifacts** (operator
  opts in knowingly); for a trusted artifact the soft mode plus the contract is
  acceptable. Not locally smoke-tested (not installed); validated on first use.
  Same caution for any soft-read-only adapter.
- **Heterogeneous read-only enforcement:** codex has a hard `-s read-only`
  sandbox; claude and grok use `--permission-mode plan` (a bare disallowed-edit
  list leaves `Bash` write-capable, so plan mode is the real control); gemini uses
  `--approval-mode plan` (soft headless — see above). `--help` confirms each flag
  *exists* but not that it blocks writes, so each adapter recipe must specify the
  strongest read-only flag the CLI offers **and** the preflight negative probe
  (Step 0c) must confirm a write is actually refused before the reviewer counts.
- **Cost with ≥3 models** grows linearly per cycle; surfaced as a warning at
  selection, not a silent default.
- **Same underlying model behind two ids** (e.g. two Claude-based wrappers)
  would be fake diversity; mitigated by the distinct-`family` selection rule and
  documented in `adding-an-adapter.md`.
```
