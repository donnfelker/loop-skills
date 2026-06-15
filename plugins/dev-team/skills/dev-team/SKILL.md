---
name: dev-team
description: "Drive a single unit of work — one spec, ticket, finding, or change request — from spec to a committed, reviewed result with a dev team of coordinating agents: Dev → QA → Reviewer/code-review in a bounded cycle. Runs as an agent team when the harness supports one, else coordinated subagents, so it works across agents and models. Use whenever the operator wants a change implemented with rigor instead of a quick edit: 'use a dev team to build and verify this', 'fire off the dev team on this', 'spin up the dev/QA/review agents', 'implement this with a dev/QA/review loop', 'drive this change through dev then QA then code review', 'QA and review this before committing'. This is also the per-subtask engine behind implement-full-spec — that skill calls this dev team once per subtask, then handles stacking, PRs, and review-response on its own. Use this skill for ONE unit of work; for a parent ticket with many subtasks shipped as stacked PRs, use implement-full-spec (which delegates here)."
---

# Dev Team — Dev → QA → Reviewer Loop

This skill drives **one unit of work** — a spec, ticket, finding, or change request — from its written spec to a **committed, reviewed result**. It does this by standing up a small **agent team** and running it through a bounded loop: a Dev implements, a QA verifies ("trust nothing"), a Reviewer does the code review and creates the commit. On any failure the loop sends the Dev the specific feedback, up to a cycle cap, then escalates. Fire it off and it builds the team for you.

It is the reusable core of a rigorous implementation flow. You can fire it off on whatever you're working on right now, and `implement-full-spec` calls it once per subtask before layering on its own stacking, PR, and review-response machinery.

## When to use this skill

Use it when the operator wants a change implemented with rigor — built, independently verified, and code-reviewed before it lands — rather than a quick inline edit. Common phrasings:

- "Implement this spec with a dev → QA → review loop."
- "Fire off the dev team on this task."
- "Use agents to build this and verify it before committing."
- "Drive this change through dev, then QA, then code review."

It is the **wrong** skill when:

- The change is a one-line tweak you can make and verify directly — the loop's overhead buys nothing.
- The work is a parent ticket with many subtasks that each want their own PR → use `implement-full-spec`, which calls this loop per subtask.
- The operator wants synchronous pair-programming → just work alongside them.

## What this loop needs (the contract)

Before dispatching the first agent, pin these. They are the inputs every role agent inherits — the prompt is the contract, so gather them up front:

| Input | What it is |
|---|---|
| **Workspace** | The absolute path the agents work in (a worktree or a checkout) and the branch. Agents must stay inside it. |
| **Verbatim spec** | The full spec for this unit of work — severity, file paths, description, impact, and **every** remediation bullet. Never summarize; the prompt is the contract. |
| **IN/OUT-of-scope partition** | For every remediation bullet, mark it IN SCOPE or OUT OF SCOPE (with a proposed follow-up name). Bullets needing infrastructure outside the repo are OUT OF SCOPE. |
| **Canonical checks** | The exact commands the operator expects to be green: typecheck, lint, the specific test suites, integration tests if any. |
| **Commit terminus** | The loop ends at a commit on the branch. Pushing / opening a PR is the **caller's** job — the loop does not push. (`implement-full-spec` pushes and opens the PR after the loop returns.) |

If you don't have the verbatim spec or the canonical checks, get them before starting — a loop run on a vague spec produces a confident wrong commit.

## Execution model: run the roles as an agent team

When you fire off this skill, **stand up an agent team** to run the roles — don't just make isolated one-off calls. The Dev, QA, and Reviewer are teammates working one unit of work together, with you as the team lead sequencing the hand-offs. Running them as a team means each role shares the unit's evolving context — the spec, the diff, prior-cycle feedback — instead of rebuilding it from a cold prompt every time.

Use the richest multi-agent primitive your harness offers, and degrade gracefully. This is what keeps the skill usable across different agents and models:

