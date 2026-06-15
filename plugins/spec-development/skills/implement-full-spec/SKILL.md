---
name: implement-full-spec
description: "Use when the operator points at a parent ticket, audit, RFC, design doc, or multi-finding report with N actionable subtasks and wants every subtask shipped as its own stacked pull request, then driven to merge-ready by addressing every bot/human review comment and re-requesting review per round. Triggers on: 'fix all subtasks of [ticket]', 'work through P1/P2/P3 items', 'open a PR per finding and stack them', 'ship the audit remediations', 'knock out this report end to end', 'use agent teams to fix [parent]', 'address all comments on these stacked PRs', 'drive the stack to merge-ready', 'do a review-response sweep', 'comment @bot re-review when done'. Depends on the dev-team and pr-autopilot plugins. For a single unit of work, see dev-team; for one PR's review-response, see pr-autopilot."
---

# Implement Full Spec

This skill turns a parent ticket (audit, RFC, multi-finding report, etc.) with N actionable subtasks into a clean stack of N pull requests — and then keeps driving that stack to merge-ready by addressing every comment and review on every PR until reviewers are satisfied. It is the durable version of the workflow proven out on a real multi-finding security audit (24 P1/P2/P3 findings → 24 stacked PRs → multi-round bot review-response cycle).

## Dependencies — check these first

This skill orchestrates two skills that ship as **separate plugins**. Before Phase B, confirm both are installed and invocable:

- **`dev-team`** (plugin: `dev-team`) — the per-subtask Dev → QA → Reviewer → commit loop that Phase B runs once per subtask.
- **`pr-autopilot`** (plugin: `pr-autopilot`) — owns the single-PR review mechanics that Phase C delegates to.

If either `/dev-team` or `/pr-autopilot` is unavailable, stop and tell the operator to install the missing plugin(s) from the same marketplace — e.g. `/plugin install dev-team` and `/plugin install pr-autopilot` — then re-run. Do not reimplement those loops inline.

## When to use this skill

Triggers when the operator points at a parent containing multiple actionable subtasks AND wants each subtask to land as its own PR. Common phrasings:

- "Work through all the P1/P2/P3 subtasks of this ClickUp/Linear/Jira ticket and create a PR for each."
- "Fix all the findings in this audit. One PR per finding, stacked."
- "Use agent teams to fix all the subtasks of [parent]."
- "Address all the comments and bot reviews on PRs #N..#M."
- "Go through these stacked PRs and respond to every review."
- "Drive these to merge-ready. Re-request review when each one is updated."

It is the right skill when **all three** are true:

1. The work is naturally a sequence of small, independently-reviewable units (≥3, typically 5–30).
2. Subtasks share enough surface that stacked branches are cheaper than 30 branches off main.
3. The user wants the orchestration to outlast a single attention span — fold in clarifications, hit review feedback, request re-review.

It is the **wrong** skill when:

- The parent has one big issue, not many subtasks → just open one PR.
- The subtasks are wholly independent (different services, different repos) → open N independent PRs off main.
- The user wants a synchronous pair-programming flow → just work alongside them, no orchestration framework needed.

## Three phases

