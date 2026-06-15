# Phase C — Respond to PR Feedback

This is the multi-round sweep that takes the open PRs from Phase B and drives them to merge-ready by addressing every comment from every source on every PR, then asking the reviewer bot to re-review.

It is the most error-prone phase. The most common failure on the first sweep is that the comment-fetch only queries inline review threads, missing top-level PR review bodies entirely — roughly a third of actionable feedback gets silently skipped. **Read the "Three sources of feedback" section before writing any fetch code.**

## How this phase differs by Phase A mode

The fetch / classify / address / reply machinery is the same in all modes. What differs is the **set of PRs to iterate over** and **whether updates cascade**:

| Mode | PRs to sweep | Cascade after each update? |
|---|---|---|
| **A — Single PR** | One PR (opened at the end of Phase B, or as a draft from the start). | No — no downstream to rebase. |
| **B — Stacked PRs** | All PRs in the stack, oldest first. | Yes — every PR downstream of the updated one needs `git rebase --onto` + force-push. |
| **C — Independent PRs with selective stacking** | All PRs. | Only along dependency chains. Independent PRs that share no parent need no rebase when a sibling is updated. |

Steps 1–10 below apply universally. Step 11 (cascade) is gated by mode: skip it for Mode A, run it always for Mode B, run it only for downstream-of-dependency in Mode C.

## Three sources of feedback (this is the trap)

GitHub stores PR feedback in three places. Each has a separate API endpoint. **You must fetch all three or you will miss feedback.**

| Source | What it is | Where to fetch | Resolve-able? |
|---|---|---|---|
| **Inline review threads** | Comments anchored to a specific file:line, created inside a review | `pullRequest.reviewThreads` (GraphQL) | Yes — `resolveReviewThread` mutation |
| **Top-level review bodies** | The text the reviewer writes in the review form before submitting "Approve / Comment / Request changes" | `gh api repos/{owner}/{repo}/pulls/{n}/reviews` | No — no resolve op |
| **General issue comments** | Comments posted directly on the PR conversation timeline (e.g., bot summary posts, "@somebody re-review" pings) | `gh api repos/{owner}/{repo}/issues/{n}/comments` | No — no resolve op |

A `claude[bot]` "Code review" post is usually a general comment OR a top-level review body. A `proj-inspect[bot]` "CHANGES_REQUESTED" review is a top-level review body with no inline threads. Either way: if you only check `reviewThreads`, you miss it.

## Sweep order

For each PR in the stack (oldest first, i.e., the one closest to `main`):

```
1. Triage   — fetch all three sources; classify each item.
2. Rebase   — bring this PR's branch onto its parent's current tip.
3. Address  — make code/doc fixes for actionable items.
4. Verify   — typecheck, lint, tests.
5. Commit   — new commit by default; amend only for bot-flagged commit-message issues.
6. Push     — plain push if new commit; --force-with-lease if amend/rebase.
7. Reply    — on every actionable item, with "Addressed in [<short>](<url>)".
8. Resolve  — inline threads only.
9. Re-ping  — "@<bot> re-review" so the bot reruns.
10. Cascade — rebase every downstream PR onto this PR's new tip; force-push each.
```

After every PR is processed, **do not stop**. Bot re-reviews often surface new items. Loop until a sweep produces zero actionable items.

## Step 1: Triage — fetch all three sources

### 1a. Inline review threads (GraphQL)

```bash
gh api graphql -f query='query($num: Int!) {
  repository(owner: "OWNER", name: "REPO") {
    pullRequest(number: $num) {
      reviewThreads(first: 100) {
        nodes {
          id            # GraphQL node id, needed for resolveReviewThread
          isResolved
          isOutdated
          path
          line
          comments(first: 20) {
            nodes {
              databaseId  # REST id, needed for /comments/{id}/replies
              author { login }
              body
              url
            }
          }
        }
      }
    }
  }
}' -F num=42
```

