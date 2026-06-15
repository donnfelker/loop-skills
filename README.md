# loop-skills

Autonomous, self-re-prompting agent loops for Claude Code. Each plugin here is a **bounded autonomous loop** that drives an artifact to a terminal state — it iterates, checks a termination condition (a quiet round, two clean passes, a cap), keeps itself alive, and commits per round for an audit trail.

Built by [Donn Felker](https://donnfelker.com).

**Contributions welcome!** Found a way to improve a plugin or have a new one to add? [Open a PR](#contributing).

Run into a problem or have a question? [Open an issue](https://github.com/donnfelker/loop-skills/issues).

## What are Plugins?

Plugins are packages of skills, commands, agents, and hooks that extend Claude Code with specialized capabilities. Each plugin in this marketplace is installed independently — pick only what you need. A few of them compose: `spec-development` orchestrates the standalone `dev-team` and `pr-autopilot` loops (see the dependency note below).

## Available Plugins

<!-- PLUGINS:START -->
| Plugin | Description |
|--------|-------------|
| [spec-development](plugins/spec-development/) | End-to-end spec workflow. Bundles `plan-to-tickets` (import a structured plan into ClickUp, Linear, Jira, Asana, Notion, GitHub Projects, or a markdown fallback as a ticket hierarchy) and `implement-full-spec` (turn a parent ticket with N actionable subtasks into N stacked PRs and drive each to merge-ready through multi-round bot and human review). **Depends on the `dev-team` and `pr-autopilot` plugins** — install all three together. |
| [dev-team](plugins/dev-team/) | Drive a single unit of work — one spec, ticket, finding, or change request — from spec to a committed, reviewed result with a dev team of coordinating agents: Dev → QA → Reviewer in a bounded cycle. Runs as a real agent team where the harness supports one, else coordinated subagents. Usable standalone, or as the per-subtask engine behind `implement-full-spec` |
| [pr-autopilot](plugins/pr-autopilot/) | Autonomously watch a single PR and loop through review rounds — fetch every unaddressed review/comment, fix the actionable ones, commit, push, reply/resolve threads, re-request review — until a quiet round or a safety cap. Includes a `--single-pass` mode (one autonomous round, no loop) for a one-shot comment sweep. Owns the shared `pr-review-mechanics.md` reference |
| [triangulated-code-review](plugins/triangulated-code-review/) | Triangulated multi-reviewer code review orchestrator. Borrows from research methodology — checks each finding against multiple independent reviewers (comprehensive, security, codex, codex adversarial) to reduce blind spots, then runs a QA analyst pass that substantiates every finding (verifying any third-party library claims via the context7 MCP) and demotes unsubstantiated ones into a dedicated "Invalidated Findings" section of the prioritized, timestamped report |
| [multi-llm-convergence](plugins/multi-llm-convergence/) | Drives any artifact (plan, PRD, design doc, spec, or implementation diff) to convergence by alternating two genuinely different LLM reviewers — a Codex reviewer and an independent Claude review subagent — applying each round's findings and looping until both independently agree it clears the bar. Grounds offline reviewers in local source-of-truth clones, runs a liveness watchdog so a silent reviewer can't hang the loop, and commits per round for an auditable trail |
<!-- PLUGINS:END -->

## Installation

### Option 1: CLI Install (Recommended)

Use [npx skills](https://github.com/vercel-labs/skills) to install skills directly:

```bash
# Install everything
npx skills add donnfelker/loop-skills

# Install specific skills
npx skills add donnfelker/loop-skills --skill dev-team pr-autopilot

# List available skills
npx skills add donnfelker/loop-skills --list
```

This installs to your `.agents/skills/` directory (and symlinks into `.claude/skills/` for Claude Code compatibility).

### Option 2: Claude Code Plugin

Install via Claude Code's built-in plugin system:

```bash
# Add the marketplace
/plugin marketplace add donnfelker/loop-skills

# Install the plugins you want
/plugin install dev-team@loop-skills
/plugin install pr-autopilot@loop-skills
/plugin install spec-development@loop-skills
/plugin install triangulated-code-review@loop-skills
/plugin install multi-llm-convergence@loop-skills
```

### Option 3: SkillKit (Multi-Agent)

Use [SkillKit](https://github.com/rohitg00/skillkit) to install skills across multiple AI agents (Claude Code, Cursor, Copilot, etc.):

```bash
# Install everything
npx skillkit install donnfelker/loop-skills

# Install specific skills
npx skillkit install donnfelker/loop-skills --skill dev-team pr-autopilot

# List available skills
npx skillkit install donnfelker/loop-skills --list
```

Each `/skill` still works standalone once installed — `/dev-team`, `/pr-autopilot`, `/implement-full-spec`, `/plan-to-tickets`, `/triangulated-code-review`, `/multi-llm-convergence`.

### Plugin dependencies

`spec-development`'s `implement-full-spec` orchestrates two skills that ship as separate plugins:

- It runs **`dev-team`** once per subtask (the Dev → QA → Reviewer → commit loop).
- It delegates each PR's review-response mechanics to **`pr-autopilot`** (`--single-pass`).

Claude Code plugins don't auto-install dependencies, so **install `dev-team` and `pr-autopilot` alongside `spec-development`**. `implement-full-spec` checks for both at the start of a run and tells you what to install if either is missing.

## License

MIT — see [LICENSE](LICENSE).

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) and [AGENTS.md](AGENTS.md) for skill-authoring conventions and the PR checklist. In short: `name` must match the skill's directory, `description` is a single line under 1024 chars with trigger phrases, `SKILL.md` stays under 500 lines, and every change updates `README.md` (the table above) and `CHANGELOG.md`.
