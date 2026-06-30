# multi-llm-convergence-beta

> **Beta / experimental.** This plugin ships in parallel with the stable
> [`multi-llm-convergence`](../multi-llm-convergence/) (v0.3.0), which stays untouched. When the beta
> proves out it is promoted into the original namespace and this plugin is removed.

A **host-agnostic, N-model** convergence loop. It drives any artifact - a plan, PRD, design doc,
spec, or implementation diff - to genuine cross-model consensus, then stops.

The stable skill is the source of truth for the work: preflight first, ground reviewers in
source-of-truth, commit each round, review sequentially, and stop only on real cross-model consensus.
The beta changes the dispatch layer so the same loop can be launched from Claude, Codex, Gemini, or
another capable host.

## What changed from stable

1. **Host-agnostic launch.** The host is an orchestrator. It can be Claude, Codex, Gemini, or another
   environment that can run the selected official reviewer CLIs.
2. **N-model reviewer set.** The operator may select any two or more built-in reviewer families with
   distinct family names. Built-ins are `claude`, `codex`, `gemini`, and `grok`.
3. **Uniform reviewer protocol.** Every selected reviewer uses the same lifecycle: static built-in
   profile, smoke-test, review-only mode, identical review contract, captured output, liveness
   supervision, structured JSON parsing, then the same apply-and-commit step.

This plugin is not a user-configurable command runner. It does not load project/user adapter files
and it does not accept arbitrary reviewer commands. Adding another model family requires a repository
change to the built-in profiles and docs.

## When to use this vs. the stable skill

- **Use `multi-llm-convergence` (stable)** for the proven Claude Code loop.
- **Use `multi-llm-convergence-beta`** when you want to launch the same convergence loop from a
  non-Claude host or converge with more than two built-in model families.

With both plugins installed, invoke the beta deliberately. Its description carries distinct triggers
such as `beta convergence`, `N-model convergence`, `run convergence from Codex/Gemini/Claude`, and
`converge with multiple models`.

## Security posture

- Built-in reviewer profiles only; no user-provided command adapters.
- Local-only source grounding; no autonomous clone/download/install/update step.
- Hard preflight for each selected family before grounding or baseline work.
- Same review-only protocol and structured output contract for every reviewer.
- Full clean-cycle convergence: every selected family must clear the same final artifact state.

## Adding reviewer families

Reviewer families are extensible only by changing this repository. Do not add a user override file,
project-local adapter registry, environment-provided command, or generic argv template. That would
turn the skill into a configurable command runner and expand the security surface beyond the stable
convergence loop.

To add a supported model family, make it a static built-in profile:

1. Add the family to `skills/multi-llm-convergence-beta/references/reviewer-profiles.md`.
2. Document its fixed official CLI command shape and review-only control.
3. Define the same preflight requirements used by every other reviewer: CLI present, smoke prompt,
   review-only mode accepted, and capturable structured output.
4. Keep the shared review contract and convergence protocol unchanged.
5. Update `SKILL.md`, this README, marketplace metadata, and `CHANGELOG.md`.
6. Run `./validate-skills.sh plugins/multi-llm-convergence-beta/skills/`.

If a model cannot be constrained to review-only behavior or cannot produce parseable findings through
the shared contract, it should not be added as a built-in reviewer family.

## Layout

```
multi-llm-convergence-beta/
├── README.md
├── .claude-plugin/plugin.json
└── skills/multi-llm-convergence-beta/
    ├── SKILL.md
    └── references/
        ├── reviewer-profiles.md
        ├── reviewer-dispatch.md
        └── coding-guidelines.md
```

## Promotion path

When the beta is accepted: copy `SKILL.md` and `references/` into the original
`multi-llm-convergence` skill, rename the skill to the original namespace, bump the original
plugin's version, update marketplace / README / CHANGELOG, then delete
`plugins/multi-llm-convergence-beta/` and its registrations.

## License

MIT - see the repo [LICENSE](../../LICENSE).
