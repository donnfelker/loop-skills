# Gotchas — Hard-won Lessons

Each entry is a real production failure that the skill is designed to prevent. Read this file fully on first invocation of the skill; skim it later.

## 1. Three sources of PR feedback — miss one, miss the work

**The failure**: in a review-response sweep, the initial triage script fetched `pullRequest.reviewThreads` and `issues/{n}/comments` but **not** `pulls/{n}/reviews`. The latter is where bots post top-level review bodies (the "Code review" summary with state `CHANGES_REQUESTED` / `APPROVED` / `COMMENTED`). The operator caught the miss when they looked at a PR themselves and saw an `.env.example` gap the bot had flagged in its review body.

Result: ~30% of actionable feedback was silently skipped on the first sweep.

**The fix**: every triage MUST fetch all three:

```bash
# inline threads
gh api graphql -f query='{ ... reviewThreads { nodes { id isResolved ... } } }' -F num=N

# top-level review bodies — THIS ONE IS EASY TO FORGET
gh api repos/OWNER/REPO/pulls/N/reviews

# general PR comments
gh api repos/OWNER/REPO/issues/N/comments
```

Even APPROVED reviews can have actionable nit lists in the body. Don't skip them based on state.

## 2. The jq + control-character trap

**The failure**: piping `gh api` JSON into `jq` via shell pipes (`gh api ... | jq ...`) fails when comment bodies contain newlines, tabs, code fences, or other control characters. The error is a confusing `jq: parse error: Invalid string: control characters from U+0000 through U+001F must be escaped` which makes it look like the API returned bad JSON. The API returned fine JSON; the shell pipe is corrupting it.

**The fix**: use `gh api`'s built-in `--jq` flag, which processes the JSON in-process and handles the control chars correctly:

```bash
# BAD — fails on bodies with newlines
gh api repos/OWNER/REPO/pulls/42/reviews | jq '.[].body'

# GOOD
gh api repos/OWNER/REPO/pulls/42/reviews --jq '.[].body'

# ALSO GOOD — write to a file first, then process
gh api repos/OWNER/REPO/pulls/42/reviews > /tmp/reviews.json
jq '.[].body' /tmp/reviews.json
```

## 3. Cascade-rebase: capture the old parent tip BEFORE fetching

**The failure**: when PR N's parent (PR N-1) gets force-pushed after a review-response commit, PR N's branch is now stacked on a SHA that no longer exists at the tip of the parent branch. Running `git rebase origin/<parent-branch>` will try to re-apply commits that conceptually are already in the new parent (just at different SHAs), producing duplicate-commit conflicts that look bewildering ("why is git trying to apply the PROJ-004 commit again? it's already in the base!").

The natural recovery — "find the SHA my branch was based on by reading `git log`" — has a subtle failure mode of its own: on round 2+ of Phase C the branch has multiple subtask commits, the "first commit" is no longer at `HEAD~1`, and the visually-identified boundary is wrong. Picking the wrong SHA silently mangles the diff.

**The mechanical fix**: capture the parent tip BEFORE the fetch. The `origin/<parent-branch>` ref before fetching is, by construction, exactly the upstream boundary `git rebase` needs:

```bash
cd <worktree>

# Snapshot the current local view of the parent before fetch updates it.
OLD_PARENT_TIP=$(git rev-parse origin/<parent-branch>)

git fetch origin -q

# origin/<parent-branch> now points at the new tip.
git rebase --onto "origin/<parent-branch>" "$OLD_PARENT_TIP"
```

This works regardless of how many commits your branch owns, and regardless of which Phase C round you're on.

**Fallback** if `OLD_PARENT_TIP` is unavailable (e.g., fresh worktree, no prior tracking ref): `git rebase --fork-point --onto origin/<parent-branch> origin/<parent-branch>` uses the reflog to find the fork point. Less robust than capturing the SHA but better than visual SHA-picking from `git log`.

**The wrong forms:**

| Form | What breaks |
|---|---|
| `git rebase origin/<parent-branch>` | Tries to replay everything from `<merge-base>..HEAD`. After parent force-pushed, that means everything from the bottom of the stack up. Many false conflicts. |
| `git rebase --onto origin/<parent-branch> <SHA from git log -5>` | Works on round 1. On round 2+, the branch has multiple subtask commits, the visually-identified "first commit" is no longer the actual boundary, and the rebase silently grabs the wrong range. |
| `git rebase --onto origin/<parent-branch> origin/<parent-branch>` *after* fetch | Both args point at the new tip; rebase has nothing to do. Silent no-op. Catch this with `git status` after — if your work disappeared, you ran this. |

