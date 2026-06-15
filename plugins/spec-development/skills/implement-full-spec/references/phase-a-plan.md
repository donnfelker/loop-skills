# Phase A — Plan the Stack

Read the parent. Filter the subtasks. Design the stack order. Pin the execution parameters. Get user approval. Do not touch code until this phase exits.

## Step 1: Read the parent

Fetch the parent ticket / audit / issue verbatim. Examples:

| Source | Fetch verb |
|---|---|
| ClickUp | `mcp__clickup__clickup_get_task <id>` and `clickup_get_task_comments` |
| Linear | `mcp__linear__get_issue` (parent) and `get_issue_children` |
| Jira | `mcp__atlassian__getJiraIssue` (parent), follow `subtasks` link |
| GitHub issue | `gh issue view <n>` + linked child issues |
| Markdown report | Read the file |

Capture the structural fields each subtask exposes. For a security audit those are usually: **severity / priority** (filter key), **component / location** (file paths), **description** (the WHAT), **data flow / impact** (the WHY), **remediation bullets** (the actionable HOW). For RFCs/design docs the field set is different — read enough subtasks to learn the shape.

## Step 2: Filter to actionable subtasks

The operator usually states the filter: "P1/P2/P3", "all status=open", "tagged security". Apply it. Exclude:

- Informational findings (the audit equivalent of "fyi this could be better")
- Already-merged or already-closed items
- Subtasks that explicitly say "deferred" or "tracked separately"

Number the survivors. If the count is <3, this skill is overkill — just open one PR per subtask off `main`. If the count is >30, surface that to the operator before designing the stack; deep stacks become painful around 20–25.

## Step 3: Decide stack order

For each filtered subtask, pick an order. Two questions decide it:

1. **Dependencies**: does subtask N+1 use a primitive that subtask N introduces? (E.g., a per-peer HMAC scheme used by 10 later tickets.) Architectural prerequisites go first.
2. **Priority within dependency tier**: within a dependency-free group, do highest-priority first. P1 before P2 before P3.

Reflect the audit / report's recommended remediation order if it has one; the author usually thought about this. Stack-merge order is the same as stack-create order, so flag explicitly if the operator should merge in any order other than the stack order.

## Step 4: Interview the operator on PR strategy

Before pinning the rest of the parameters, the PR strategy must be settled — every other parameter follows from it. Ask the operator directly via `AskUserQuestion`; do not assume the default.

The three modes are not interchangeable:

### Mode A — Single PR (one commit per subtask)

All subtasks land as separate commits on **one branch**, opened as **one PR** off `main`. The reviewer agent commits each subtask in sequence on the same branch.

- **Branching**: one feature branch off `main`, no stacking.
- **Worktrees**: one worktree shared across all subtasks (no per-subtask isolation).
- **PR open**: opens once, at the end, after every subtask is committed. Or as a draft at the start with updates as commits land.
- **Phase C cascade**: trivially none — there's one PR, no downstream.
- **Best for**: closely-related changes where reviewing the subtasks in isolation buys little, or where the operator wants a single merge event.
- **Cost**: large diff for the reviewer to read; bisecting a regression after merge requires identifying the right commit, not the right PR.

### Mode B — Stacked PRs (one PR per subtask, each branch on the previous)

Each subtask is **its own branch**, each based on the previous subtask's branch. N subtasks → N PRs, each with `--base <prior-branch>`.

- **Branching**: linear stack off `main`.
- **Worktrees**: one per subtask at `~/.claude-worktrees/<branch>/`.
- **PR open**: one `gh pr create --base <stack-parent>` per subtask, immediately after commit.
- **Phase C cascade**: when an earlier PR in the stack is updated, every downstream PR's branch must be rebased onto its parent's new tip and force-pushed.
- **Best for**: dependency-ordered subtasks (each builds on the previous), reviewers who want to sign off in tiers, change sets large enough that one PR would be unreviewable.
- **Cost**: cascade-rebase machinery in Phase C; stack-tool discipline; force-push prompts from `dcg` on every parent update.

### Mode C — Independent PRs with selective stacking (dependency-aware)

Each subtask is its own PR. Subtasks with **no dependency** on prior subtasks branch off `main` directly. Subtasks that **do depend** on a prior subtask branch off that subtask (creating a small stack for just that chain).

- **Branching**: one branch per subtask. Most branches off `main`; chains form only where dependency exists.
- **Worktrees**: one per subtask.
- **PR open**: `gh pr create --base main` for independent subtasks; `gh pr create --base <dependency-branch>` for dependent ones.
- **Phase C cascade**: cascade only along dependency chains, not across the whole batch.
- **Best for**: a batch of subtasks where most are independent (e.g., "fix these 10 unrelated lint findings") and only a handful share machinery.
- **Cost**: requires accurate dependency analysis during planning — getting it wrong (declaring a real dependency as independent) produces merge conflicts later. See "Dependency analysis" below.

### How to interview the operator

Use `AskUserQuestion` (or its equivalent) with these three modes as options. Present each with one-line tradeoffs. Don't pick for them — pick for them only if their original request unambiguously specified ("stack them all" → Mode B; "one big PR with everything" → Mode A; "open a PR for each finding" → Mode C if obvious independence, Mode B if dependency).

Example interview prompt:

> "How should the PRs be structured?
> - **(A) Single PR** — all subtasks land as separate commits on one branch, opened as one PR. Best for tightly-related changes.
> - **(B) Stacked PRs** — N branches, each stacked on the previous, N PRs (recommended for dependency-ordered work).
> - **(C) Independent PRs with selective stacking** — one branch per subtask, off main when independent, stacked only where there's a real dependency."

