# multi-llm-convergence-beta

> **⚠️ Beta / experimental.** This plugin ships **in parallel** with the stable
> [`multi-llm-convergence`](../multi-llm-convergence/) (v0.3.0), which stays **untouched**. Install
> and exercise both side by side. When the beta proves out it is promoted into the original namespace
> and this plugin is removed (see [Promotion path](#promotion-path)).

A **host-agnostic, configurable, N-model** convergence loop. It drives any artifact — a plan, PRD,
design doc, spec, or implementation diff — to genuine cross-model consensus, then stops. Three things
distinguish it from the stable skill:

1. **Direct-CLI delegation.** Reach every model by shelling out to its own CLI — `codex exec`,
   `claude -p`, `gemini -p`, `grok -p` — instead of the Codex companion script. One symmetric
   primitive, no plugin dependency.
2. **Configurable adapter registry.** A JSON catalog (`assets/adapters.json`) of argv-array
   invocation templates. Built-ins for `claude`, `codex`, `gemini`, `grok`; adding any other CLI is a
   data edit (see [`references/adding-an-adapter.md`](skills/multi-llm-convergence-beta/references/adding-an-adapter.md)).
3. **User-selected reviewer set.** At startup the operator picks which models converge (≥2 distinct
   families). The host is a pure orchestrator and need not be one of the reviewers.

It keeps the original loop's guarantees: local grounding so offline reviewers don't stall, a liveness
watchdog so a silent reviewer can't hang the loop, per-round commits for an auditable trail, and a
**hard preflight stop** when fewer than two reviewers can be functionally verified (reachable *and*
read-only).

## When to use this vs. the stable skill

- **Use `multi-llm-convergence` (stable)** for the proven, hardcoded two-model loop: a Claude driver
  plus one Codex reviewer via the Codex Claude Code plugin.
- **Use `multi-llm-convergence-beta`** when you want to choose which models converge, converge with
  **three or more** models, run from a **non-Claude host**, or add a new LLM CLI without rewriting the
  skill.

With both plugins installed, invoke the beta deliberately — its description leads with
**(beta/experimental)** and carries distinct triggers (`beta convergence`, `converge with codex and
gemini`, `N-model convergence`, `direct-CLI convergence`) so the two don't fire ambiguously.

## Read-only safety

A reviewer must only read the artifact, never edit it. Each adapter runs with the strongest read-only
flag its CLI offers (codex `-s read-only` is a hard sandbox; claude/grok/gemini use `plan` mode, which
is softer), and the review contract says "review only; do not edit." That's enough for the **trusted**
artifacts this loop converges. For **untrusted** code, run the reviewer under an OS sandbox (Docker, or
macOS Seatbelt). `gemini` and `grok` are **validate-on-first-use** — neither was locally smoke-tested
at design time.

## Layout

```
multi-llm-convergence-beta/
├── README.md                                  # this file
├── .claude-plugin/plugin.json                 # name, version 0.1.0, license MIT
└── skills/multi-llm-convergence-beta/
    ├── SKILL.md                               # the loop
    ├── assets/adapters.json                   # canonical registry (claude, codex, gemini, grok)
    └── references/
        ├── reviewer-dispatch.md               # contract + per-adapter dispatch + watchdog + probe
        ├── adding-an-adapter.md               # field schema + worked example + override path
        └── coding-guidelines.md               # behavioral rules (copied from the stable skill)
```

## Promotion path

When the beta is accepted: copy `SKILL.md`, `assets/`, and `references/` into the original
`multi-llm-convergence` skill (renamed to the original skill name), bump the original plugin's
version, update its `marketplace.json` / README / CHANGELOG, then delete
`plugins/multi-llm-convergence-beta/` and its registrations. This notice exists so the intent is
explicit.

## License

MIT — see the repo [LICENSE](../../LICENSE).
