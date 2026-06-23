# Adding an adapter

Adding a new LLM to the convergence loop is a **data edit**, not a skill rewrite. You need three
things from the CLI: a binary, a non-interactive **read-only** invocation, and a way to return its
final text. The shared findings contract is prompt-enforced for every adapter, so a CLI-native schema
flag is a bonus, not a requirement.

## Where adapters live

- **Shipped (canonical):** `assets/adapters.json` in this skill — built-ins `claude`, `codex`,
  `gemini`, `grok`.
- **User override (persists across plugin updates):**
  `${CLAUDE_CONFIG_DIR:-$HOME/.claude}/multi-llm-convergence/adapters.json`.

The orchestrator loads the shipped file, then **deep-merges the override by `id`**. To add an adapter
without touching the plugin, drop a record into the override file. To tweak a built-in (e.g. change a
flag), put a record with the same `id` in the override — its fields win.

## Field contract

| Field | Required | Meaning |
|-------|----------|---------|
| `id` | yes | Unique adapter id. Selecting two adapters with the same `family` is rejected (no fake consensus). |
| `family` | no (defaults to `id`) | Model family for the "genuinely different model" guarantee. Two ids that wrap the **same** underlying model must share a `family` so they can't both be selected. |
| `bin` | yes | Binary checked with `command -v <bin>`. |
| `detect_env` | no (default `[]`) | Env vars that, if set, mark this adapter as the host. Empty ⇒ never auto-detected as host. |
| `invoke` | yes | **argv array** (no shell string). Placeholders `{prompt}`, `{schema_file}`, `{out_file}` are substituted at dispatch. `{prompt}` is the multi-line review contract injected as one argv element (no shell-escaping). Omit `{schema_file}`/`{out_file}` if the CLI has no schema/output-file flag. |
| `stream_invoke_extra` | no | Extra argv appended for the streamed/liveness variant (e.g. `["--json"]`, `["--output-format", "stream-json"]`). For CLIs that *replace* the format token, the appended `--output-format` wins (last value). |
| `result` | yes | Where the final structured text is read from: `{ "from": "out_file" }`, `{ "from": "stdout" }`, or `{ "from": "stdout", "json_path": ".key" }`. Each JSON-output CLI has its own envelope key, so the path is per-adapter. |
| `smoke_test` | yes | A cheap argv that proves installed + authed + network-reachable (exit 0 and output contains the OK token ⇒ reachability pass). Preflight (Step 0c) pairs it with the negative read-only probe; both must pass. |
| `native_subagent` | no (default `null`) | Host-native dispatch when this adapter is also the host (e.g. `"claude_task"`). The host **must** dispatch it with an enforced read-only tool set; if it can't, use the CLI. `null` ⇒ always use the CLI. |
| `note` | no | Free-text caveat surfaced to the operator (e.g. "validate on first use", soft-read-only warnings). |

## Worked example — add a hypothetical `acme` CLI

Suppose `acme` runs headless via `acme run --readonly --format json "<prompt>"`, prints a JSON
envelope `{ "answer": "…" }` to stdout, and has no schema flag. Drop this into the override file:

```json
{
  "adapters": [
    {
      "id": "acme",
      "family": "acme",
      "bin": "acme",
      "detect_env": [],
      "invoke": ["run", "--readonly", "--format", "json", "{prompt}"],
      "stream_invoke_extra": ["--stream"],
      "result": { "from": "stdout", "json_path": ".answer" },
      "smoke_test": ["run", "--readonly", "--format", "json", "Reply with the single token: OK"],
      "native_subagent": null,
      "note": "validate on first use"
    }
  ]
}
```

It now appears in Step 0b selection (if `command -v acme` succeeds), gets the shared contract as
`{prompt}`, and must pass the Step 0c reachability + read-only probe before it can review.

## Read-only is the load-bearing requirement

The single thing every adapter must get right is **read-only enforcement**. `--help` proving a flag
exists is *not* proof the flag blocks writes — Step 0c verifies that with a negative probe that drops
a sentinel inside the artifact worktree and requires an enforcement-level denial (a model that merely
declines is **inconclusive**, not certified). Pick the **strongest** read-only flag the CLI offers:

- A **hard sandbox** (codex `-s read-only`) is best — writes are blocked by the OS/runtime.
- A **plan/permission mode** (claude/grok `--permission-mode plan`, gemini `--approval-mode plan`) is
  acceptable for trusted artifacts but may be **soft** (gemini headless auto-allows `exit_plan_mode`
  → YOLO). For untrusted artifacts, wrap a soft adapter in a hard OS sandbox or mark it
  **unsafe-for-untrusted-artifacts** and have the operator opt in knowingly.
- A bare disallowed-tools list is **not** sufficient on Claude-family CLIs — it can leave `Bash`
  write-capable. Use the plan/permission mode.

## Fake-diversity caution

Two adapter ids backed by the **same** underlying model (e.g. two Claude-based wrappers) would give
*correlated* blind spots — fake consensus. Give them the **same `family`** so the distinct-`family`
selection rule (Step 0b) prevents picking both as "two different models."