### Dependency analysis (only for Mode C)

If the operator picks Mode C, before designing the stack you have to know which subtasks depend on which. Inputs:

- Subtask descriptions — does subtask N+5 reference primitives, files, or APIs introduced by subtask N?
- File overlap — do two subtasks edit the same files? If yes, one usually depends on the other.
- The audit / report author's stated order — if they ordered the findings deliberately, treat that as evidence of dependency.

Output a small directed graph: nodes are subtasks, edges are "B depends on A". Subtasks with in-degree 0 branch off `main`. Subtasks with in-degree ≥ 1 branch off their (latest) dependency. Each chain is a mini-stack.

If you can't confidently classify a subtask, ask the operator about that specific one rather than guessing. A wrong classification ("independent" when it actually depends) means merge conflict at PR-time; a wrong classification ("dependent" when it actually isn't) just means an unnecessary stack edge — cheaper to recover from.

## Step 5: Pin the other execution parameters

Once PR strategy is chosen, the other parameters follow. Surface them to the operator before exiting plan mode.

| Parameter | Choices | Default by strategy |
|---|---|---|
| **Worktrees** | yes (per-subtask) · no (shared checkout) | yes for Mode B / C; no for Mode A |
| **Concurrency** | strict serial · N-parallel | strict serial for Mode A and Mode B; up to N-parallel for Mode C's independent subtasks |
| **Cycle cap** | bounded (recommend 3, mirroring the `dev-team` default) · unbounded | 3, then escalate |
| **Canonical checks** | the exact typecheck/lint/test commands the operator expects green | required — the `dev-team` takes these as a contract input per subtask, so pin them now |
| **Tier boundaries** (Mode B/C only) | pause for go/no-go · continue with sweep | continue with quick sweep |
| **Stop condition** | each tier · all subtasks · only on hard block | all subtasks, only stop on hard block |

If the operator's request explicitly answers any of these ("ensure all done, only stop if absolutely necessary"), record the answer and proceed.

## Step 6: Lay out the worktree + branch plan

The shape of this plan depends on the strategy picked in Step 4.

### Mode A — Single PR

One feature branch, one worktree, all subtasks share both.

| Worktree | Branch | Base | Notes |
|---|---|---|---|
| `~/.claude-worktrees/<task-slug>/` | `<feat-prefix>/<task-slug>` | `main` | One commit per subtask lands here in order. |

### Mode B — Stacked PRs

For each subtask in order, produce a row:

| # | Subtask ID | Branch | Worktree | Parent branch |
|---|---|---|---|---|
| 1 | PROJ-003 | `security/proj-003-...` | `~/.claude-worktrees/proj-003-.../` | `main` |
| 2 | PROJ-001 | `security/proj-001-...` | `~/.claude-worktrees/proj-001-.../` | `security/proj-003-...` |
| 3 | PROJ-004 | `security/proj-004-...` | `~/.claude-worktrees/proj-004-.../` | `security/proj-001-...` |
| ... | | | | |

### Mode C — Independent PRs with selective stacking

For each subtask, produce a row noting whether it stacks on a dependency or branches off `main`:

| # | Subtask ID | Branch | Worktree | Parent branch | Depends on |
|---|---|---|---|---|---|
| 1 | PROJ-003 | `security/proj-003-...` | `~/.claude-worktrees/proj-003-.../` | `main` | — |
| 2 | PROJ-014 | `security/proj-014-...` | `~/.claude-worktrees/proj-014-.../` | `main` | — |
| 3 | PROJ-007 | `security/proj-007-...` | `~/.claude-worktrees/proj-007-.../` | `security/proj-004-...` | PROJ-004 |

The "Depends on" column is the output of Step 4's dependency analysis. Subtasks with no dependency get `main` as their parent.

### Slug rules (all modes)

Short-kebab from the subtask title, total branch length ≤50 chars, prefix that signals the work-type (`security/`, `feat/`, `chore/`).

## Step 7: Write a stable plan file

Save the plan to `~/.claude/plans/<task-slug>.md` so it survives context resets. Include:

- The parent URL (single source of truth)
- The filtered subtask list with order numbering
- The chosen PR strategy (Mode A / B / C)
- The dependency graph (Mode C only)
- The execution parameters with their pinned values
- The branch + worktree + parent-branch table from Step 6
- The escalation contract (what counts as "hard block")
- The canonical checks (typecheck/lint/test commands) — both the per-subtask set the loop runs and the verification commands the user expects on the final state

This file is the persistence path. Re-read it after every few subtasks. If the plan is producing pain (e.g., a Mode C dependency classification turns out to be wrong), update the file and write down why.

## Step 8: Present and get approval

Show the operator:

1. The filter applied and the count it produced.
2. The chosen PR strategy.
3. The first 3–5 subtasks named explicitly, with their parent branch from the Step 6 table.
4. The pinned execution parameters as a one-line summary.
5. The plan-file path.
6. The first thing the orchestrator will do on approval.

Wait for explicit approval before creating any worktrees. The plan is a contract; do not exit plan mode unilaterally.

## What this phase produces

- A persisted plan file at `~/.claude/plans/<task-slug>.md`.
- A pinned stack order with branch + worktree + parent-branch for every subtask.
- The operator's go/no-go and any mid-flight corrections folded in.
- Conversation memory holding the ordered queue. **No TaskList. No shared state.**

When this phase is done, hand off to Phase B (`phase-b-execute-subtask.md`).