## 4. Force-push only with `--force-with-lease`

`--force` will overwrite the remote even if someone else has pushed since your last fetch. `--force-with-lease` refuses the push in that case, which catches the "I forgot the cron job pushed a checkpoint commit while I was working" footgun.

```bash
git push --force-with-lease origin <branch>
```

Note that `git push --force` and `git push --force-with-lease` are both flagged by the destructive-command-guard (`dcg`) on this harness and will prompt the operator for approval. That prompt is the right behavior — don't try to route around it.

**If `--force-with-lease` is rejected by git** (not by `dcg` — by git itself, because the remote moved):

1. `git fetch origin <branch>` to see what's on the remote.
2. `git log origin/<branch>..HEAD` — your local commits not yet pushed.
3. `git log HEAD..origin/<branch>` — the remote commits you didn't expect.
4. If the remote has unexpected work: stop, ask the operator. Don't rebase on top of work whose origin you don't understand.
5. If the remote work is benign (e.g., a CI auto-format commit you can re-create), re-do your rebase from the new remote tip and try again.

Do NOT escalate to plain `--force` to bypass either git's lease check or `dcg`'s prompt. Both protections exist because force-pushing the wrong thing destroys other people's work.

## 5. Amend vs new commit — default to new commit

The default for Phase C fixes is **a new commit on top of the existing branch**. That:

- Keeps the history human-readable (one commit per PR-comment round).
- Lets the operator see what was added in response to feedback as its own diff.
- Avoids the cascade-force-push cost across downstream PRs.

**The one exception**: the bot's complaint is specifically about the **original commit's** subject or body. Example: `"The commit subject is 75 chars, over the 72-char CLAUDE.md limit"`. The only way to fix that complaint is to rewrite the existing commit.

First check the repo's merge strategy — for squash-merge repos, `gh pr edit <n> --title "<new>"` replaces the rebase below entirely (see gotcha #16). The rebase is only needed when in-branch commits land on `main` verbatim.

When the rebase is needed, the non-interactive recipe is:

```bash
# Pre-set the variables — substitute concrete values, don't leave <angle-brackets> in the command.
SHORT=eca52af                                   # short SHA of the commit to reword
NEW_MSG='fix(security): PROJ-024 remove anonymous userId fallback'

# Edit the todo list to change "pick <SHORT>" to "reword <SHORT>".
# Edit the commit message via GIT_EDITOR to write NEW_MSG directly.
GIT_SEQUENCE_EDITOR="sed -i.bak \"s/^pick ${SHORT}/reword ${SHORT}/\"" \
  GIT_EDITOR="sh -c 'printf %s \"\$NEW_MSG\" > \$1'" \
  NEW_MSG="$NEW_MSG" \
  git rebase -i HEAD~N
```

`N` is "how many commits back is the target commit". Get it with `git log --oneline | grep -n "^${SHORT}"` — the line number is `N`.

Or do this interactively if a real editor is wired up — `git rebase -i HEAD~N`, change `pick` → `reword` for the target line, save, then write the new message when prompted.

After amending: `gh pr edit <n> --title "$NEW_MSG"` to keep the PR title aligned. The amend is itself a force-push that will cascade through every downstream PR in the stack. That cost is real — confirm with the operator that this fix is worth the cascade before reaching for it.

## 6. Evolving-API conflicts cascade through stacks

**The failure**: in the audit, the internal HMAC token format mutated 5 times across the 24-PR stack:

```
PROJ-004:  ${peer}.${scopes}.${ts}.${hmac}              (4-part)
PROJ-007:  ${peer}.${scopes}.${act}.${ts}.${hmac}       (5-part, added `act`)
PROJ-013:  ${peer}.${scopes}.${act}.${nonce}.${ts}.${hmac}  (6-part, added `nonce`)
```

When PR #29's review fix moved the scope-check after HMAC verification, the conflict in `auth.ts` re-occurred on every downstream PR (#34, #41) that had touched the same area with its own evolution of the token format. The conflict shape was the same each time — three-way merge of:
- HEAD: the latest format + the HMAC-first reorder
- Original PR base: the pre-evolution format
- The PR's commit: the next-evolution format

**The fix**: recognize that the conflict shape repeats. Resolve it the same way each time: keep HEAD's structural choice (the reorder) AND apply the incoming PR's format change. Don't re-debate the design every time.

If the API is mutating that much across the stack, that's a planning signal — consider whether the API change can land as its own PR earlier so the rest of the stack doesn't keep colliding.

## 7. `git add -A` / `git add .` is dangerous in worktrees

