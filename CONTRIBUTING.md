# Contributing to loop-skills

Thanks for contributing! This repo is a Claude Code plugin marketplace of autonomous agent loops. The full authoring conventions live in [AGENTS.md](AGENTS.md); this is the short version.

## Adding or changing a skill

1. Skills live at `plugins/<plugin>/skills/<skill>/SKILL.md`. The `name` in the frontmatter MUST match the skill's directory name exactly.
2. Keep `SKILL.md` under 500 lines — move detail into `references/`.
3. Write the `description` as a single line (no YAML block scalars), 1-1024 chars, with trigger phrases and scope-boundary pointers to related skills.
4. Put plugin-wide references at the plugin root `references/` (cited via `${CLAUDE_PLUGIN_ROOT}/references/...`); keep skill-specific ones in the skill's own `references/`.
5. If your skill invokes a skill from another plugin, reference it by name (not a relative path), preflight-check that it's installed, and document the dependency.
6. Don't include source-specific details (real names, ticket IDs, URLs, keys) — generalize to placeholders. See the generalization rule in AGENTS.md.

## Before you open a PR

Run the validator:

```bash
./validate-skills.sh plugins/*/skills/
```

Then confirm the PR checklist in [AGENTS.md](AGENTS.md#pr-checklist), including updating the `README.md` Available Plugins table and adding a dated `CHANGELOG.md` entry.

## Adding a new plugin

1. Create `plugins/<plugin>/.claude-plugin/plugin.json` (`name` matching the directory, a single-line `description`, `version`, `license`).
2. Add the plugin to `.claude-plugin/marketplace.json`.
3. Add a row to the README's Available Plugins table and a CHANGELOG entry.

## Commits

Use [Conventional Commits](https://www.conventionalcommits.org/): `type(scope): description`.
