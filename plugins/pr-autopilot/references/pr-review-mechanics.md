# PR Review Mechanics (shared)

This is the shared, **control-flow-free** mechanics layer for working with GitHub PR reviews, consumed
by the PR skills in this plugin.

It deliberately contains **no approval gates, no commit policy, and no loop control** — those belong
to each calling skill, because that's exactly what differs between them. This file answers only the
mechanical "how do I talk to GitHub's review API correctly" questions, which are identical everywhere
and easy to get subtly wrong. Keep that separation when editing: behavior policy lives in the skills;
API mechanics live here.

## 1. Discover the PR

```bash
gh pr view <number-or-url> --json number,url,headRefName,baseRefName,state,isDraft
```

Omit the argument to use the current branch's PR. If this fails, the branch has no open PR. Capture
`headRefName` — that's the branch any commits must go to.

## 2. Fetch all THREE comment sources

This is the trap: PR feedback arrives through three separate endpoints, and fetching only one
silently misses the rest. Always fetch all three and verify counts so an empty result is recognized
as "none" rather than "fetch failed."

**Inline review threads** (code comments, with resolution status) — GraphQL is the only way to get the
`isResolved` flag:

```bash
gh api graphql -f query='
  query($owner:String!,$repo:String!,$number:Int!){
    repository(owner:$owner,name:$repo){
      pullRequest(number:$number){
        reviewThreads(first:100){ nodes {
          id isResolved isOutdated
          comments(first:50){ nodes { id databaseId author{login} body path line } }
        }}
      }
    }
  }' -F owner=<owner> -F repo=<repo> -F number=<number>
```

The thread `id` (a node ID like `PRRT_...`) is what you resolve in §5. `databaseId` is the REST
comment id used for replies.

**Top-level review bodies** (a reviewer's overall verdict + summary text, and the review `state`):

```bash
gh api repos/{owner}/{repo}/pulls/{number}/reviews
```

`state` is one of `APPROVED`, `CHANGES_REQUESTED`, `COMMENTED`, `DISMISSED`. Missing this fetch is a
documented failure mode — a bot's "changes requested" verdict often lives only here, not inline.

**General PR conversation comments** (issue-level, not attached to code):

```bash
gh api repos/{owner}/{repo}/issues/{number}/comments
```

**Verify counts.** After fetching, note how many items each source returned. If you expected feedback
(e.g. a bot just reviewed) and a source returns zero, treat it as a possible fetch error and retry,
not as "nothing to do."

Record each reviewer's `login` (or `author.login`) **exactly as returned**, including any `[bot]`
suffix — you need the exact string to mention them in §6.

## 3. Determine what is unaddressed

A thread/comment is **handled** if any of these is true:
- The inline thread's `isResolved` is `true`.
- It has a reply from you containing `Addressed in` (the marker written in §5).
- It's older than, and already covered by, work you tracked in a previous round.

Everything else is a candidate for this pass.

## 4. Classify: actionable vs. not

Not every comment needs a code change. Read each and classify.

**Actionable — address these:**
- Requests for a specific code change ("use a constant here", "add a null check", "rename this").
- Bug reports or logic issues ("this fails when X is null", "off-by-one").
- Style/formatting requests ("add trailing comma", "camelCase this").
- Concrete recommendations to implement something from an automated tool.

**Not actionable — skip these:**
- Praise/acknowledgment ("Nice!", "LGTM", "Good catch").
- Open-ended questions ("Why this approach?").
- Opinions/discussion without a clear ask ("We might want to consider…").
- Informational notes ("FYI, this module was refactored last week").
- Bot summary/walkthrough headers (CodeRabbit walkthrough, review summary banners).

When a comment is ambiguous, the calling skill decides what to do with it (an interactive skill asks
the user; an autonomous skill makes a judgment call and reports it). Either way, tag each actionable
item with its reviewer and whether it is a **change request or a recommendation to implement** — §6
uses that to decide who gets re-pinged.

## 5. Reply and resolve

After the fix is committed (commit policy is the calling skill's decision), reply on each addressed
thread and resolve it.

**Reply format** — keep it exactly this so the `Addressed in` marker in §3 stays detectable:

```
Addressed in [`<short-hash>`](https://github.com/{owner}/{repo}/commit/<full-hash>)
```

**Reply to an inline review comment thread** (REST, using the thread's first comment `databaseId`):

```bash
gh api repos/{owner}/{repo}/pulls/{number}/comments/{comment_id}/replies \
  -f body='Addressed in [`abcdef0`](https://github.com/owner/repo/commit/abcdef0123456789)'
```

**Resolve the inline thread** (GraphQL — REST has no resolve op), using the thread node `id` from §2:

```bash
gh api graphql -f query='
  mutation($threadId:ID!){ resolveReviewThread(input:{threadId:$threadId}){ thread { isResolved } } }
' -F threadId=<thread-node-id>
```

**Top-level reviews and general comments** have no resolve operation — just reply. For a general
comment, reply in the PR conversation and link back to the original for context:

```bash
gh pr comment {number} --body 'Addressed [this comment](<comment-url>) in [`abcdef0`](https://github.com/owner/repo/commit/abcdef0123456789)'
```

## 6. Re-request review — the right bot, the right way

Re-request review **only** from reviewers whose feedback you actually addressed this round and whose
feedback was a **change request or a recommendation to implement** (a `CHANGES_REQUESTED` review, or
actionable inline comments you fixed). Do not ping a reviewer whose comments you skipped, who only
left praise/approval/informational notes, or who wasn't involved in this round's changes — that's
noise, and it trains reviewers (and bots) to ignore your pings.

You can combine the "Addressed in `<hash>`" reply and the re-review request into one comment on the
same thread.

Each reviewer listens differently — trigger them the way they actually respond:

| Reviewer (`login`)                                      | How to re-request                                                                                                                                              |
| ------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Claude / Claude Code Review (`claude[bot]`)             | Comment mentioning `@claude` and asking for a re-review. Claude's app answers to `@claude` with or without the `[bot]` suffix.                                  |
| CodeRabbit (`coderabbitai[bot]`)                        | Comment `@coderabbitai review` for an incremental re-review, or `@coderabbitai full review` for a fresh full pass.                                              |
| GitHub Copilot (`copilot-pull-request-reviewer[bot]`)   | Re-request via the API, **not** a mention: `gh api -X POST repos/{owner}/{repo}/pulls/{number}/requested_reviewers -f 'reviewers[]=copilot-pull-request-reviewer[bot]'` (or the GitHub MCP `request_copilot_review` tool). |
| Any other bot (`*[bot]`)                                | Reply on its thread mentioning it by its **exact** `login`, including the `[bot]` suffix, e.g. `@review-bot[bot] please re-review`. Without `[bot]` the mention won't resolve and the bot won't fire. |
| Human reviewer                                          | Re-request via `gh pr edit {number} --add-reviewer <login>` (or the requested_reviewers API). Humans respond on their own schedule.                             |

**Why the `[bot]` suffix matters:** GitHub names bot accounts with a `[bot]` suffix. To @-mention one
you must include it, or the mention resolves to a non-existent/ wrong account and the bot is never
notified. The only exceptions are apps known to accept the bare handle (Claude → `@claude`). When in
doubt, use the exact `login` you captured in §2.
