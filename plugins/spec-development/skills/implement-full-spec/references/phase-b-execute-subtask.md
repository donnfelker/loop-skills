# Phase B — Execute a Subtask

This file is the per-subtask loop. It runs once for each filtered subtask, in stack order.

## Loop overview

```
For each subtask N in stack order:
  ┌────────────────────────────────────────────────────────┐
  │ 1. Set up the worktree                                  │
  │ 2. Update ticket status → "in progress"                 │
  │ 3. Run the dev-team skill on this subtask               │
  │      → returns an APPROVED & COMMITTED SHA, or escalates│
  │ 4. Push the returned commit + gh pr create (mode-dep.)  │
  │ 5. Comment SHA + branch + PR URL; mark ticket complete  │
  │ 6. Advance to N+1, parent = N's branch                  │
  │ 7. (Mode A only) Open the single PR after the last one  │
  └────────────────────────────────────────────────────────┘
```

The Dev → QA → Reviewer loop itself — including its cycle cap and role collapsing — is the `dev-team` skill. This phase is the wrapper around it: set up the worktree, run the loop, then push / open the PR / close the ticket. When the loop escalates (cycle cap hit), the stack is blocked — see Escalations below.

## Step 1: Set up the worktree

The worktree path and the parent branch come from the Phase A plan. The setup differs by Phase A mode:

### Mode A — Single PR

Worktree + branch created ONCE, at the start of the run. Reused for every subtask.

```bash
# At the start of Phase B, before any subtask:
git worktree add -b <feat-prefix>/<task-slug> ~/.claude-worktrees/<task-slug>/ main
```

For subtasks 2..N, Step 1 is a no-op — the worktree already exists, just verify it:

```bash
git -C ~/.claude-worktrees/<task-slug>/ status --short
git -C ~/.claude-worktrees/<task-slug>/ log --oneline -5
```

### Mode B — Stacked PRs

Per-subtask worktree, each branched off the prior subtask's branch.

```bash
git worktree add -b <new-branch> <worktree-path> <stack-parent-branch>
```

For subtask #1, `<stack-parent-branch>` is `main`. For every subsequent subtask, it's the prior subtask's branch.

### Mode C — Independent PRs with selective stacking

Per-subtask worktree. `<stack-parent-branch>` comes from the "Depends on" column in the Phase A plan: `main` for independent subtasks, the depended-on subtask's branch for dependent ones.

```bash
git worktree add -b <new-branch> <worktree-path> <parent-branch-from-plan>
```

### Verify (all modes)

After creating the worktree, verify the state before dispatching the dev:

```bash
git -C <worktree-path> status --short
git -C <worktree-path> log --oneline -3
```

## Step 2: Update the ticket to in-progress

Examples by tracker:

| Tracker | Status update |
|---|---|
| ClickUp | `clickup_update_task` with `status: "in progress"` |
| Linear | `update_issue` with `state: "In Progress"` |
| Jira | `transitionJiraIssue` to the in-progress transition id |
| GitHub Issue | `gh issue edit <n> --add-label "in-progress"` |

This is intentional friction — it broadcasts that work has started, useful when other humans are watching the tracker.

## Step 3: Run the dev-team skill

This is where the subtask actually gets implemented, verified, reviewed, and committed. Do **not** re-implement the Dev → QA → Reviewer cycle here — invoke the `dev-team` skill and let it drive the cycle.

Hand the loop this subtask's contract:

- **Workspace**: the worktree path from Step 1 and the branch name.
- **Verbatim spec**: the full subtask spec — severity, file paths + line numbers, description, impact, every remediation bullet. No summarization.
- **IN/OUT-of-scope partition** for every bullet. If a bullet requires infrastructure outside the repo (creating a GitHub App, provisioning a KV namespace, deploying a sidecar), mark it OUT OF SCOPE with the proposed follow-up ticket name.
- **Canonical checks**: the exact commands the operator expects green (typecheck, lint, the specific package test suites, integration tests if applicable).
- **Stack note**: this branch stacks on `<upstream subtasks>` — tell the Dev to leverage their primitives.

The loop returns an **APPROVED & COMMITTED** SHA, or it escalates (cycle cap hit, spec ambiguity, out-of-scope infrastructure). It commits but does **not** push — that's Step 4. On escalation, the stack is blocked; surface it per Escalations below rather than advancing.

The loop owns its own execution model, cycle cap, role collapsing, commit hard rules, and prompt skeletons. Read the `dev-team` SKILL.md for those mechanics and the operational cap value — this file doesn't re-document them. (Phase A pins the cap as a planning parameter; that default should mirror dev-team's.)

## Step 4: Push the commit (and open the PR if appropriate)

After the loop returns APPROVED & COMMITTED (with a SHA), the orchestrator pushes. **Whether to open the PR right now depends on the mode.**

### Mode A — Single PR

Push the commit to the shared branch. **Do not open a PR after each subtask** — Mode A opens one PR at the end of the whole run (or once, as a draft, at the start with subsequent pushes appending commits).

