# AGENTS.md

Guidelines for AI agents working in this repository.

## Repository Overview

This repository contains **Agent Skills** following the [Agent Skills specification](https://agentskills.io/specification.md), packaged as a **Claude Code plugin marketplace** via `.claude-plugin/marketplace.json`.

- **Name**: loop-skills
- **GitHub**: [donnfelker/loop-skills](https://github.com/donnfelker/loop-skills)
- **License**: MIT

Every plugin here is a **bounded autonomous loop**: it drives an artifact to a terminal state, checks an explicit termination condition (quiet round, two clean passes, a cap), and commits per round for an audit trail.

## Repository Structure

```
plugins/
└── plugin-name/
    ├── .claude-plugin/
    │   └── plugin.json        # Plugin manifest (name, description, version, license)
    ├── references/            # Optional - plugin-wide docs cited via ${CLAUDE_PLUGIN_ROOT}/references/...
    └── skills/
        └── skill-name/
            ├── SKILL.md       # Required skill file (<500 lines)
            ├── references/    # Optional - skill-local docs loaded on demand
            ├── scripts/       # Optional - executable code
            └── assets/        # Optional - templates, data files
```

## Build / Lint / Test Commands

Skills are content-only (no build step). Validate with:

```bash
./validate-skills.sh plugins/*/skills/
```

Manually verify: YAML frontmatter is valid; `name` matches the directory exactly; `name` is 1-64 chars, lowercase alphanumeric and hyphens; `description` is 1-1024 characters.

## Required Frontmatter

```yaml
---
name: skill-name
description: What this skill does and when to use it. Include trigger phrases.
---
```

| Field         | Required | Constraints                                                      |
|---------------|----------|------------------------------------------------------------------|
| `name`        | Yes      | 1-64 chars, lowercase `a-z`, numbers, hyphens. Must match dir.   |
| `description` | Yes      | 1-1024 chars. Describe what it does and when to use it.          |
| `license`     | No       | License name (default: MIT)                                      |
| `metadata`    | No       | Key-value pairs (author, version, etc.)                          |

Write `description` as a **single line**. Do not use YAML block scalars (`>`, `|`, `>-`, `|-`).

## References: shared vs. skill-local

- **Plugin-wide references** that more than one skill in the plugin uses live at the plugin root `references/` and are cited as `${CLAUDE_PLUGIN_ROOT}/references/<file>` (e.g. `pr-autopilot`'s `pr-review-mechanics.md`).
- **Skill-local references** stay inside each skill's own `references/` and are cited by relative path within that skill.

## Cross-plugin dependencies

Skills are invoked by **name**, and skill names resolve globally once installed — so a skill in one plugin can invoke a skill in another. When it does:

- Reference the other skill by **name** (`` `dev-team` ``), not by a relative path (`../dev-team/SKILL.md` will not resolve across a plugin boundary).
- Claude Code does not auto-install plugin dependencies. A skill that orchestrates skills from other plugins MUST **preflight-check** that they're available and tell the operator which plugin(s) to `/plugin install` if not (see `implement-full-spec`).
- Document the dependency in the plugin's `plugin.json` description, its README row, and `marketplace.json`.

## Generalization rule (no source-specific details)

When authoring or editing any content under `plugins/`, never include specifics from the source material that produced the lesson — workspace/team/company names, real project or ticket IDs, URLs, UUIDs, API keys, or one-off exact counts. Use placeholder tokens (`<TEAM>-<N>`, `<workspace>`, `/Users/<user>/...`). Keep platform-default vocabulary, tool/parameter names, and exact error strings. The test: would a reader at a different company learn the same lesson, or would they have to mentally substitute every example?

## Writing Style

- Keep `SKILL.md` under 500 lines (move details to `references/`).
- H2 for main sections, H3 for subsections; short paragraphs; bullets and tables liberally.
- Direct, instructional, second person. Active voice. Bold for key terms. No excessive emojis.
- The `description` is critical for discovery: include what it does, when to use it (trigger phrases), and related skills for scope boundaries.

## Git Workflow

- **Branches**: `feature/<skill>`, `fix/<skill>-<desc>`, `docs/<desc>`.
- **Commits**: [Conventional Commits v1.0.0](https://www.conventionalcommits.org/) — `type(scope): description`.

### PR Checklist

- [ ] `name` matches directory exactly and follows naming rules (lowercase, hyphens, no `--`)
- [ ] `description` is a single line, 1-1024 chars, with trigger phrases
- [ ] `SKILL.md` is under 500 lines
- [ ] Cross-plugin references use skill names, not relative paths; any new dependency is preflight-checked and documented
- [ ] No sensitive data, credentials, or source-specific details
- [ ] Plugin `version` bumped in `plugin.json` (SemVer) whenever a skill or plugin is edited
- [ ] `README.md` Available Plugins table updated (`<!-- PLUGINS:START -->` / `<!-- PLUGINS:END -->`)
- [ ] `CHANGELOG.md` updated with a dated entry
