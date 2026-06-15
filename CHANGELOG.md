# Changelog

All notable changes to the loop-skills marketplace are documented here. This project adheres to [Conventional Commits](https://www.conventionalcommits.org/) and dates use `YYYY-MM-DD`.

| Plugin | Version |
|--------|---------|
| spec-development | 1.0.0 |
| dev-team | 1.0.0 |
| pr-autopilot | 1.0.0 |
| triangulated-code-review | 1.3.0 |
| multi-llm-convergence | 0.1.0 |

## 2026-06-15

### Added — initial `loop-skills` marketplace

Created the `loop-skills` marketplace: a set of autonomous, self-re-prompting agent loops, packaged as five independently-installable plugins.

The spec/PR-workflow plugins were relocated here from the `donnfelker-plugin-marketplace` repo and repackaged by code coupling and standalone-usage rather than as one bundle:

- **`spec-development`** (new plugin, 1.0.0) — `plan-to-tickets` + `implement-full-spec`, slimmed out of the former `durable-spec-development` plugin. `implement-full-spec` now depends on the standalone `dev-team` and `pr-autopilot` plugins and checks for them via a startup preflight.
- **`dev-team`** (new plugin, 1.0.0) — split out of `durable-spec-development` so it can be installed and used on its own.
- **`pr-autopilot`** (new plugin, 1.0.0) — split out of `durable-spec-development`; now **owns** the shared `pr-review-mechanics.md` reference.
- **`triangulated-code-review`** (1.3.0) — relocated unchanged.
- **`multi-llm-convergence`** (0.1.0) — relocated unchanged.

### Changed

- **`pr-autopilot`** gained a `--single-pass` mode (alias for `--max-rounds 1`): one autonomous address → commit → push → reply → resolve round, then stop, with no human approval gate.
- **`implement-full-spec`** Phase C now delegates each individual PR's comment mechanics to `/pr-autopilot --single-pass`, keeping only the stacked-PR cascade-rebase layer. Its cross-skill links to `dev-team` were converted from relative paths to skill-name references (they now cross a plugin boundary).

### Removed

- **`address-pr-comments`** was retired. Its single-pass, no-loop behavior is now `pr-autopilot --single-pass`.