```bash
git -C <worktree> push -u origin <branch>
```

If a draft PR was opened at the start of the run, the push will appear as a new commit on it; no further action. If not, defer the PR open until after the last subtask is committed.

### Mode B — Stacked PRs

Push, then open a PR with `--base <stack-parent>`:

```bash
git -C <worktree> push -u origin <branch>
gh pr create \
  --base <stack-parent-branch> \
  --head <branch> \
  --title "<conventional-commit-style subject>" \
  --body "$(cat <<'EOF'
## Summary
<1–3 bullets>

## What was implemented
<bullets from the spec — IN-SCOPE only>

## Deferred (proposed follow-ups)
- <bullet> — <new-ticket-id-if-any>

## Stack position
- Base: <stack-parent>
- Next in stack: <next-subtask> (or "none — top of stack")

## Test plan
- [ ] <command 1>
- [ ] <command 2>
EOF
)"
```

`--base <stack-parent>` is what makes this PR base on the prior subtask's branch instead of `main`. GitHub auto-rebases children when the parent merges.

### Mode C — Independent PRs with selective stacking

Push, then open with `--base` set per the Phase A plan — `main` for independent subtasks, the dependency's branch for dependent ones.

```bash
git -C <worktree> push -u origin <branch>
gh pr create \
  --base <parent-branch-from-plan> \
  --head <branch> \
  --title "..." \
  --body "$(cat <<'EOF'
## Summary
<1–3 bullets>

## What was implemented
<bullets from the spec — IN-SCOPE only>

## Dependency
<"This PR is independent and bases on main." OR
 "This PR depends on PR #<N> (<subtask-id>) and bases on its branch. Merge that PR first.">

## Test plan
- [ ] <command 1>
EOF
)"
```

Be explicit in the PR body when the PR has a non-`main` base — reviewers won't intuit dependency relationships from branch names alone.

## Step 5: Close the loop on the ticket

After the commit lands (and the PR is open, if appropriate for the mode), comment on the source ticket with the addressing artifacts and mark complete.

For Mode A, the ticket comment goes up after each subtask commit, even though the PR doesn't exist yet:

```
Implemented in <full-sha> on <branch>; PR pending (Mode A: single-PR run).
```

For Mode B / Mode C:

```
Resolved in <full-sha> on <branch>, PR: <url>
```

Then update status to `complete` (or equivalent). The ticket now records SHA + branch + PR URL (or pending status, for Mode A), which lets future archaeology re-find the work.

## Step 6: Advance

The next subtask's parent depends on the mode:

| Mode | Next subtask's parent |
|---|---|
| A | Same branch as this subtask. Same worktree. |
| B | This subtask's branch (stack continues). |
| C | Looked up in the Phase A plan's "Depends on" column. |

Don't re-fetch the parent or re-derive the queue — both are already in the Phase A plan. Move on.

## Step 7 (Mode A only): Open the PR after the final subtask

For Mode A, when the last subtask in the run has been committed and pushed, open one PR covering the whole batch:

```bash
gh pr create \
  --base main \
  --head <feat-prefix>/<task-slug> \
  --title "<conventional-commit-style subject describing the whole batch>" \
  --body "$(cat <<'EOF'
## Summary
<one-sentence description of the spec being implemented>

## What was implemented
<one bullet per subtask, in commit order, each linking to the source ticket>
- <subtask-1>: <one-line description> (<ticket-url>)
- <subtask-2>: ...
- ...

## Deferred (proposed follow-ups)
- ...

## Test plan
- [ ] <command 1>
- [ ] <command 2>
EOF
)"
```

Then go back to the tracker and update each subtask's "PR pending" comment with the now-known PR URL.

## Escalations

These conditions mean STOP and report to the operator. Do not silently retry:

- **The loop escalated** — cycle cap exhausted on the subtask, spec ambiguity, or out-of-scope infrastructure required. The `dev-team` skill surfaces these with the diff, failing output, and last feedback; relay that to the operator. A cycle-cap escalation blocks the stack, since downstream subtasks need this branch as their base.
- **Stack-broken conflict** (the rebase from prior subtasks introduced a real conflict the dev can't trivially resolve). This one is Phase B's own — it lives in the worktree setup / advance steps, not inside the loop. Surface the conflicting hunks.

## Collapsing roles for smaller subtasks

Role collapsing (full 3-agent vs combined QA+Reviewer vs single-shot) is decided per subtask **inside the loop** — it's owned by the `dev-team` skill, which has the decision table. Don't pre-commit to one pattern across the whole stack; let the loop pick based on each subtask's complexity.

## What this phase produces

- One commit per subtask on its own branch.
- One open PR per subtask, each based on the prior subtask's branch.
- The source ticket marked complete with SHA + branch + PR URL.
- Updated orchestrator memory: progress queue, any cycle-2 events, any deferred items captured as proposed follow-ups.

Hand off to Phase C (`phase-c-review-response.md`) once all subtasks have open PRs and you (or the user) are ready to start the review-response sweep.