Filter `isResolved == false`. For each thread, capture the GraphQL `id` (for the resolve mutation) and the FIRST comment's `databaseId` (for posting the reply).

### 1b. Top-level review bodies (REST)

```bash
gh api repos/OWNER/REPO/pulls/42/reviews \
  --jq '.[] | select(.body != "" and .body != null) |
  "── \(.user.login) — \(.state) id=\(.id) ──\n\(.body)\n"'
```

`.state` is `APPROVED` / `COMMENTED` / `CHANGES_REQUESTED`. **All three states can have actionable bodies.** Don't skip `APPROVED` — bots happily approve while listing 5 nits in the body.

### 1c. General issue comments (REST)

```bash
gh api repos/OWNER/REPO/issues/42/comments \
  --jq '.[] | "[\(.user.login)] id=\(.id) url=\(.html_url)\n\(.body)\n"'
```

### 1d. Avoid the jq control-char gotcha

Comment bodies contain newlines, tabs, code fences, and other control characters that break `jq` when piped through bash. **Use `--jq` directly on the `gh api` command** (above), or pipe to a file and read it. Do NOT do `gh api ... | jq ...` for review bodies — it will fail on many comments and they'll appear empty when they aren't.

### 1e. Verify all three sources were fetched

After running 1a, 1b, and 1c, print counts so the all-zero state is visible — silent-empty is the failure mode this whole section guards against:

```bash
PR=42
OWNER=...; REPO=...

INLINE=$(gh api graphql -f query='query($n:Int!){repository(owner:"'"$OWNER"'",name:"'"$REPO"'"){pullRequest(number:$n){reviewThreads(first:100){nodes{isResolved}}}}}' -F n=$PR \
  --jq '[.data.repository.pullRequest.reviewThreads.nodes[] | select(.isResolved==false)] | length')
TOPLEVEL=$(gh api repos/$OWNER/$REPO/pulls/$PR/reviews \
  --jq '[.[] | select(.body != "" and .body != null)] | length')
GENERAL=$(gh api repos/$OWNER/$REPO/issues/$PR/comments --jq 'length')

echo "PR #$PR: inline=$INLINE  top-level-reviews=$TOPLEVEL  general-comments=$GENERAL"
```