This skill has three distinct phases. They share state (the worktree set, the orchestrator's conversation memory, the PRs on the forge) but are independently invoked.

| Phase | What happens | Trigger |
|---|---|---|
| **A. Plan** | Read parent, filter actionable subtasks, design the stack order, pin execution parameters. | First user message with a parent ticket URL. |
| **B. Execute** | For each subtask in order: worktree → run the `dev-team` loop (Dev → QA → Reviewer → commit) → push → `gh pr create --base <stack-parent>` → mark ticket complete. | After Phase A is approved. |
| **C. Respond to feedback** | Sweep every PR for inline-thread + top-level-review + general-comment feedback. Classify, fix, commit, push, reply with `Addressed in <SHA>`, cascade-rebase downstream PRs, then ping `@<bot> re-review`. Multi-round until reviewers stop flagging things. | After Phase B is open on the forge, OR triggered fresh ("address all the comments on these PRs"). |

Phase A and B are well-covered by `references/phase-a-plan.md` and `references/phase-b-execute-subtask.md`. Phase C is in `references/phase-c-review-response.md`. Common gotchas across phases are in `references/gotchas.md`. The Phase C agent skeleton and the Phase A kickoff prompt live in `references/prompt-templates.md`.

**The per-subtask Dev → QA → Reviewer loop is its own skill.** Phase B does not implement the loop inline — it calls the `dev-team` skill once per subtask to drive that subtask's spec to a committed, reviewed result, then layers on the stacking, PR, and ticket machinery that's specific to this skill. That keeps the loop reusable: you can fire it off directly on a single change without the full-spec scaffolding. Everything about how that loop runs lives in `dev-team` — this skill delegates to it and never reimplements those mechanics (see "The per-subtask loop, cycle cap, and role collapsing" below).

**For full execution, read the three phase files in order.** For a partial invocation (e.g., the user asks only "respond to comments on the PRs we already opened"), read the relevant phase file directly.

## Top-level workflow

```
operator: "Fix all P1/P2/P3 subtasks of parent ticket X and stack the PRs"
   │
   ▼
PHASE A — Plan
   • Read parent ticket. Filter actionable subtasks by priority/label/criteria.
   • **INTERVIEW the operator on PR strategy — three modes:**
       - (A) Single PR: one commit per subtask, all on one branch, one PR.
       - (B) Stacked PRs: N branches stacked, N PRs (the default for
             dependency-ordered work).
       - (C) Independent PRs with selective stacking: one PR per subtask,
             off main when independent, stacked only on real dependency.
   • Decide subtask order (priority + dependency-ordered within tier).
   • Pin remaining parameters (cycle cap, concurrency, worktrees, etc.).
   • Surface the plan; get user approval before touching code.
   │
   ▼
PHASE B — Execute (per subtask, serially in A/B; serial-or-parallel in C)
   For each subtask in order:
   • Set up worktree per the chosen mode (one shared for A; per-subtask
     for B/C, branched off the planned parent).
   • Run the `dev-team` skill on this subtask's spec:
       Dev → QA → Reviewer/code-review → commit, bounded by a cycle cap.
       It returns an APPROVED & COMMITTED SHA (or escalates). The loop
       commits but does NOT push — that's this phase's job.
   • Push the returned commit.
   • Open PR — varies by mode:
       - Mode A: defer; open one PR at the end of the run.
       - Mode B: `gh pr create --base <stack-parent>` after each subtask.
       - Mode C: `gh pr create --base main` for independent;
                 `--base <dep-branch>` for dependent.
   • Mark ticket complete with SHA + branch + PR URL (or "PR pending" in A).
   • Advance.
   │
   ▼
PHASE C — Respond to feedback (after PRs are open OR on demand)
   • For every PR in order:
       - Fetch ALL THREE comment sources (this is the trap):
           * reviewThreads (inline)
           * pulls/{n}/reviews (top-level bodies)
           * issues/{n}/comments (general)
       - Verify counts so silent-empty fetches are caught.
       - Classify each per the pr-autopilot / pr-review-mechanics decision matrix.
       - Fix actionable items; commit (new commit by default).
       - Push.
       - Reply on each thread/comment with: Addressed in [<short>](<commit-url>).
       - Resolve inline threads via GraphQL resolveReviewThread mutation.
       - Cascade rebase — varies by mode:
           * Mode A: skip (only one PR).
           * Mode B: rebase ALL downstream PRs onto the new tip; force-push.
           * Mode C: rebase only PRs in this PR's dependency chain.
       - Comment "@<reviewer-handle> re-review" so they re-evaluate.
   • Multi-round: bots often respond to re-review with NEW items. Repeat.
```

## The per-subtask loop, cycle cap, and role collapsing

The per-subtask loop — its cycle cap, role-collapsing decision table, agent-team execution model, and commit hard rules — lives in the `dev-team` skill. Phase B calls it once per subtask; read its SKILL.md for the mechanics and the operational cap value. The one knob this skill surfaces is the cycle cap, which Phase A pins as an execution parameter (its default mirrors dev-team's); otherwise this skill delegates the loop and doesn't reimplement it.

Two stack-specific consequences worth stating here:

- **The cycle cap blocks the stack.** When a subtask exhausts the cycle cap without an APPROVED commit, downstream subtasks can't proceed — they need this branch as their base. Escalate to the user (the loop hands you the diff + failing output + last feedback); do not silently retry past the cap.
- **Each subtask gets its own dev team; never share one across subtasks.** Keep your parent-level progress tracking out of any shared TaskList the teammates can see (it leaks and they auto-claim the wrong work) — track it in conversation memory. See the dev-team skill's "Team guardrails" for the rest.

## Hard rules (do not relax without explicit user override)

The loop's own commit-and-workspace invariants — no `--no-verify`, no `git add -A`, no amend by default, don't touch the operator's primary checkout, verbatim spec, mental-revert clause, IN/OUT-of-scope partition — are owned by the `dev-team` skill and inherited by every per-subtask agent. The rules below are the ones **specific to driving a stack of PRs**, which the loop knows nothing about:

- **Amend is the stack's call, not the loop's.** The loop never amends. The one place amend is correct: a bot explicitly flags the original commit's subject/body (e.g., "subject 75 chars > 72") — fix that with `git rebase -i HEAD~N` + `reword`, then force-push and cascade the stack.
- **Stack-aware rebases** after every push: `git rebase --onto <new-base> <local-old-upstream>`. See `references/gotchas.md` for why the naive `<old-upstream-on-origin>` form breaks once upstream has been force-pushed.
- **Three comment sources** in Phase C. Inline review threads + top-level review bodies + issue-level comments. Missing the top-level-review fetch is a documented failure mode; `references/gotchas.md` has the GraphQL/REST snippets that get all three.
- **Reply format**: `Addressed in [\`<short-sha>\`](<commit-url>)`. Inline threads get a `resolveReviewThread` GraphQL mutation; top-level reviews and general comments don't have a resolve op, just the reply.
- **Re-review request**: after pushing changes that address review feedback, post `@<reviewer-bot-handle> re-review` so the bot reruns. Do this for every PR in every round.

## Adapt the plan in flight

The Phase A plan is a starting hypothesis, not a contract. Three things will almost certainly change in flight:

1. **Role collapsing.** A run might start full-fat 3-agent and discover ticket #5 is trivial enough to single-shot. Switch when the evidence supports it — the `dev-team` skill makes this decision per subtask.
2. **Cycle cap behavior.** The loop escalates when its cycle cap is exhausted. When the user has explicitly said "don't stop until done," push the cap with judgment — but report any cap hit even when continuing past it.
3. **Conflict resolution patterns during cascade-rebase.** Architectural changes (auth token format, API shapes) tend to mutate across the stack. The same conflict shape recurs; resolve it consistently. `references/gotchas.md` has a worked example.

If a documented step is producing pain instead of value, change it and write the change down. The plan file at `~/.claude/plans/<task>.md` is the persistence path across context resets — update it; don't pretend the old version still applies.

## Pointers

- `dev-team` — the per-subtask Dev → QA → Reviewer/code-review → commit loop, cycle cap, role collapsing, commit hard rules, and the Dev/QA/Reviewer prompt skeletons. Phase B calls this once per subtask.
- **`pr-autopilot` skill (separate plugin)** — owns the single-PR review mechanics (three-source fetch, classify, reply/resolve, per-bot re-review triggers) in its `pr-review-mechanics.md` reference. Phase C is the stacked-PR superset: it delegates each individual PR's comment mechanics to `/pr-autopilot --single-pass`, then layers the cascade-rebase + per-round re-review in `references/phase-c-review-response.md` on top. Requires the `pr-autopilot` plugin (see "Dependencies — check these first" above).
- `references/phase-a-plan.md` — survey the parent, filter, design the stack, present the plan
- `references/phase-b-execute-subtask.md` — the per-subtask flow: worktree → loop → push → PR → ticket, in mechanical detail
- `references/phase-c-review-response.md` — three-source fetch + classify + fix + reply + cascade-rebase + re-review ping
- `references/prompt-templates.md` — Phase C agent skeleton + Phase A kickoff prompt; copy-paste
- `references/gotchas.md` — cross-cutting traps: jq+control-chars, three-source fetch, rebase --onto, force-push-with-lease, amend-vs-new-commit, evolving-API conflicts
