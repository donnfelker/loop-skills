# Prompt Templates

Copy-paste skeletons for the agents this skill dispatches **outside the per-subtask loop**. Replace `<angle-brackets>` with concrete values. Treat these as starting points — adapt to the specific PR or parent.

> **The Dev / QA / Reviewer / combined-single-shot skeletons live in the `dev-team` skill** (`references/prompt-templates.md` there). Phase B calls that loop per subtask, so those role prompts are owned by it. This file holds only the templates that are specific to driving a stack of PRs: the Phase C per-PR review-response agent and the Phase A operator kickoff.

These templates are distilled from real multi-PR orchestration runs. The phrases marked **important** are the ones that demonstrably change agent behavior in production.

## Table of contents

- [Phase C single-PR agent](#phase-c-single-pr-agent-for-prs-with-multiple-comments-to-address)
- [Phase A kickoff prompt](#phase-a-kickoff-prompt-for-the-orchestrator-to-surface-to-the-operator)
- [Adapting the templates](#adapting-the-templates)

Phase files reference these anchors directly — e.g., `prompt-templates.md#phase-c-single-pr-agent-for-prs-with-multiple-comments-to-address`.

---

## Phase C single-PR agent (for PRs with multiple comments to address)

```
You are addressing review comments on PR #<N> for <REPO>. One-shot agent.

## Workspace
- **Worktree**: <absolute path>
- **Branch**: <branch> (already rebased on its parent's current tip)
- Always prefix Bash with `cd <worktree> && …`.
- Do NOT touch <operator's primary checkout>.

## Unresolved feedback to address

### Inline review threads
<for each unresolved thread, paste:
- Thread graphql id (for resolveReviewThread mutation)
- First-comment databaseId (for /comments/{id}/replies)
- File:line, author, full body
>

### Top-level review bodies
<for each non-empty review body, paste:
- review id
- author
- state (APPROVED / COMMENTED / CHANGES_REQUESTED)
- full body
>

### General PR comments
<for each non-bot-noise comment, paste:
- comment id
- author
- full body
>

## Classification rules
<per the pr-review-mechanics decision matrix — actionable vs informational>
Spell out which items are actionable. Skip the rest.

## What to do

1. For each actionable item, read the file and make the change.
2. Add or update tests if applicable.
3. Run: <canonical checks>
4. Commit (default: NEW commit, no amend) with subject ≤72 chars:

   <conventional commit message template>

5. Push: `git push origin <branch>` (plain push).

6. For each addressed item, post a reply with the addressing commit:

   **Inline thread** (has resolve op):
   ```
   gh api repos/OWNER/REPO/pulls/<N>/comments/<first-comment-databaseId>/replies \
     -f body="Addressed in [\`<short>\`](https://github.com/OWNER/REPO/commit/<full>)"
   gh api graphql -f query='mutation($threadId: ID!) {
     resolveReviewThread(input: {threadId: $threadId}) { thread { isResolved } }
   }' -F threadId='<thread.id>'
   ```

   **Top-level review** (no resolve op):
   ```
   gh pr comment <N> --body "Addressed [@<reviewer>'s review](https://github.com/OWNER/REPO/pull/<N>#pullrequestreview-<review-id>) in [\`<short>\`](https://github.com/OWNER/REPO/commit/<full>):
   - **Item 1:** <change>.
   - **Item 2:** <change>.

   @<reviewer-bot-handle> re-review"
   ```

   **General comment** (no resolve op): same as top-level review reply, referencing the comment URL.

## Return format
- Each addressed item: <which source, what changed>
- Commit SHA
- Reply URLs
- Thread resolution statuses (for inline threads)
- Test output summary

## Constraints
- Worktree only.
- NO --no-verify. NEW commit, no amend (unless explicitly told to amend for a commit-message-subject-too-long item).
- Don't open PRs, don't merge, don't touch other branches.
```

---

## Phase A "kickoff" prompt (for the orchestrator to surface to the operator)

After Phase A is drafted but before exiting plan mode, present this back to the operator for sign-off:

```
## Plan: <PARENT-TICKET-ID> → <N>-PR stack

**Source**: <URL to the parent>
**Filter**: <criteria — e.g., P1/P2/P3 subtasks, status=open>
**Subtask count**: <N>

## Stack order

| # | Subtask | Branch | Worktree | Stack-parent |
|---|---|---|---|---|
| 1 | <ID> — <title> | <branch> | ~/.claude-worktrees/<slug>/ | main |
| 2 | <ID> — <title> | <branch> | ~/.claude-worktrees/<slug>/ | <prev branch> |
| ... |

## Execution parameters

- **PR strategy**: one per subtask, stacked.
- **Branching**: one branch per subtask, off the prior subtask's tip.
- **Worktrees**: yes, one per subtask.
- **Concurrency**: strict serial (the stack enforces it).
- **Cycle cap**: 3 Dev → QA → Reviewer cycles, then escalate.
- **Tier boundaries**: <continue without stopping | pause for go/no-go>.
- **Stop condition**: <only on hard block | end of each tier>.

## What I'll do when you approve

1. Create the worktree for subtask #1 off `main`.
2. Mark subtask #1 status as in-progress.
3. Run the dev-team on subtask #1 (Dev → QA → Reviewer → commit).
4. Open PR #1 with base = main.
5. Advance to subtask #2 (parent = subtask #1's branch).
6. Continue through subtask #<N>.
7. Hand off to Phase C (review-response sweep) when all PRs are open.

Plan saved to ~/.claude/plans/<task-slug>.md.

OK to proceed?
```

Don't exit plan mode until the operator approves.

---

## Adapting the templates

In practice, these templates get modified in flight. Common adaptations:

1. **Drop the "tier boundary pause"** — when the operator says "don't stop until done", do a fast verification at each tier boundary and continue.
2. **Embed the Phase C reply templates into the loop's agents** when a single agent is also doing Phase C work — reduces round-trips and lets the agent commit + reply in one shot.
3. **Trim the unresolved-feedback block** to only the items that are actually actionable before handing the Phase C agent the prompt — informational bot summaries add noise.

The skeletons are a starting point, not a contract. For the Dev / QA / Reviewer adaptations, see the `dev-team` skill's own template notes.
