---
name: plan-to-tickets
description: "Import a structured planning document (implementation plan, project plan, design doc with tasks) into a task-tracking system as a ticket hierarchy with dependencies wired. Use whenever the operator says 'break this plan into tickets', 'import this into ClickUp/Linear/Jira/Asana', 'create tickets for this plan', 'turn this doc into tasks', 'set up the project in [tracker]', or any variation of converting a written plan into trackable work items. Also use when the operator references a planning doc and a target tracker URL or list in the same request. Covers hierarchy negotiation, ID mapping, dependency wiring, and the platform-specific gotchas that bite on the first attempt. Includes a readiness check up front that will stop and ask the operator to spec the plan out further if it's not yet structured enough to ticket — refusing to import a half-baked plan is part of the contract, not a failure mode."
---

# Plan to Tickets

This skill converts a written plan into a hierarchy of tickets in a task tracker (ClickUp, Linear, Jira, Asana, Notion, GitHub Projects, etc.). It captures the durable process lessons that aren't obvious the first time you do it at scale, plus platform-specific quirks that will bite if you don't know about them.

The work is mostly mechanical — read structured input, create things, wire relationships — but two failure modes are common and expensive:

1. **Wrong hierarchy depth.** A 3-level hierarchy with 5 step-subtasks per task can balloon a 130-task plan into 770 tickets. Most of those step subtasks carry no information beyond the parent description and just add noise. Negotiate the depth before you create anything.
2. **No ID map.** Trackers return their own opaque IDs (`86ahjxrh2`, `LIN-1234`, etc.) when you create a ticket. You need those to wire dependencies later. If you don't capture them as you create each ticket, you'll have to do a second pass to look them all up — slow and error-prone.

The rest of this skill exists to make sure you don't fall into either trap and to surface the smaller gotchas (status string formats, markdown rendering, batching strategy).

## Step 1: Capture intent

The operator will hand you either a planning document or point at one. Before any tool calls, confirm three things:

- **Source doc** — the markdown/text file. Read it before asking questions; you'll want to know task counts, phase structure, and dependency density.
- **Target tracker** — which system to import into (ClickUp, Linear, Jira, Asana, Notion, GitHub Projects, etc.). **If the operator didn't name one, ask before doing anything else** — the choice affects what's possible (status workflows, native dependencies, markdown rendering, tag semantics) and which MCP/integration is required.
- **Target container** — within that tracker, the specific list / project / board / repo. Get the URL or ID; ambiguity here wastes a lot of work.
- **Status of items already done** — if the doc has any "complete" or "in progress" callouts (e.g., prereqs already finished), confirm which to mark accordingly.

### Readiness check: is the plan well-scoped enough to ticket?

After reading the source doc and before doing anything else, assess whether the plan is actually ready to become tickets. Importing a half-baked plan produces half-baked tickets — and unlike a draft doc, you can't easily edit 100+ tickets into shape after the fact. It's much cheaper to refuse the work and ask the operator to tighten the plan first.

**Required signals (if any are missing, stop and inform the operator)**:

- **Discrete, separable task units.** Not a wall of prose; not a vision statement; not a meeting transcript. There has to be a clear unit of work that maps to a ticket. Look for numbered/bulleted task lists, task headings, or sections labelled with task identifiers.
- **Each task has a title or identifier.** It doesn't have to be a formal ID scheme like `P1.T07` — even just numbered tasks per section is fine — but every task needs to be uniquely addressable.
- **Some hierarchical or grouping structure.** Phases, sections, milestones, themes — anything that lets a reader find related tasks together. A flat list of 200 unrelated tasks is technically importable but the tracker view will be unusable.

**Strongly recommended signals (if missing, warn the operator but proceed if they confirm)**:

- **Per-task context.** Even a sentence describing what the task accomplishes. Tickets without context become "what does P3.T22 mean again?" lookups within a week.
- **Acceptance criteria or definition of done.** How does a person picking up the task know when they're finished? Without this, the ticket is an open invitation to scope creep.
- **Dependencies expressed somehow.** `Blocked by:`, "after X is done", arrows in a diagram — any signal of ordering. Without this, dependency wiring (Step 6) is impossible and the tracker becomes a flat bag of work.

**Nice-to-have (worth noting but never blocking)**:

- Effort estimates (S/M/L or hours).
- Cross-references to a design doc or spec.
- Verification/test specs.
- Commit-message or branch-naming conventions.

**If required signals are missing, stop here and tell the operator something like**:

> "The plan you've shared isn't ready for ticket creation yet — it reads more like a [brief / design narrative / meeting notes / vision doc]. Before we can ticket this, the plan needs to:
>
> - [specific gap 1, e.g., 'Break the work into named or numbered tasks; right now it's a single section of prose']
> - [specific gap 2, e.g., 'Group those tasks under phases or milestones']
> - [specific gap 3, e.g., 'Add a one-sentence "what does done look like" line per task']
>
> Want help spec'ing this out first? I can [reference relevant skill: `superpowers:writing-plans` if available, or 'walk through it with you']. Once the plan looks like an implementation plan, come back and we'll ticket it."

Be specific about what's missing — generic "the plan needs more detail" feedback isn't actionable. Quote actual lines from the source doc to show what you saw vs. what you'd need to see.

**If strongly-recommended signals are missing**, warn explicitly and get the operator's explicit consent before continuing:

> "I can ticket this, but heads-up: the plan doesn't include acceptance criteria per task. The resulting tickets will have titles and a context paragraph but no 'definition of done' — which usually causes scope creep later. Want to add ACs before we ticket, or proceed as-is?"

Don't quietly proceed in this case. The cost of asking is 30 seconds; the cost of importing 100 low-quality tickets is hours of cleanup.

### Tracker confirmation pattern

If the operator didn't name a tracker, use `AskUserQuestion` with a single question listing the trackers you can actually drive. Don't just list every popular tracker — only offer ones where you have a working MCP or integration. Before asking, do a quick capability check against your actual available-tools list (the MCP prefixes vary by environment — common ones are `mcp__clickup__*`, `mcp__linear__*`, `mcp__atlassian_*` for Jira/Confluence, `mcp__asana__*`, `mcp__github__*` — but always verify against what's actually loaded; don't assume from these prefixes).

If none are available, offer the file-based fallback before assuming the operator wants to wire up an integration: "I don't see a connected task-tracker integration. I can either (a) generate a directory of markdown files that act as checkbox-driven tickets — same hierarchy, checkable by agents, version-controlled in your repo — or (b) wait while you add a tracker MCP. Want the markdown fallback?" If they say yes, read `references/markdown-fallback.md` for the full pattern (root `README.md` with phase checkboxes that link to per-phase files, each task with a `[ ] Task complete` box and a `### Steps` checklist underneath).

If exactly one is available, default to it and just confirm: "I'll import this into ClickUp via the connected MCP — is that right?"

If multiple are available, ask which one, with each option labeled by the integration name so the operator sees what's actually wired up.

## Step 2: Ask the four clarifying questions (always)

These prevent the most expensive rework. Ask all four in one pass with `AskUserQuestion` rather than one at a time. Tailor the option labels to the source doc's actual structure (e.g., if the doc uses "Phases" and "Tasks", say so).

1. **Hierarchy depth**
   - Option A: 3-level (Phase → Task → Steps). Most faithful to the doc, but multiplies ticket counts by ~5–6x. Only recommend this if the steps are genuinely distinct work items, not boilerplate process (TDD red/green/refactor, etc.).
   - Option B: 2-level with steps embedded as a markdown checklist inside the task description. Same information as 3-level, ~5x fewer tickets, still works for per-step progress tracking via the checklist UI most trackers expose.
   - Option C: Flat task list with phase tags. Loses phase landing pages but easiest to filter.
   - **Default recommendation: Option B** for most plans. Explain the math (count × multiplier) so the operator sees the tradeoff before they pick.

2. **Handling of already-done items** — Skip / include all open / include with appropriate statuses set. If a prereq section calls some items complete and others in progress, "include with statuses set" is usually right.

3. **Dependencies** — Wire `Blocked by:` relationships as tracker-native dependency links, or leave them in the description only. Native dependencies are worth it if the tracker has a critical-path view; descriptions-only is fine for small plans.

4. **Tags** — Phase / effort (S/M/L) / domain. Skip entirely is also a fine answer if hierarchy already encodes the grouping.

**Why ask all four together:** they're not independent. If hierarchy is 2-level, then phase tags become less useful because the parent already encodes phase. If you skip dependencies, the ordering question changes. Asking together lets the operator see the shape of the build before committing.

## Step 3: Plan the build (before any tool calls)

Estimate ticket count and time before you start:

- Hierarchy choice → multiplier (1x flat, 1.2x for phase-grouped 2-level, 5–6x for 3-level)
- Multiply by source task count
- Each create is one API call; each dependency is one more
- MCP call latency varies (typically 1–3s sequential); benchmark on the first 5 calls before extrapolating. Parallel batches of 15–20 per turn are a reasonable ceiling for independent calls.

If the math says >30 minutes of tool calls, tell the operator the rough estimate before starting. They may want to narrow scope (one phase at a time) or accept the time cost.

## Step 4: Set up an ID-mapping file (non-negotiable)

Before creating the first ticket, decide where you'll keep the `plan_id → tracker_id` map. This is what makes dependency wiring possible without a second lookup pass — and it's also what makes the build resumable if the conversation gets interrupted.

**Default to a repo-local path** so the file survives reboots and is reviewable by the operator:

```bash
mkdir -p .plan-import && echo "plan_id,tracker_id" > .plan-import/ids.csv
echo ".plan-import/" >> .gitignore   # if a git repo and not already ignored
```

Use `/tmp/plan-import/` only if explicitly asked for ephemeral state, or if you're not in a directory you can write to. `/tmp` is wiped on reboot, which kills resume.

After every successful create call, append `<plan_id>,<returned_tracker_id>` to the file. Don't batch this — do it immediately. The cost is one tiny `echo` per ticket; the benefit is that if anything goes wrong, the file accurately reflects what's actually in the tracker.

### Failure and resume semantics

Things can and do go wrong mid-build (network blip, rate limit, operator changes their mind). Treat the ID map as the single source of truth:

- **A single create fails**: log the error and the failing `plan_id`, but **do not** write to ids.csv. Continue with the next ticket. At the end, report the failed `plan_id`s so the operator can decide whether to retry or fix manually. Don't retry inline — the failure may be informative (e.g., a permission issue that affects every subsequent call).
- **Conversation interrupted, agent resumes later**: read `.plan-import/ids.csv` first. Every `plan_id` already in that file is already created — skip it. Pick up creating from the first unmapped `plan_id`. This is why per-ticket appends matter: a batched write that never flushed would leave you with no resume point.
- **Operator changes hierarchy mid-build**: stop and ask explicitly: "I've created N tickets under the 3-level model. Switching to 2-level now means either (a) delete those N and restart, or (b) flatten them in place by deleting the step subtasks under each task. Which would you like?" Don't decide unilaterally — the operator's tolerance for deletion vs. cleanup varies.
- **Operator wants to abort entirely**: stop. Offer to delete what's been created if the partial state is worse than no state. The ids.csv tells you exactly what to delete.

## Step 5: Create the hierarchy top-down

Build in dependency order: root → phase parents → task children. Within each parent, ticket-creation calls share no state with each other, so siblings *could* be parallelized — but in practice the parent ID is the dependency, so creates are sequential within a parent and the speedup is small. Keep it linear and simple.

**Skip dependency wiring during this step** — that's a separate batch pass in Step 6. Trying to set dependencies inline forces creates to be ordered by dependency depth and slows everything down for no benefit. Just create the tickets here; wire edges later.

**For each ticket**:

- Use the tracker's markdown-rendering description field. Examples: ClickUp uses `markdown_description` (the plain `description` field renders as literal text); see `references/<tracker>.md` for your tracker's exact field name.
- Lead with structured metadata block (ID, Phase, Effort, Blocked-by, Cross-refs) so the operator can skim quickly in the tracker UI.
- Mirror the source doc's sections faithfully (Context, Test spec, Implementation requirements, Acceptance criteria, Verification gate, Definition of Done) — operators come back to these tickets repeatedly, so don't summarize.
- If 2-level was chosen, append a `## Steps` markdown checklist at the bottom of each task description with the step items (e.g., for TDD: red → green → refactor → verify → complete).

**For status-setting on already-done items**: status strings vary per tracker — see `references/<tracker>.md`. Trackers commonly fail the *entire create call* when an unknown status is passed (ClickUp does this; others may too). Safe pattern: create the ticket without a status, then call the update endpoint with the desired status separately. An update failure just leaves the status as the default — far less destructive than a failed create.

## Step 6: Wire dependencies in a separate pass

Don't try to interleave dependency creation with ticket creation. Do all creates first, then dependencies as a single batch pass.

**The pattern**:

1. Extract every `Blocked by:` edge from the source doc into a separate CSV: `.plan-import/deps.csv` with format `child_plan_id,parent_plan_id`.
2. Translate plan IDs → tracker IDs using your `ids.csv` from Step 4. Use `awk` with exact-match on the first field, not `grep` — plan IDs like `P1.T1` and `P1.T10` will false-match each other under a prefix regex:
   ```bash
   while IFS=, read -r child parent; do
     child_id=$(awk -F, -v k="$child" '$1==k {print $2}' .plan-import/ids.csv)
     parent_id=$(awk -F, -v k="$parent" '$1==k {print $2}' .plan-import/ids.csv)
     echo "$child_id,$parent_id"
   done < .plan-import/deps.csv > .plan-import/deps-resolved.csv
   ```
3. Sanity-check before firing calls: `wc -l deps.csv deps-resolved.csv` should match. Any missing line means an unresolved `plan_id` — likely a typo in the source doc or a create that failed silently. Fix before proceeding.
4. Fire the dependency calls in **parallel batches of 15–20 per turn**. They're independent and the API is happy to take them all at once.
5. If any calls in a batch fail (network, rate limit), log the failed edge to `.plan-import/deps-failed.csv` and continue. At the end, retry the failures once; report what still fails so the operator can review.

**Direction matters**: trackers usually have two ways to express the same edge (`A waiting_on B` and `B blocking A`). Pick one direction and use it consistently. The natural reading of `Blocked by:` in a plan doc is "child waiting on parent" — so most trackers map cleanly with `child_id waiting_on parent_id`. ClickUp follows this convention; verify your tracker's reference file before assuming.

## Step 7: Verify

Before declaring done, sanity-check by spot-reading 2–3 tickets in the tracker UI:

- Did markdown render? (If you see literal `#` headers, you used `description` instead of `markdown_description`.)
- Are dependencies showing up in the tracker's "Waiting on" / "Blocked by" view?
- Is the hierarchy nested correctly (child tickets appearing under their parent)?

Count the tickets created (one per ticket creation API call) and compare to expected. Report the totals back to the operator with a link to the root ticket.

## Platform-specific gotchas

When working with a specific tracker, read the corresponding reference file:

- ClickUp → `references/clickup.md` (filled in)
- Linear → `references/linear.md` (filled in)
- Jira → `references/jira.md` (stub)
- Asana → `references/asana.md` (stub)
- Notion → `references/notion.md` (stub)
- GitHub Projects → `references/github-projects.md` (stub)
- **No tracker connected, or operator prefers file-based**: `references/markdown-fallback.md` — produces a checkbox-driven directory of phase + task markdown files that agents can self-coordinate against, with git as the audit trail.

The stubs aren't empty — each lists the specific things to capture (MCP tool names, status workflow, dependency direction, field-type quirks) so the first person to use the skill against that tracker has a checklist instead of a blank page. When you finish a job against a stubbed tracker, fill in what you learned before signing off — future you will thank present you.

## Common patterns to remember

- **Always confirm before deletes.** Most tracker MCPs require explicit confirmation; even when they don't, deletions are unrecoverable without admin work. If you create the wrong hierarchy and need to clean up, ask first.
- **Don't ask "should I continue?" mid-build.** If the operator has approved scope in Step 2, just keep going. Status updates between major phase milestones are useful; per-ticket check-ins are noise.
- **A 770-ticket build is a 770-ticket build.** Don't quietly pivot to fewer tickets without asking. If you realize partway through that the approach is too heavy, *stop and ask*, don't silently cut scope.
- **Keep your own TODO tracker in sync.** Use whichever todo-tracking tool your environment exposes (`TodoWrite` in Claude Code, `TaskCreate` / `TaskUpdate` in some Anthropic-internal harnesses) so the operator can see progress at a glance. Create one todo per phase; mark it in-progress when you start the phase and completed when every task in that phase is done.
