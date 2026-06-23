# Reviewer dispatch — contract, per-adapter recipes, watchdog, read-only probe

Every reviewer gets the **same review contract** so their outputs are mechanically comparable. The
only thing that differs is which CLI (or native subagent) runs it. Substitute every `<PLACEHOLDER>`
before dispatching — reviewers start with no context and will not resolve placeholders themselves.

Required substitutions:

- `<ARTIFACT_PATH>` → absolute path to the artifact under review (the file, or the repo + diff range
  for code).
- `<BAR>` → the convergence bar, e.g. "critical or high severity".
- `<GROUND_TRUTH>` → newline list of absolute local paths the reviewer must verify claims against
  (the `tmp/<name>` clones from Step 1, plus relevant in-repo paths). Include the version/tag/SHA.
- `<ROUND>` → the current round number, for the log.

## The shared review contract (the `{prompt}` injected into every adapter)

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

Only the severity vocabulary (`critical | high | medium | low`) deliberately matches the Codex
plugin's review schema; the rest of the JSON is this loop's own shape. **Every** adapter
prompt-enforces this contract — a CLI's own `--output-schema`/`--output-format json` is a *bonus*
parse path, not a requirement. So even if an adapter's `result` envelope key is wrong, the driver can
extract the findings JSON from the reviewer's final text.

## Building a dispatch from the registry

For the selected reviewer, take its registry record and build the command from `invoke`, substituting:

- `{prompt}` → the fully-substituted shared contract above, injected as a **single argv element** (no
  shell-escaping — it's an argv array, never a shell string).
- `{schema_file}` → a temp file the orchestrator writes the findings JSON Schema to (only for adapters
  whose CLI has a schema flag, e.g. codex `--output-schema`).
- `{out_file}` → a temp output path the orchestrator picks (only for adapters that declare it, e.g.
  codex `-o`).

An adapter whose CLI has no schema flag simply omits those two placeholders. Append
`stream_invoke_extra` for the streamed/liveness variant. Read the result via `result`:

- `{ "from": "out_file" }` → read the temp `{out_file}`.
- `{ "from": "stdout" }` → scan stdout text for the findings JSON.
- `{ "from": "stdout", "json_path": ".result" }` → parse stdout as JSON and read that key; if parsing
  fails, fall back to scanning the text (the contract is prompt-enforced).

## Per-adapter recipes (built-ins)

The argv below come straight from `assets/adapters.json` (verified against live `--help` on the dev
machine and/or the cloned docs). `gemini` and `grok` carry **validate-on-first-use** notes in the
registry — neither was locally smoke-tested at design time.

### codex (hard read-only sandbox)

```
# read-only by sandbox (-s read-only); writes the final message to {out_file}; schema to {schema_file}
codex exec -s read-only --skip-git-repo-check --output-schema <schema_file> -o <out_file> "<prompt>"
# streamed/liveness variant appends --json (jsonl to stdout)
codex exec -s read-only --skip-git-repo-check --output-schema <schema_file> -o <out_file> --json "<prompt>"
```

Result from `<out_file>`. Sandbox modes are exactly `read-only | workspace-write | danger-full-access`.

### claude (`--permission-mode plan`)

```
claude -p --output-format json --permission-mode plan "<prompt>"
# streamed/liveness variant (last --output-format wins): stream-json + --verbose
claude -p --output-format stream-json --verbose --permission-mode plan "<prompt>"
```

`-p/--print` is non-interactive; the prompt is the positional `[prompt]`. Result: parse stdout JSON,
read `.result`. `plan` is the read-only control — a bare `--disallowedTools "Edit Write NotebookEdit"`
still leaves `Bash` write-capable, so plan mode is the real boundary. (claude also exposes
`--json-schema <schema>`, which takes the schema *inline as a string*, not a file path; the built-in
recipe relies on prompt-enforcement instead, so `{schema_file}` is intentionally omitted.)

### gemini (`--approval-mode plan` — SOFT headless)

```
gemini --approval-mode plan --output-format json -p "<prompt>"
# streamed/liveness variant: stream-json
gemini --approval-mode plan --output-format stream-json -p "<prompt>"
```

Result: parse stdout JSON, read `.response`. **Soft read-only headless:** `plan.toml` auto-allows
`exit_plan_mode` (`decision=allow`, `interactive=false`) → YOLO, so plan mode is not a hard boundary.
For untrusted artifacts, sandbox it hard or mark it unsafe-for-untrusted-artifacts (see SKILL.md
Risks). The Step 0c probe certifies it only on an *attempted-and-blocked* write.

### grok (`--permission-mode plan`)

```
grok -p "<prompt>" --output-format json --permission-mode plan
# streamed/liveness variant: streaming-json (note the token — not stream-json)
grok -p "<prompt>" --output-format streaming-json --permission-mode plan
```

`-p/--single <PROMPT>` takes the prompt as the flag value. `--permission-mode` enum is
`default, acceptEdits, auto, dontAsk, bypassPermissions, plan` — there is no `read-only` value, so
`plan` is the control. Result: parse stdout JSON, read `.result` (inferred from grok's
Claude-Code-compatible lineage — confirm on first use).

## The liveness watchdog (unified, REQUIRED on every dispatch)

`ScheduleWakeup`, `Agent(...)`, and `TaskList`/`TaskGet`/`TaskOutput` are illustrative capability
names — map them to whatever the host exposes for scheduling a self-wake, dispatching a background
job/subagent, and polling it.

> **The rule that prevents babysitting:** never end your turn while a background reviewer is in flight
> unless a `ScheduleWakeup` is already armed. The runtime re-invokes you at that time whether or not
> the reviewer finishes, so a hung, silent reviewer cannot strand the loop.

The supervision loop, either dispatch path:

1. **Dispatch in the background**, capturing the process handle / log path. Stream the
   `stream-json`/`--json` variant to a log file so liveness has something to advance.
2. **Self-pace a poll** with `ScheduleWakeup` (~270 s, under the 5-min prompt-cache window). Drive it
   with `ScheduleWakeup` directly — do **not** invoke the user-facing `/loop` command.
3. **Each tick, read state + output growth.** Process alive + log advanced ⇒ healthy, let it run.
   Terminal exit ⇒ read the result (non-zero exit ⇒ treat as stall/fail).
4. **Stall verdict:** alive but log unchanged across **2 consecutive ticks** with no result ⇒ hung.
   Don't keep waiting; don't blindly re-dispatch the same way (it stalls identically).
5. **Recover by fixing the cause:** confirm the Step-1 `tmp/` clones exist and are complete, then
   re-dispatch with explicit "read these local paths, make no network calls." Escalate if it stalls
   again after grounding.
6. **Long fallback `ScheduleWakeup`** (~1200 s) as a backstop, so a missed poll still wakes the loop.
7. **Fallback where no self-wake exists:** run foreground under a portable hard cap so a hang can't be
   infinite. GNU `timeout` is **not** on macOS/BSD (exit 127 / `command not found`), so use:

   ```
   # portable hard cap (macOS/BSD + Linux): exits ~142 on timeout
   perl -e 'alarm shift; exec @ARGV' 600 <the full reviewer command>
   # or, if GNU coreutils is installed: gtimeout 600 <the full reviewer command>
   ```

   A timeout/non-zero exit means it blew the cap → treat as a stall. You lose "thinking vs stuck"
   discrimination but keep the no-hang guarantee.

**Host-native subagent path.** When a host-family reviewer is dispatched as a `native_subagent`
instead of via its CLI, keep the host's existing task-polling (`TaskList`/`TaskGet`/`TaskOutput`) for
state and output growth — same stall logic, same wake discipline.

> The signal that matters is *state plus output advancement*, not elapsed time. A reviewer
> legitimately thinking for four minutes (alive, log trickling) is fine; a live process whose log
> hasn't moved across two checks is stuck.

## The Step 0c read-only negative probe (mechanics)

A CLI reviewer's read-only-ness rests on a flag, so prove it before the reviewer counts:

1. **Drop a sentinel inside the artifact's own worktree** — the exact surface the reviewer reads —
   *not* an arbitrary temp path. Read-only policies are path-specific (gemini plan mode *allows*
   writes under `.gemini/tmp/.../plans/*.md` while denying others), so a temp-dir probe can pass while
   the artifact surface is still writable. Record the sentinel's checksum.
2. **Dispatch a one-shot prompt** through the adapter's real `invoke` path telling the reviewer to
   modify that sentinel file.
3. **Grade three ways:**
   - **pass** — the write was *attempted* and *blocked at enforcement level* (a sandbox/policy denial)
     **and** the sentinel's checksum is unchanged.
   - **inconclusive** — the model *declined without attempting* ("I can't write in plan mode"). Treat
     as fail/abstain: a polite decline says nothing about what a prompt injection in the reviewed
     artifact could later induce.
   - **fail** — the sentinel was modified.
4. **Clean up** the sentinel afterward.

For a **native subagent**, there is no flag to probe — it is read-only only when the host dispatches
it with an **enforced read-only tool set** (an allowlist of read/search tools, or a read-only agent
type), *not* the default all-tools subagent. If the host can't enforce that, route the reviewer
through its CLI and run the probe above instead.

## Why every reviewer gets the identical contract

Comparable, structured output is what lets the driver decide convergence mechanically: a round is
clean iff `clears_bar == true` and `findings` has nothing at or above the bar. If reviewers used
different formats or severity language, you'd be back to interpreting prose — and prose is where a
fake "looks good" slips through.
