# Changelog

All notable changes to the loop-skills marketplace are documented here. This project adheres to [Conventional Commits](https://www.conventionalcommits.org/) and dates use `YYYY-MM-DD`.

| Plugin | Version |
|--------|---------|
| spec-development | 1.0.0 |
| dev-team | 1.0.0 |
| pr-autopilot | 1.1.0 |
| triangulated-code-review | 1.3.0 |
| multi-llm-convergence | 0.3.0 |
| multi-llm-convergence-beta | 0.1.0 |

## 2026-06-30

### Changed — `pr-autopilot` (1.1.0)

Version bump capturing the changes shipped since 1.0.0:

- **CI checks to green** — each round now also drives failing CI checks back to green, not just review comments (`feat(pr-autopilot): also drive failing CI checks to green`).
- **Trust-boundary guidance** — documented the untrusted-input and command-execution trust boundary in the review mechanics so PR content is treated as data, not instructions.

## 2026-06-23

### Added — `multi-llm-convergence-beta` (new plugin, 0.1.0)

A host-agnostic, configurable, **N-model** variant of `multi-llm-convergence`, shipped **in parallel** with the stable plugin (which stays untouched until promotion). Implements the approved design at `docs/superpowers/specs/2026-06-22-multi-llm-convergence-beta-design.md`. Three changes over the stable loop:

- **Direct-CLI delegation** — reaches every model by shelling out to its own CLI (`codex exec`, `claude -p`, `gemini -p`, `grok -p`) instead of the Codex companion script. No plugin dependency.
- **Configurable adapter registry** — `assets/adapters.json`, an argv-array invocation catalog with built-ins for `claude`, `codex`, `gemini`, `grok`; adding a CLI is a data edit (with a layered user override at `${CLAUDE_CONFIG_DIR:-$HOME/.claude}/multi-llm-convergence/adapters.json`).
- **User-selected reviewer set** — the operator picks which models converge (≥2 distinct families); the host is a pure orchestrator and need not review.

Keeps the original guarantees (local grounding, liveness watchdog, per-round commits) and adds a hard preflight that proves each reviewer is reachable **and** read-only via a negative probe (a decline is inconclusive; only an attempted-and-blocked write certifies). `gemini` and `grok` are validate-on-first-use.

### Changed

- **`multi-llm-convergence`** version row corrected from `0.2.0` to `0.3.0` to match the plugin manifest (the v0.3.0 Codex-plugin preflight hard stop shipped in commit `e710796`; the version table had not been updated). The plugin itself is unchanged.

## 2026-06-15

### Added — initial `loop-skills` marketplace

Created the `loop-skills` marketplace: a set of autonomous, self-re-prompting agent loops, packaged as five independently-installable plugins.

The spec/PR-workflow plugins were relocated here from the `donnfelker-plugin-marketplace` repo and repackaged by code coupling and standalone-usage rather than as one bundle:

- **`spec-development`** (new plugin, 1.0.0) — `plan-to-tickets` + `implement-full-spec`, slimmed out of the former `durable-spec-development` plugin. `implement-full-spec` now depends on the standalone `dev-team` and `pr-autopilot` plugins and checks for them via a startup preflight.
- **`dev-team`** (new plugin, 1.0.0) — split out of `durable-spec-development` so it can be installed and used on its own.
- **`pr-autopilot`** (new plugin, 1.0.0) — split out of `durable-spec-development`; now **owns** the shared `pr-review-mechanics.md` reference.
- **`triangulated-code-review`** (1.3.0) — relocated unchanged.
- **`multi-llm-convergence`** (0.2.0) — relocated, and carries the v0.2.0 anti-over-engineering rules (see Changed below).

### Changed

- **`pr-autopilot`** gained a `--single-pass` mode (alias for `--max-rounds 1`): one autonomous address → commit → push → reply → resolve round, then stop, with no human approval gate.
- **`implement-full-spec`** Phase C now delegates each individual PR's comment mechanics to `/pr-autopilot --single-pass`, keeping only the stacked-PR cascade-rebase layer. Its cross-skill links to `dev-team` were converted from relative paths to skill-name references (they now cross a plugin boundary).
- **`multi-llm-convergence`** bumped to v0.2.0: added a set of general anti-over-engineering rules (Think Before Coding, Simplicity First, Surgical Changes, Goal-Driven Execution) in a new `references/coding-guidelines.md`. They bind both roles — the driver follows them when applying findings (Steps 4 & 6) and reviewers report violations as findings (`references/reviewer-dispatch.md`). Carried over from `donnfelker-plugin-marketplace` PR #18.

### Removed

- **`address-pr-comments`** was retired. Its single-pass, no-loop behavior is now `pr-autopilot --single-pass`.
