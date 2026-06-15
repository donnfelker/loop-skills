# Markdown fallback — file-based ticket tracking

Use this when no task-tracker MCP is connected, or when the operator explicitly prefers a filesystem-based approach (lighter weight, version-controllable, no third-party dependency).

## What you produce

A directory of markdown files that together act as the ticket system. Agents (or humans) check off boxes as work progresses. Git becomes the audit log.

```
plan-tickets/
├── README.md                      # root: phase index with checkboxes
├── P0-prerequisites.md            # all P0 tasks, each with a checkbox + Steps checklist
├── P1-foundation.md
├── P2-existing-providers.md
├── P3-new-providers.md
├── P4-web-ui.md
└── P5-acceptance-polish.md
```

The root `README.md` lists phases with top-level checkboxes and links into each phase file. Each phase file lists tasks with their own checkboxes plus an inline Steps checklist (TDD red → green → refactor → verify → complete, or whatever cadence the source plan uses).

## Why this works

- **Agents can check boxes.** Markdown checkboxes (`- [ ]` → `- [x]`) are trivially editable with `Edit` or `Write` tools. An agent picking up a task flips the box at the start, and again at the end. This gives you the same coordination benefit as ClickUp's "in progress" / "complete" statuses without any external system.
- **Git is the audit trail.** Every state change is a commit. You see exactly who flipped which box and when. No need for a separate activity log.
- **Progressive disclosure of detail.** Root file fits in 30 lines; each phase file holds full task specs. An agent reading the root only loads the phase landing pages, then opens just the phase it's working on.
- **Reversible.** If the operator later wires up a real tracker, the markdown structure maps cleanly into Phase → Task → Step subtasks (or Phase → Task with checklist) — you've already negotiated the hierarchy.

## Root `README.md` template

```markdown
# [Project name] — Implementation plan

> Source: [link to source doc]. Imported: YYYY-MM-DD.

## How to use this directory

Each phase below links to a file with its tasks. Tasks have a `[ ] Task complete` checkbox at the top — flip it when the whole task is done. Inside each task, a `### Steps` section has fine-grained checkboxes for the TDD / build cadence — flip those as you work.

Picking up a task: read the task spec, check the source doc sections it cross-references, then start at the first unchecked Step.

## Master progress

- [ ] [**Phase 0** — Human prerequisites](./P0-prerequisites.md) (13 items)
- [ ] [**Phase 1** — Foundation](./P1-foundation.md) (18 tasks)
- [ ] [**Phase 2** — Re-wire existing providers](./P2-existing-providers.md) (28 tasks)
- [ ] [**Phase 3** — Wire new providers](./P3-new-providers.md) (40 tasks)
- [ ] [**Phase 4** — Web UI](./P4-web-ui.md) (16 tasks)
- [ ] [**Phase 5** — Acceptance + polish](./P5-acceptance-polish.md) (12 tasks)

Check a phase box when **every** task box inside that phase is checked.

## Conventions

- Task IDs: `P{phase}.T{nn}` (e.g., `P1.T07`).
- Cross-task dependencies: each task has a `**Blocked by:**` line citing the IDs.
- Commit convention: `<type>(<scope>): P{n}.T{nn} <short imperative>` (e.g., `feat(ingest-anthropic): P1.T15 implement raw landing`).
- One task per branch / PR.
```

## Phase file template (`P{n}-{slug}.md`)

```markdown
# Phase {n} — {theme}

> {one-paragraph phase narrative from source doc}

**Acceptance gate:** {phase-level acceptance criteria}

---

## P{n}.T01 — {short title}

- [ ] **Task complete** — flip when every Step below is checked

**Effort**: S (1h) / M (2-3h) / L (4h)
**Blocked by**: {list of P{x}.T{yy} IDs, or `none`}
**Cross-refs**: {design doc § references}

### Context

{1–3 sentences explaining what this task accomplishes and why.}

### Test spec (write FIRST)

- File: `path/to/test.spec.ts`
  - `test("AC1: …")` — purpose
  - `test("AC2: …")` — purpose

### Implementation requirements

- {file paths to create/edit}
- {specific behavior}

### Acceptance criteria

- **AC1:** {assertion}. Verified by `path/to/test.spec.ts → test("AC1: …")`.
- **AC2:** {assertion}. Verified by `test("AC2: …")`.

### Verification gate

```
pnpm lint
pnpm typecheck
pnpm test
```

### Definition of Done

- All ACs pass.
- Commit: `<type>(<scope>): P{n}.T01 <short imperative>`.
- Task box flipped.

### Steps

- [ ] **Red** — write the named tests; watch them fail.
- [ ] **Green** — implement to pass tests.
- [ ] **Refactor** — clean up; tests still pass.
- [ ] **Verify** — run verification gate; all exit 0.
- [ ] **Complete** — commit and flip the task-complete checkbox above.

---

## P{n}.T02 — {next task}

{...}
```

## How to drive an agent against this structure

The whole point of the checkbox convention is that agents can self-coordinate. A common pattern:

1. Agent reads `README.md` to see which phases are still open.
2. Agent opens the first phase file with an unchecked phase-box.
3. Agent finds the first task whose `Task complete` box is unchecked AND whose `Blocked by:` tasks all have their boxes checked.
4. Agent flips the task's first unchecked Step to `[x]` (claim) and starts work.
5. After each Step's work, agent flips the corresponding Step checkbox.
6. When all Steps are checked, agent flips the `Task complete` box and commits.
7. After committing, agent re-reads `README.md` and the phase file to see if the phase is now fully done. If yes, flip the phase box too.

This is essentially the same coordination protocol ClickUp's dependency view gives you — just expressed in plain text.

## Producing the files programmatically

From the source plan doc, build the structure in this order:

1. **Parse phases.** Identify the top-level phase headings and their narrative paragraphs.
2. **Parse tasks per phase.** For each phase, extract the task entries with their IDs, titles, effort, blocked-by, and full spec sections.
3. **Write the root `README.md`** with one bullet per phase.
4. **Write each phase file** with the template above filled in for each task.
5. **Sanity-check cross-refs.** Every `Blocked by: P{x}.T{yy}` should reference a task ID that actually exists somewhere in the directory. Grep for orphans.

Use `Write` for the initial files. Once they exist, agents working tasks use `Edit` to flip individual checkboxes — never `Write` (which would clobber other in-progress checkbox state if multiple agents are coordinating).

## When to recommend this over a real tracker

- The operator wants version-controlled progress alongside the code.
- No tracker MCP is wired up and the operator doesn't want to set one up just for this.
- The project is small enough (~20–50 tasks) that a UI doesn't add much.
- Multiple agents will be working in parallel and the operator wants the coordination state to live in the same repo.

## When *not* to recommend it

- The operator already uses a tracker and wants leadership visibility there.
- The plan has >150 tasks (file size gets unwieldy; tracker filters/views become valuable).
- Cross-team coordination is needed (non-engineers won't read markdown files).
- The operator wants critical-path or burndown views — those need a real tracker.