If all three are zero, double-check — a PR with **any** review usually has at least one general comment (the bot's posted summary, the CI status comment, etc.). All-zero is suspicious; investigate before declaring the PR clean. If you have ground-truth from the PR UI showing comments exist, the fetch is wrong.

**Sources to consult when something looks off:**
- `gh pr view <N> --json comments,reviewDecision` — gross-level check that the PR has feedback at all.
- The PR's web UI — final source of truth when scripted fetches disagree with what's visible.

## Step 2: Classify per the pr-review-mechanics decision matrix

For each item from all three sources:

**Actionable — address it:**
- Concrete code-change request ("use a constant here", "add null check", "rename this")
- Bug report or logic issue ("this throws on dot-containing emails")
- Real CLAUDE.md / style violation ("comment restates the literal `90 days`")
- Suggestion from automated tools (CodeRabbit, Claude Code Review, etc.) requesting a concrete change

**Not actionable — skip:**
- Praise ("LGTM", "Nice!")
- Open-ended questions ("Why did you choose this approach?")
- Informational bot summaries ("No issues found. Checked for bugs and CLAUDE.md compliance.")
- Walkthrough headers from CodeRabbit-style summary comments

**Borderline — note in the plan and ask:**
- "Commit subject too long" — fix is amend + force-push, which cascades the stack. Cheap to do during the existing force-push but ask first.
- "Pre-existing duplication" — observation, not introduced by this PR. Optional follow-up.
- "Test coverage gap" — usually worth doing if small; defer if it requires re-architecting the test harness.

Don't act on praise. Don't reply to bot "No issues found" summaries. Both create noise.

## Step 3: Rebase this PR onto its parent's current tip

If the parent PR has been updated (force-pushed) during a prior sweep, the local branch is behind. Rebase before making any new commits.

**Use this mechanical recipe — do NOT eyeball SHAs from `git log`:**

```bash
cd <worktree>

# Capture the OLD parent tip BEFORE fetching. This is the SHA your local
# branch's stack-parent edge currently points at, which is exactly the
# upstream argument git rebase needs.
OLD_PARENT_TIP=$(git rev-parse origin/<parent-branch>)

git fetch origin -q

# Now origin/<parent-branch> points at the NEW tip.
git rebase --onto "origin/<parent-branch>" "$OLD_PARENT_TIP"
```

Why this form: `git rebase --onto NEW OLD` replays the commits in `OLD..HEAD` onto `NEW`. By capturing `origin/<parent-branch>` *before* the fetch, `$OLD_PARENT_TIP` is exactly the boundary that separates your local subtask commits from the parent's old tip — no visual SHA-picking from `git log`, no off-by-one as your branch accumulates more commits across multiple Phase C rounds.

**The naive forms that break:**

| Wrong form | What goes wrong |
|---|---|
| `git rebase origin/<parent-branch>` | After parent force-pushed, this tries to re-apply commits that are conceptually already in the new base (at different SHAs), producing duplicate-commit conflicts. |
| `git rebase --onto origin/<parent-branch> <SHA from git log>` | Works once. On round 2+ of Phase C, the agent picks the wrong SHA because the branch now has multiple subtask commits and the visually-identified "first commit" is no longer the actual boundary. |

If for some reason you can't capture the OLD tip first (e.g., the worktree was just opened and there's no prior tracking ref), the fallback is `git rebase --fork-point --onto origin/<parent-branch> origin/<parent-branch>` which uses the reflog to find the right fork point. Less robust than capturing the SHA, but mechanical enough to avoid visual SHA-picking.

See `gotchas.md` gotcha #3 for why this recipe matters.

## Step 4: Address actionable items

For each actionable item, edit the file(s) to make the change the comment asked for. Group related changes from the same PR into one commit when possible.

For Phase C the choice between "do it myself inline" and "dispatch an Agent" is volume-based:

| Volume per PR | Pattern |
|---|---|
| 1–2 trivial edits (e.g., comment-only changes) | Inline via Edit tool |
| 3+ items, or any item that requires test additions | One-shot Agent call with the comment specs and the resolve pattern baked in |
| Architectural fix from a review | Run the `dev-team` skill on the review's spec, just like Phase B — then push and reply |

**`dev-team` commits but does not push.** It stops at an APPROVED & COMMITTED SHA; pushing the branch, posting the `Addressed in <SHA>` reply, resolving threads, and the re-review ping all stay with you — i.e., the push step onward in this sweep. Don't post the reply until you've pushed the SHA `dev-team` returned — otherwise the comment links a commit the forge doesn't have yet.

