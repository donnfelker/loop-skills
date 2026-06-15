---
name: pr-autopilot
description: "Autonomously watch a single pull request and drive its review feedback to completion without supervision: each round fetch every unaddressed review and comment (Claude, CodeRabbit, Copilot, other bots, humans), address the actionable ones, commit, push, reply/resolve threads, and re-request review from the bots whose change requests it addressed, then loop until a quiet round or a safety cap and print a summary. Use whenever the user wants to 'watch a PR', 'babysit a PR', 'keep addressing PR comments until they are done', 'auto-respond to reviews', 'handle the review back-and-forth on PR #N', or run a hands-off review-response loop on ONE PR — even if they don't name this skill. Pass --single-pass (or --max-rounds 1) for a single autonomous round with no loop, a one-shot comment sweep. For driving MANY stacked PRs from a parent ticket, see /implement-full-spec."
user_invocable: true
---

# PR Autopilot

Drive a single pull request's review feedback to completion without supervision. It runs round after
round on its own — committing, pushing, re-requesting review, and waiting for the next batch of
feedback — until the reviews settle down. For a one-shot sweep instead of a loop, run it with
`--single-pass`: a single autonomous round, then stop.

**Read `${CLAUDE_PLUGIN_ROOT}/references/pr-review-mechanics.md` first** (if that path doesn't
resolve, it's at the plugin root, two levels up from this skill:
`../../references/pr-review-mechanics.md`). That file is the shared "how to talk to the GitHub review
API correctly" layer — fetching all three comment sources, classifying, replying, resolving threads,
and the per-bot re-review trigger table. The steps below are this skill's *control flow*: when to
loop, when to commit, and when to stop.

## Operating mode (read this first)

This skill runs **fully autonomously** — that's the whole point:

- **No plan-approval gate.** Still *form* a plan internally so your changes stay coherent, but don't
  stop for sign-off — the user has opted into unattended operation.
- **You DO commit and push** each round. Otherwise the loop can't make progress.
- **`--single-pass` mode** runs exactly one autonomous round (equivalent to `--max-rounds 1`) and then
  stops without waiting for the next batch — a one-shot "address everything outstanding right now"
  sweep, still with no approval gate. Everything else below is identical; the loop just doesn't
  re-enter.

Pushing commits and posting comments are outward actions, so keep them safe and faithful: only ever
push to the PR's own head branch, never force-push, and never post an "Addressed in `<hash>`" reply or
resolve a thread for a change you didn't actually make. The user can interrupt the loop at any time.

## Step 0 — Identify the PR and set the caps

Apply **§1** of the reference to resolve the PR (from a number/URL the user gave, or the current
branch). If there's no open PR, or it's already merged/closed, say so and stop.

Make sure your local checkout is on the PR's `headRefName` — you'll commit to it. If you're on a
different branch, check it out first.

Set the safety caps. The loop stops on a **quiet round OR** when a cap is hit, whichever comes first:

- `--max-rounds N` — default **6**. A "round" is one address→commit→push→re-request cycle.
- `--time-cap MIN` — default **60** minutes of wall-clock from when watching started.
- `--single-pass` — shorthand for `--max-rounds 1`: do one autonomous round, then finish and report
  (Step 6) without waiting for the next batch of reviews.

Tell the user, in one line, what you're watching and the caps, e.g.:
`Watching PR #123 (feature/foo). Stops on a quiet round, or after 6 rounds / 60 min.`
In single-pass mode: `Addressing PR #123 (feature/foo) once, then stopping.`

## Step 1 — Poll: fetch and triage

