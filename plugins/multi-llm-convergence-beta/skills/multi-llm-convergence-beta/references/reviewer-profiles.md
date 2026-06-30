# Reviewer Profiles

The beta is host-agnostic, not arbitrary-command configurable. Use only these built-in profiles.
Every profile must run through the uniform protocol in `reviewer-dispatch.md`.

## Built-in profiles

| Family | Official CLI | Review-only control | Result handling |
|--------|--------------|---------------------|-----------------|
| `claude` | `claude` | `--permission-mode plan` | parse final JSON from stdout |
| `codex` | `codex` | `exec -s read-only` | parse final JSON from stdout/output file |
| `gemini` | `gemini` | `--approval-mode plan` | parse final JSON from stdout |
| `grok` | `grok` | `--permission-mode plan` | parse final JSON from stdout |

Select at least two distinct families. Selecting multiple wrappers around the same underlying family
is fake consensus and must be rejected.

## Fixed command shapes

These are the only built-in command shapes. Substitute the shared review contract for `<prompt>` as a
single prompt payload.

```bash
claude -p --output-format json --permission-mode plan "<prompt>"
codex exec -s read-only --skip-git-repo-check "<prompt>"
gemini --approval-mode plan --output-format json -p "<prompt>"
grok -p "<prompt>" --output-format json --permission-mode plan
```

Use streaming/json output variants only when needed for liveness, and only if the installed CLI
documents that flag. Do not add model-specific behavior that changes the reviewer contract.

## Preflight

Each selected profile must pass the same preflight before Step 1:

1. `command -v <cli>` succeeds.
2. A one-token smoke prompt returns the expected token.
3. The review-only control is accepted by the CLI.
4. Output can be captured and parsed.

If a profile cannot satisfy these checks, it is unavailable for this run.

## Extension rule

Do not add reviewer families through user config, project files, environment variables, or runtime
command snippets. To add another family, change the skill in the repository:

1. Add a static profile to this file.
2. Define its fixed command shape and review-only control.
3. Update `SKILL.md`, README, marketplace metadata, and CHANGELOG.
4. Re-run `./validate-skills.sh plugins/*/skills/`.

If a proposed reviewer cannot be constrained to review-only behavior, it is not eligible for the
built-in profile list.