When dispatching an Agent for Phase C, pre-supply: the unresolved thread / review bodies (verbatim), the resolve template, and the reply template. See [`prompt-templates.md#phase-c-single-pr-agent-for-prs-with-multiple-comments-to-address`](prompt-templates.md#phase-c-single-pr-agent-for-prs-with-multiple-comments-to-address) for a ready-to-fill prompt.

## Step 5: Verify

Run the canonical checks before committing. Same commands as Phase B. Don't commit if these fail — fix the failure first.

## Step 6: Commit

**Default: a new commit on top of the existing branch.** Subject ≤72 chars, conventional-commits, body referencing the PR and the review:

```
docs(security): reference WEBHOOK_KEY_DEFAULT_TTL_DAYS in comments

Addresses proj-inspect review on PR #36: literal "90 days" restated
the WEBHOOK_KEY_DEFAULT_TTL_DAYS constant value. Updated comments
in types.ts, automations.ts, and the SQL migration.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
```

**Exception: amend the existing commit** when the bot's complaint is specifically about the original commit's subject or body (e.g., "subject exceeds 72-char limit", "commit body contains implementation details", "remove this debug line you accidentally committed"). Use `git rebase -i HEAD~N` with `reword` to change just that commit. Then force-push. Then also update the PR title if it matched the old subject (`gh pr edit <n> --title "..."`).

**Before reaching for `git rebase -i`, check the repo's merge strategy.** If the repo uses **squash-merge** (the most common GitHub setting), the in-branch commit subjects and bodies are discarded on merge — only the PR title and the merge commit's generated body land on `main`. In that case, `gh pr edit <n> --title "<new>"` and `gh pr edit <n> --body "<new>"` fully address the bot's complaint without rewriting history. That's strictly better than `git rebase -i` because it avoids the force-push cascade through the rest of the stack.

Use `git rebase -i` only when the repo uses **merge-commit** or **rebase-merge** strategy (every in-branch commit lands on `main` as-is) AND the bot is complaining about something that will actually be visible on `main`. See gotcha #16 for the squash-merge mechanics.

## Step 7: Push

- New commit (default): `git push origin <branch>` (plain push).
- Amend or rebase happened: `git push --force-with-lease origin <branch>`.

Always `--force-with-lease`, never `--force`. The `--with-lease` form refuses the push if someone else has updated the remote since your last fetch, which catches the "I forgot I had a checkpoint" footgun.

## Step 8: Reply on every addressed item

The format is consistent across all three sources:

```
Addressed in [`<short-sha>`](https://github.com/OWNER/REPO/commit/<full-sha>)
```

Where `<short-sha>` is the first 7 characters.

### 8a. Inline review thread reply

```bash
gh api repos/OWNER/REPO/pulls/{N}/comments/{first-comment-databaseId}/replies \
  -f body="Addressed in [\`<short>\`](https://github.com/OWNER/REPO/commit/<full>)"
```

### 8b. Top-level review reply

Top-level reviews don't have a reply endpoint. Post a general PR comment that links back to the review and to the addressing commit:

```bash
gh pr comment <N> --body "Addressed [@reviewer's review](https://github.com/OWNER/REPO/pull/<N>#pullrequestreview-<review-id>) in [\`<short>\`](https://github.com/OWNER/REPO/commit/<full>):
- **Item 1 (brief summary):** <what changed>.
- **Item 2:** <what changed>.

The other observations (X, Y) were not actionable / already addressed earlier."
```

### 8c. General issue comment reply

Same as 8b — a new general comment that references the original by URL and links the addressing commit.

## Step 9: Resolve inline threads

Inline review threads have a resolve op; top-level reviews and general comments do not.

```bash
gh api graphql -f query='mutation($threadId: ID!) {
  resolveReviewThread(input: {threadId: $threadId}) {
    thread { isResolved }
  }
}' -F threadId='<thread.id from step 1a>'
```

After this returns `isResolved: true`, the thread is closed and won't show up in the next triage pass.

## Step 10: Re-request review

After every push that addresses a reviewer's feedback, ping the reviewer. **Whoever left the comment or review is the entity that needs to be re-pinged** — bot OR human. The general form:

```bash
gh pr comment <N> --body "@<reviewer-handle> re-review"
```

### Identifying the right handle

For each addressed item, the handle to re-ping is the `author.login` (or `user.login`) field from the original comment / review / thread. Capture it during the Step 1 triage so it's available here without re-fetching.

### Bot-specific trigger phrases

Different review bots accept different re-trigger phrases. Use the documented one when known; fall back to a plain `re-review` mention when not. Common forms seen in the wild:

| Reviewer handle | Trigger phrase | Notes |
|---|---|---|
| `claude[bot]` (Claude Code Review GHA, official) | `@claude review` | Re-runs the Anthropic GHA review on the latest commit. |
| `coderabbitai` / `coderabbitai[bot]` | `@coderabbitai full review` or `@coderabbitai review` | `full review` re-scans everything; `review` does an incremental pass. |
| `copilot-pull-request-reviewer[bot]` | `@copilot review` | GitHub Copilot's PR-review bot. |
| `gemini-code-assist[bot]` | `@gemini-code-assist review` | Google's Gemini PR-review bot. |
| `sweep-ai[bot]` | `@sweep review` | Sweep AI. |
| `qodo-merge-pro[bot]` / `pr-agent[bot]` | `/review` | Slash-command style, posted as a regular comment. |
| Internal / custom org bots (e.g., `proj-inspect[bot]`) | Usually `@<bot-handle> re-review` | Confirm by reading the bot's documentation, its README, or its prior posts on the same PR. |
| Human reviewer | `@<github-handle> when you have a chance, ready for another look` | No automated re-trigger; the ping just notifies them. |

**When the trigger phrase isn't documented:** check the bot's prior comments on this or any other PR — bots typically describe their trigger commands in their first post on a PR ("Comment `@<bot> review` to re-run me"). If still unknown, `@<bot-handle> re-review` is the safe default — most bots interpret `re-review` as a re-trigger; ones that don't will simply ignore the mention.

### Include the ping with the addressing reply

Include the re-ping in the reply comment posted in step 8, or as a separate comment immediately after — either works. The important thing is don't skip it. A reviewer that never gets pinged about the response will assume their feedback wasn't acted on.

For PRs with multiple distinct reviewers, ping each one separately based on whose feedback was addressed. Don't blanket-ping every reviewer on every PR — only the ones whose items received fixes this round.

## Step 11: Cascade the rebase

Now that this PR's branch has been updated (whether plain push or force-push), every downstream PR's branch is potentially behind.

For each PR `M` immediately downstream:

```bash
cd <worktree-for-M>
git fetch origin -q
git rebase --onto origin/<this-PR-branch> <local-old-upstream-sha-for-M>
# Resolve conflicts if any.
git push --force-with-lease origin <M's branch>
```

If conflicts arise, this is where evolving-API hazards bite (the audit's HMAC-token format changed shape across 5 PRs). `gotchas.md` has the worked example.

**You don't need to address comments on PR M as part of this cascade** — that's M's turn in the sweep loop. The cascade is purely "keep M based on the updated parent."

## Multi-round response

After one sweep, every PR has fresh commits and re-review pings. Bots usually respond within seconds to minutes. New review bodies appear.

Re-run the sweep. Round 2 catches:

- New items the bot surfaced in response to the re-review.
- Items the round-1 fix didn't fully address (the bot was right, you weren't done).
- Cascading effects: fixing PR N may have introduced a new issue in PR N+1 (rare, but seen in practice with auth-test fixtures).

A sweep with zero new actionable items is your termination condition. Tell the operator the stack is merge-ready.

## What this phase produces

- Every PR has zero unresolved review threads (verified via the same GraphQL query from step 1a, with `isResolved == false` returning length 0).
- Every actionable comment has a reply linking to its addressing commit.
- Every reviewer has been re-pinged at least once after their feedback was addressed.
- The stack is cleanly rebased and pushed.
- The orchestrator's memory captures any deferred items the operator should know about.

## Cross-references

- `gotchas.md` — three-source fetch trap, jq control chars, rebase --onto pattern, force-push-with-lease semantics, evolving-API conflict patterns
- `prompt-templates.md` — Phase C agent skeleton when you delegate the per-PR work
- The `pr-autopilot` skill (plugin: `pr-autopilot`) is the single-PR version of this phase. If the operator only has one PR to handle, that skill is the better fit — `/pr-autopilot --single-pass` covers fetch → fix → commit → push → reply → resolve for a single PR without the cascade-rebase machinery (drop the flag to loop it until reviews settle). Use it directly rather than invoking this whole skill for one PR.
