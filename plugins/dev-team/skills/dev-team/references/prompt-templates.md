# Prompt Templates

Copy-paste skeletons for the role agents this loop dispatches. Replace `<angle-brackets>` with concrete values. Treat these as starting points — adapt to the specific unit of work. The same prompt works whether the role runs as a **teammate** (team mode) or a **one-shot subagent** (fallback mode) — see the execution model in SKILL.md.

These templates are distilled from real multi-agent implementation runs. The phrases marked **important** are the ones that demonstrably change agent behavior in production.

## Table of contents

- [Dev agent](#dev-agent)
- [QA agent](#qa-agent)
- [Reviewer + commit agent](#reviewer--commit-agent)
- [Combined single-shot agent (small units)](#combined-single-shot-agent-small-units)
- [Adapting the templates](#adapting-the-templates)

The SKILL.md references these anchors directly — e.g., `prompt-templates.md#dev-agent`.

---

## Dev agent
<a id="dev-agent"></a>

```
You are the developer implementing **<UNIT_ID>** — <one-line description>.

## Workspace
- **Workspace**: <absolute path>
- **Branch**: <branch> (off <base>)
- Always prefix Bash with `cd <workspace> && …`. Absolute paths in file ops.
- Do NOT touch <operator's primary checkout path>.

## Spec (verbatim from <source>)
<paste the full spec body — severity, location, description, data flow, impact, every remediation bullet>

## Scope partition
- **(IN SCOPE)** Bullet 1: …
- **(IN SCOPE)** Bullet 2: …
- **(OUT OF SCOPE — deferred to <follow-up-name>)** Bullet 3: … (e.g., needs infrastructure provisioned outside the repo)
- **(OUT OF SCOPE — large new capability)** Bullet 4: …

For deferred bullets, list them in your READY report so they can become follow-up items.

## What to do
1. Read these files: <paths>
2. Implement bullet 1 by: <hint>
3. Implement bullet 2 by: <hint>
4. Write tests for: <cases>
5. Run: <canonical test command> and <typecheck && lint command>
6. DO NOT COMMIT. Leave changes unstaged.

## Constraints
- Follow CLAUDE.md conventions (don't re-list them — inherit).
- No back-compat shims unless explicitly justified.
- No --no-verify, no `git add -A`.

## READY report
Return: files changed (grouped), approach summary, test counts + key new tests,
deferred items with proposed follow-up names, anything escalated.

If you hit a real blocker, return BLOCKED with specifics. Otherwise push through.
```

---

## QA agent
<a id="qa-agent"></a>

```
You are QA verifying the developer's implementation of **<UNIT_ID>**. Return
`PASSED — <UNIT_ID>` or `FAILED — <UNIT_ID> — cycle N`.

## Workspace
<same as dev>

## Spec
<verbatim spec>

## What the dev reports
<paste the dev's READY message>

**Trust nothing — verify by reading code and running tests yourself.**

## Step 1 — Inspect the diff
- `git status && git diff --stat`
- Read every file the dev claims to have changed.

## Step 2 — Verify each remediation bullet
For each bullet:
- Find the code that addresses it.
- Verify implementation actually achieves the bullet's intent (not log-only, not partial).
- Specifically: <bullet-specific checks>

## Step 3 — Check test quality
- Read the new tests' assertion bodies, not just titles.
- **Mental revert**: if the fix were reverted, which tests would fail? Confirm at least one new test would actually fail without the fix. If none, the tests are theater — flag that.
- Run the tests yourself.

## Step 4 — Run the canonical checks
<paste exact commands>

## Step 5 — Decision
PASSED with 4-6 bullet verification summary OR FAILED with itemized issues
(file:line + severity + remediation hint). Nits-only findings → PASSED with nits noted.
```

---

## Reviewer + commit agent
<a id="reviewer--commit-agent"></a>

```
Combined code review + commit for **<UNIT_ID>**. Return
`APPROVED & COMMITTED` (SHA) or `REJECTED` (itemized).

## Workspace
<same>

## Spec (verbatim from <source>)
<paste the full spec body — same as the Dev/QA prompts. The reviewer commits, so it
must check the diff against the original remediation bullets, not just QA's summary.>

## Context
<summary of dev's implementation + QA's findings if applicable>

## Steps
### Step 1 — Read the diff
### Step 2 — Code review (use superpowers:requesting-code-review skill if available)
### Step 3 — Manual checklist
- Correctness: <unit-specific>
- Security: <unit-specific>
- Style (CLAUDE.md): follow the project's conventions; no comments restating WHAT; no dead code.
- No new deps.
- No back-compat shims.
- Imports / dead code clean.

### Step 4 — Run canonical checks
<paste commands>

### Step 5 — Decision

### Step 6 — On APPROVED, commit
git add <specific files by name>
git commit -m "$(cat <<'EOF'
<conventional-commit subject, ≤72 chars>

<paragraph: what changed and why>
<paragraph: deferred items + follow-ups if any>

Closes <UNIT_ID> (<unit-URL if any>).

Co-Authored-By: <your assistant co-author line>
EOF
)"

No push. No --no-verify. No `git add -A`.

Return SHA + git log -1 --stat summary, OR itemized rejection.
```

---

## Combined single-shot agent (small units)
<a id="combined-single-shot-agent-small-units"></a>

```
Full Dev + verify + commit for **<UNIT_ID>**. Single-shot. Return
`APPROVED & COMMITTED` (SHA) or `BLOCKED` (reason).

## Workspace + Spec
<as above>

## What to do
1. Read <files>.
2. Implement: <bullets>.
3. Tests for <cases>.
4. Run <commands>.
5. Commit with this message template:
   <type>(<scope>): <UNIT_ID> <short subject>
   <body>
   Closes <UNIT_ID> (<URL if any>).
   Co-Authored-By: <your assistant co-author line>

No push. No --no-verify. Stage by file name, never `git add -A`.
```

---

## Adapting the templates

In practice, these templates get modified in flight. Common adaptations:

1. **Collapse Dev + QA + Reviewer into one agent for small units** — a single-comment-in-single-file fix doesn't warrant 3 round-trips. See the role-collapsing table in SKILL.md.
2. **Tighten or expand the canonical checks** — match the exact suites the operator cares about for this unit, not a generic "run all tests."
3. **Thread cycle-N feedback into the Dev prompt** — on cycle 2+, paste the QA's or Reviewer's itemized findings into the Dev's "What to do" so it sees exactly what failed.

The skeletons are a starting point, not a contract.