Apply **§2 (fetch all three sources, verify counts, capture exact reviewer logins + review states)**
and **§3 (determine what's unaddressed)**. Track across rounds the comment/review IDs you've already
handled this session, on top of what's recorded on GitHub itself (resolved threads, "Addressed in
`<hash>`" replies).

Then apply **§4 (classify)**. For ambiguous comments you don't have a user to ask, make a judgment
call and note it in the final report rather than blocking. Tag each actionable item with its reviewer
and whether it's a **change request / recommendation to implement** — you need that in Step 4.

### Human reviewers

Address actionable human comments the same as bots — fix, commit, reply, and re-request their review
(§6). **But do not let the loop wait on humans.** People respond on their own schedule, so the loop's
continue/stop decision (Step 2) is based only on bot activity — you never spin indefinitely waiting
for a person. Surface outstanding human items in the final report instead.

## Step 2 — Decide: is this a quiet round?

If there are **no new actionable comments** and **no outstanding `CHANGES_REQUESTED` review from any
bot**, this is a **quiet round** → go to Step 5 (finish and report). Otherwise continue.

One more condition before calling it quiet: **each bot's most recent verdict must target the current
HEAD commit, and no review run may still be in flight.** "No new comments at poll time" is not the
same as "the bot has signed off on this HEAD" — review bots take a couple of minutes after a push or
re-ping, and a new finding can land seconds after your poll. If you just pushed or just re-pinged a
bot, treat its review as pending and poll again rather than concluding on a verdict for an older
commit. When a bot's review runs as a CI job (e.g. a `claude-review` workflow), an in-progress check
run on the PR is a reliable pending signal.

## Step 3 — Address, commit, push

Make the changes (read context, fix, group items touching the same file, run the project's
linter/formatter if it has one). No approval gate — just do it carefully.

Then commit and push:

- Commit with a message following the **repo's existing convention** (check recent `git log`; many
  repos use Conventional Commits). A clear default: `fix: address PR review feedback (round <N>)`.
  Include a `Co-Authored-By` trailer if this environment requires one. Stage files by name; don't
  `git add -A` unrelated leftovers.
- `git push` to the PR head branch. **Never** `--force`.
- Capture the pushed commit's full hash + URL for the replies.

## Step 4 — Reply, resolve, and re-request

Apply **§5 (reply `Addressed in <hash>` and resolve inline threads)** and **§6 (re-request review)**.
Per §6, re-ping a reviewer **only** when you addressed a change request / recommendation from them
this round, and use each bot's correct trigger (`@claude`, `@coderabbitai review`, Copilot via API,
exact `[bot]` mention for others). Combine the "Addressed" reply and the re-review ping into one
comment when you can.

## Step 5 — Wait for the next batch, then loop

**If running `--single-pass` (`--max-rounds 1`): skip this step entirely.** The single round is done —
go straight to Step 6 and report, without scheduling a wake-up or waiting for the next batch.

After pushing and pinging, bots take time to come back (CodeRabbit and Claude usually within a few
minutes; Copilot similar). Keep watching across that gap using the **`/loop` self-pacing mechanism** —
schedule a wake-up to re-enter Step 1 rather than blocking:

- If the user launched you under `/loop <interval> ...`, complete one round per invocation and let
  `/loop` re-fire you — return to Step 1 on the next firing.
- Otherwise self-pace: schedule the next poll with a short delay (start around **3 minutes** so fast
  bots have time to respond), and back off if a round comes back quiet-but-pending (e.g. 3 → 5 → 8
  minutes) so you're not hammering the API. The wake-up re-enters this skill at Step 1 for the same
  PR.

Each re-entry, increment the round counter and re-check the caps. **Stop immediately and go to Step 5
[finish]** if `--max-rounds` is exceeded or `--time-cap` elapses, even mid-wait.

## Step 6 — Finish and print results

Stop the loop and print a clear summary. Lead with **why** you stopped (quiet round vs. hit a cap):

```
## PR #123 review watch — complete

Stopped: quiet round (no new actionable comments, no outstanding change requests)
Rounds run: 3 of 6 max · elapsed 14 min

### Addressed this session
- coderabbitai[bot]: 4 comments → fixed in a1b2c3d, e4f5g6h (re-reviewed, now passing)
- claude[bot]: 2 comments → fixed in a1b2c3d (re-review requested, approved)
- @jsmith: 1 comment → fixed in e4f5g6h (re-review requested — awaiting human response)

### Still outstanding
- @jsmith: 1 question ("why this approach?") — left for you, not a change request

### Current review state
- claude[bot]: APPROVED  ·  coderabbitai[bot]: no outstanding comments
- copilot: re-review pending  ·  human reviewers: 1 pending (jsmith)

PR: https://github.com/owner/repo/pull/123
```

If you stopped on a cap, say so plainly and list what's still unaddressed so the user can decide
whether to re-run the watch or step in.

## Guardrails

- **Branch safety:** only ever commit/push to the PR's head branch; never force-push; never touch the
  base/default branch.
- **No false resolutions:** only reply "Addressed in `<hash>`" / resolve a thread for a change you
  actually made and pushed. If you chose not to act on a comment, leave it open and report it.
- **Termination is guaranteed by design:** the quiet-round check plus the round/time caps are what
  stop the loop. If you can't make progress on an item (a comment you don't understand, a fix that
  breaks tests you can't repair), stop, report it, and let the user decide rather than churning.
- **Respect reviewer attention:** re-ping a bot only for change requests/recommendations you actually
  addressed this round.