Each worktree carries its own untracked files: temporary `uv.lock`s from prior runs, vitest cache files, `.DS_Store` on macOS, etc. `git add -A` happily stages all of them, producing commits that include garbage.

**The fix**: stage specific files by name. Always.

```bash
# BAD
git add -A
git commit

# GOOD
git add packages/control-plane/src/auth/webhook-key.ts packages/control-plane/src/auth/webhook-key.test.ts
git commit
```

If you're not sure what should be staged, `git status --short` and read the list.

## 8. `--no-verify` is treating the symptom

Pre-commit hooks run on every commit unless you pass `--no-verify`. Hook failures are signal — the code is in a state the hooks consider invalid (lint error, type error, test failure, secret leak, etc.). Fix the code; don't bypass the hook.

**The only legitimate use of `--no-verify`**: the hook itself is broken (e.g., environment misconfiguration) AND fixing the hook is out of scope for the current PR. Even then, prefer fixing the hook.

## 9. Conversation memory > shared TaskList for cross-agent state

**The failure**: the first audit attempt created a TaskList with 27 entries (one per subtask) at the orchestrator level, then spawned a team. Teammates spawned into the team auto-claimed the orchestrator's tasks and started working on the wrong tickets. Dev tried to start PROJ-002 while in the middle of PROJ-003. QA tried to start PROJ-004 instead of QA'ing PROJ-003.

**The fix**: track multi-subtask progress in conversation memory only. Never put orchestrator-level tasks into a TaskList that any subagent can see. For 24 subtasks, the progress queue is small enough — a simple inline list of "P1: X ✓ → Y ✓ → Z" is sufficient.

**This is not an argument against teams per se.** The `dev-team` skill runs each *subtask's* Dev/QA/Reviewer roles as an agent team on purpose — that's fine, because that team is scoped to one unit of work. The failure here was an *orchestrator-level, cross-subtask* TaskList shared with a team, which is the one thing to avoid. Keep the parent-level progress queue private to the orchestrator; let each subtask's `dev-team` own its own scoped team. See the `dev-team` skill's "Team guardrails".

## 10. Bot re-review pings don't trigger immediate response

After posting `@<bot> re-review`, the bot typically responds in seconds to a few minutes — but not synchronously. Don't sleep-poll for the response. If the operator is watching, they'll see it land; if you need to address the response, it'll arrive as a new comment when you next triage that PR.

## 11. APPROVED state ≠ done

A bot's `APPROVED` review can still list nits in the body. Don't treat the state as a signal to skip the body. Always read the body and triage its contents.

## 12. Don't reply to bot "No issues found" summaries

These are informational — they confirm the bot ran and produced no actionable output. Replying to them adds noise to the PR conversation and (depending on the bot's wiring) can re-trigger the bot to post a follow-up summary. Skip silently.

## 13. Conflict during rebase: resolve in place, don't `--abort` reflexively

When `git rebase --onto` reports a conflict, the worktree has `<<<<<<<` markers in the conflicting files. Your instinct may be to `git rebase --abort` and try a different approach. Resist that — usually the conflict is real and the resolution is straightforward (you've seen the shape before; see gotcha #6).

Edit the conflicting files, `git add` them by name, `GIT_EDITOR=true git rebase --continue`. The `GIT_EDITOR=true` short-circuits the commit-message prompt that interactive rebase opens.

## 14. Worktrees persist across sessions

If the operator's last session left worktrees at `~/.claude-worktrees/...`, you'll find them on this session's `git worktree list`. Don't blow them away — they're your scaffold. Just `git fetch origin` and check `git status` in each one.

When the stack is fully merged and the operator wants to clean up, `git worktree remove <path>` (one at a time, deliberately).

## 15. PR base auto-rebase on merge

When PR N (base = PR N-1's branch) is merged via "Squash and merge" or "Rebase and merge", GitHub automatically changes PR N+1's base to whatever PR N's old base was (usually `main` or the prior PR's branch). This is correct GitHub behavior but it means the operator can merge the stack out of order if they want — the auto-rebase keeps things sane.

If the operator merges PR #29 before #27 (for some reason), PR #27's base auto-flips to `main`, which may produce a real conflict because PR #29's changes aren't in `main` yet. Let the operator know this is a risk if they ask about merging out of order.

## 16. PR titles drive the merge-commit subject

For squash-merges, GitHub uses the PR title (not the commit subject of any single commit) as the subject of the resulting merge commit on `main`. So fixing a "subject too long" complaint can sometimes be accomplished by just `gh pr edit <n> --title "..."` without rewriting the in-branch commit. Worth checking before reaching for `git rebase -i`.
