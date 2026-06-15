# Reviewer dispatch templates

Both reviewers get the **same review contract** so their outputs are mechanically comparable. The only difference is which model runs it. Substitute every `<PLACEHOLDER>` before dispatching — subagents start with no context and will not resolve placeholders themselves.

Required substitutions for both templates:

- `<ARTIFACT_PATH>` → absolute path to the artifact under review (the file, or the repo + diff range for code).
- `<BAR>` → the convergence bar, e.g. "critical or high severity".
- `<GROUND_TRUTH>` → newline list of absolute local paths the reviewer must verify claims against (the `tmp/<name>` clones from Step 1, plus relevant in-repo paths). Include the version/tag/SHA you cloned.
- `<ROUND>` → the current round number, for the log.

## The shared review contract (paste into both prompts)

```
You are reviewing an artifact for a multi-LLM convergence loop. This is round <ROUND>. A DIFFERENT model than you last edited this artifact; your job is to find what it got wrong or left unsafe — independently. Review only; do NOT edit any files.

Artifact under review: <ARTIFACT_PATH>
Convergence bar (what must be clean to stop): <BAR>

Source of truth — verify EVERY claim about external libraries/APIs against these LOCAL paths. Do NOT use the network; if you find yourself wanting to fetch something, read the local clone instead. These were cloned at known versions for exactly this purpose:
<GROUND_TRUTH>

Rules:
- Verify, don't assume. For any claim about a third-party library/framework/API/CLI, open the local source above and confirm the actual signature/behavior/version. If the source contradicts a concern you had, drop the concern.
- Internal/first-party code: read it in the repo; don't speculate.
- Only report real issues. Do not pad. A short, correct list beats a long, hedged one.
- Hold the artifact to these general rules and report violations as findings: (1) Think before coding — unstated assumptions, unsurfaced tradeoffs, or ambiguity resolved silently. (2) Simplicity first — speculative features, needless abstraction/configurability, error handling for impossible cases, code that could be far shorter. (3) Surgical changes — edits that touch more than the request required, drive-by refactors, or style churn in untouched code. (4) Goal-driven execution — changes with no verifiable success criterion (e.g. behavior changed with no test that pins it down).
- Severity: critical (the artifact is wrong/will fail as written), high (serious correctness or design flaw), medium (quality/clarity), low (nit).

Return ONLY this JSON object — no preamble, no trailing commentary:
{
  "round": <ROUND>,
  "findings": [
    {
      "severity": "critical|high|medium|low",
      "location": "file:line or section name",
      "claim": "what is wrong and why, with the reasoning and the trigger scenario",
      "verified_against": "which local source path you checked, or 'internal/n-a'",
      "suggested_fix": "concrete change"
    }
  ],
  "clears_bar": true|false,
  "notes": "optional cross-cutting observation"
}

`clears_bar` is true only if there are zero findings at or above the bar (<BAR>).
```

Only the severity vocabulary (`critical | high | medium | low`) deliberately matches the Codex plugin's review schema; the rest of the JSON contract above is this convergence loop's own shape. The `task` path returns free-form final-output text rather than schema-validated review JSON, so the driver extracts the structured verdict from that output rather than receiving it directly from a schema-enforced layer.

## Reviewer A — Codex (non-Claude model), read-only via the `task` path

Use the free-form `codex-companion task` command. Do **not** use `review`/`adversarial-review` (they diff git and assume a code change — wrong for a plan/PRD/doc), and do **not** route through the `codex:codex-rescue` agent (it defaults to a write-capable run, only forwards to `task`, and exposes no `status`/`result` job-control for the watchdog).

Resolve the companion script path, trying in order, first hit wins:

1. `$CLAUDE_PLUGIN_ROOT/scripts/codex-companion.mjs` (if `CLAUDE_PLUGIN_ROOT` is set)
2. `find "$HOME/.claude/plugins" -path '*/codex/scripts/codex-companion.mjs' -print -quit 2>/dev/null`

If neither resolves, Codex is not installed → tell the user to install the Codex plugin from https://github.com/openai/codex-plugin-cc (then run `/codex:setup` to authenticate); see "Only one reviewer available" in SKILL.md edge cases.

**Primary (background + native job-control), so the watchdog can poll:**

```
# dispatch read-only (omit --write) in the background; capture the printed job id
node "<companion>" task --background "<the shared review contract above>"

# each watchdog tick (~270s): read authoritative job state, returns immediately
node "<companion>" status <job-id> --json

# when state is terminal (completed/failed/cancelled): fetch the structured output
node "<companion>" result <job-id> --json
```

The review JSON lives inside the task's final-output text within that result payload — extract it from the output text rather than expecting a top-level schema-validated object.

Omitting `--write` keeps Codex read-only — a reviewer must not edit the artifact. `status --json` is the liveness signal: read `job.status` (`queued`/`running`/`completed`/`failed`/`cancelled` — `done` is a *phase*, not a status), plus whether output advanced. It is independent of the companion's internal `--wait` timeout, so your tick cadence is yours.

**Fallback (no `ScheduleWakeup` in this host):** run foreground under a hard cap so a hang can't be infinite. GNU `timeout` is **not** present on macOS/BSD (you'll get `command not found` / exit 127), so use a portable cap —

```
# portable hard cap (macOS/BSD + Linux): exits ~142 on timeout
perl -e 'alarm shift; exec @ARGV' 600 node "<companion>" task "<the shared review contract above>"
# or, if GNU coreutils is installed: gtimeout 600 node "<companion>" task "<…>"
```

A timeout/non-zero exit means it blew the cap → treat as a stall (ground-truth check, then one re-dispatch; escalate if it stalls again). You lose "thinking vs stuck" discrimination but keep the no-hang guarantee.

## Reviewer B — independent Claude review subagent

Dispatch `Agent(subagent_type: "general-purpose", run_in_background: true, description: "Independent convergence review round <ROUND>", prompt: <the shared review contract above>)`. It runs on the Claude family — a different mind from Codex, and a fresh context from the driver that just applied the last round's edits. Capture the returned agent id; the watchdog polls it via `TaskList`/`TaskGet`/`TaskOutput` for state and output growth.

## Why both get the identical contract

Comparable, structured output is what lets the driver decide convergence mechanically: a round is clean iff `clears_bar == true` and the `findings` list has nothing at or above the bar. If the two reviewers used different formats or severity language, you'd be back to interpreting prose — and prose is where a fake "looks good" slips through.