- **Agent-team primitive available** (e.g., Claude Code's `TeamCreate` + `SendMessage`, or any equivalent multi-agent/team feature): create a team scoped to this one unit of work, add Dev / QA / Reviewer as teammates, and coordinate them over the team's messaging channel. Cycle-2+ feedback flows from QA/Reviewer straight to the Dev teammate without you re-pasting it.
- **No team primitive** (single-shot subagent dispatch only, or a runtime/model without teams): fall back to a **coordinated set of subagents** — dispatch Dev, read its result, dispatch QA with that result, dispatch the Reviewer with the diff + QA's findings. You are the communication bus, carrying each role's output to the next.

Either way the sequence and the contract are identical: Dev → QA → Reviewer → commit, bounded by the cycle cap. Below, **"dispatch a role" means assign it to the teammate** (team mode) or **spawn the subagent** (fallback mode). See "Team guardrails" for the sharp edges.

## The loop

```
Pin the contract (workspace, verbatim spec, IN/OUT scope, canonical checks)
   │
   ▼
1. Dev role (verbatim spec, leaves changes UNSTAGED)
   │
   ▼
2. QA role ("trust nothing — read code and run tests yourself")
   │   └── FAILED → send Dev QA's itemized findings (cycle N+1)
   ▼
3. Reviewer + commit role (reads the diff, code-reviews, commits)
   │   └── REJECTED → back to Dev with the itemized changes (cycle N+1)
   ▼
APPROVED & COMMITTED → return the SHA to the caller
```

Each role is a distinct agent — a teammate in team mode, or a one-shot subagent in fallback mode (see the execution model above). The sequence is the same regardless: each role's output feeds the next, and you sequence the hand-offs. The prompt skeletons are in [`references/prompt-templates.md`](references/prompt-templates.md) — copy them, fill the `<angle-brackets>`, adapt to the spec.

### Step 1 — Dispatch the Dev Agent

Pass the verbatim spec, workspace, IN/OUT-of-scope partition, and the expected READY report format. Skeleton: [`prompt-templates.md#dev-agent`](references/prompt-templates.md#dev-agent). Critical content:

- **Verbatim spec** — severity, file paths + line numbers, description, impact, every remediation bullet. No summarization.
- **IN/OUT-of-scope partition** for every bullet, with proposed follow-up names for the deferred ones.
- **CLAUDE.md reference** — say "follow CLAUDE.md conventions" and trust inheritance; don't re-list conventions.
- **Workspace restriction** — explicitly name the operator's primary checkout and say "do not touch it." The agent `cd`s into the workspace and stays there.
- **Do NOT commit** — leave changes unstaged so the Reviewer can produce one consistent commit message.
- **READY report format** — files changed grouped by package, approach summary, tests added, deferred items, anything escalated.

### Step 2 — Dispatch the QA Agent

After Dev returns READY, dispatch the QA role. Skeleton: [`prompt-templates.md#qa-agent`](references/prompt-templates.md#qa-agent). Critical content:

- **"Trust nothing — verify by reading code and running tests yourself."** Without this opener, QA rubber-stamps.
- **Mental-revert clause** — "For each new test, ask: if you reverted the fix to its previous form, which assertions would fail? If none, the tests are theater. Flag that." This is the single highest-leverage line; it catches tests that pass whether or not the fix exists.
- **Bullet-by-bullet verification** — for each remediation bullet, find the code that addresses it and confirm the implementation achieves the bullet's intent (not log-only, not partial).
- **Canonical checks** — the exact commands the operator expects to be green.
- **PASSED / FAILED return** — on FAILED, itemize `file:line — severity — proposed remediation`. Nits-only findings → PASSED with nits noted.

On FAILED: re-dispatch the Dev with QA's itemized findings as additional context. That is the next cycle.

### Step 3 — Dispatch the Reviewer + commit Agent

After QA PASSED, dispatch the Reviewer role that also creates the commit. For trivial units, combine QA and Reviewer into one agent (see role-collapsing below). Skeleton: [`prompt-templates.md#reviewer--commit-agent`](references/prompt-templates.md#reviewer--commit-agent). Critical content:

- **Read the diff first** — the actual diff, not the Dev's claims.
- **Code-review skill** — use `superpowers:requesting-code-review` if available.
- **Manual checklist** — correctness, security, style per CLAUDE.md, no new deps, no back-compat shims, imports/dead code clean.
- **APPROVED → commit** — the Reviewer is the agent that creates the commit. Conventional-commit message; subject ≤72 chars; body lists what changed and why, plus deferred items and any `Closes` link.
- **REJECTED → itemized changes needed** — back to Dev for the next cycle.

The Reviewer commits but does **not** push. Pushing and opening any PR is the caller's decision.

## Cycle cap

**3 Dev → QA → Reviewer cycles total per unit of work.** Most units pass on cycle 1. If you hit cycle 3 without an APPROVED commit, stop and escalate to the operator with: the current diff, the failing test output, the reviewer's last feedback, and what a cycle 4 might try. Do not silently retry past 3 — a unit that won't converge in three cycles usually has a spec problem, not an implementation problem.

**You (the lead) own the count.** Cycle 1 is the first Dev pass; each time a QA FAIL or Reviewer REJECT sends work back to the Dev, that's the next cycle. The `cycle N` label in a role agent's return and the `(cycle N+1)` markers in the loop diagram are advisory — they describe the same counter, but a role agent can't see how many cycles preceded it, so don't rely on its number. Track the count yourself and apply the cap.

When the operator has explicitly said "don't stop until done," you may push past the cap with judgment — but still report every cycle-3 hit, even when continuing.

## Collapsing roles for smaller units

The full Dev → QA → Reviewer pipeline is full-fat for architectural changes and collapses down for simpler work. Decide per unit based on the spec's complexity, not by habit:

| Signal | Pattern | Agent calls |
|---|---|---|
| Architectural change, multi-package, type changes, new infrastructure | Separate Dev, QA, Reviewer+commit | 3 |
| Single package, 2–5 files, no API change | Dev → combined QA+Reviewer+commit | 2 |
| One file, one comment, trivial fix | Single Dev+verify+commit agent | 1 |
| Cycle 2+ on any unit | Always keep Dev separate so it sees the specific feedback | n |

The combined and single-shot skeletons are in `prompt-templates.md`. Collapsing trades isolation for speed — a one-comment-in-one-file fix doesn't warrant three round-trips, but anything that changes an API or touches multiple packages wants the independent QA pass.

## Team guardrails

A team is the right tool here, but it has sharp edges. Each of these is a real failure someone already hit, not a hypothetical:

- **Scope the team to ONE unit of work, and tear it down at the commit.** A team built for unit A will auto-claim and mangle unit B's work if you reuse it. Once the Reviewer commits, the team's job is done — close it.
- **Keep your own progress-tracking out of any shared TaskList the teammates can see.** Tasks you create to track your own work leak to teammates, who auto-claim them and start the wrong thing. Track your own progress in conversation memory, not a shared list.
- **Don't let idle teammates burn context.** Waiting teammates emit idle notifications every 10–15s. Sequence the hand-offs so a role is only active when it has work, and stop the team promptly once the unit is committed.
- **For a genuinely trivial unit, skip the team entirely.** A one-line, one-file fix you collapse to a single agent (see role-collapsing above) doesn't need a team — the coordination overhead would dwarf the work. Teams earn their keep when there are real hand-offs between distinct roles.

## Hard rules (do not relax without explicit operator override)

These are the commit-and-workspace invariants every role agent inherits. They exist because each one is a real failure someone already hit.

- **No `--no-verify`** on commits. Hook failures are signal — fix the code, don't bypass the hook. The only legitimate exception is a genuinely broken hook whose fix is out of scope, and even then prefer fixing the hook.
- **No `git add -A` / `git add .`** — stage files by name. Worktrees carry untracked leftovers (caches, `.DS_Store`, stray lockfiles) that `-A` sweeps into the commit.
- **No amend by default** — add a new commit on top so history stays readable. (The one place amend is correct — a reviewer flagging the original commit's subject/body — belongs to whatever caller owns the push/rebase, not this loop.)
- **Don't touch the operator's primary checkout.** Every agent works exclusively inside the workspace path it was given.
- **Verbatim spec in every Dev/QA/Reviewer prompt.** The prompt is the contract; summarizing it loses the bullets the QA verifies against.
- **Mental-revert clause in every QA prompt** — "if the fix were reverted, which test would fail? If none, the tests are theater."
- **Explicit IN/OUT-of-scope partition** for every remediation bullet in the Dev prompt.

## Escalations

These conditions mean STOP and report to the operator — do not silently retry:

- **Cycle cap exceeded** (3 cycles on the same unit). Report the current diff, the failing tests, the reviewer's last feedback, and what cycle 4 might try.
- **Spec ambiguity** — the remediation bullets contradict each other, or the stated impact doesn't match the proposed fix. Don't guess; ask.
- **Out-of-scope infrastructure required** for the minimum credible fix (e.g., provisioning new cloud infrastructure or creating a third-party app integration) — that's an operator action, not an in-repo edit. Surface it and decide together.

## What this loop produces

- One commit on the branch (unstaged → reviewed → committed), with the SHA returned to the caller.
- Or an escalation with the diff, failing output, and last feedback if the cycle cap was hit.
- Pushing, opening a PR, and updating any ticket are the **caller's** responsibility — this loop stops at the commit. When `implement-full-spec` is the caller, it does this in Phase B. When you run `dev-team` standalone, *you* are the caller: end by surfacing the committed-but-unpushed state to the operator — e.g. "Committed `<sha>` on `<branch>`; not pushed. Want it pushed / a PR opened?" — so the work doesn't silently sit unpushed.

## Pointers

- [`references/prompt-templates.md`](references/prompt-templates.md) — copy-paste skeletons for the Dev, QA, Reviewer+commit, and combined single-shot agents.
