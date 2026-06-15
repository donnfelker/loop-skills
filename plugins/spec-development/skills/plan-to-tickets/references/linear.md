# Linear — platform-specific notes

Use when the target tracker is Linear.

## Authentication setup

The Linear MCP server runs at `https://mcp.linear.app/mcp` over HTTP transport and is gated by OAuth. Until the operator completes OAuth, the Linear tools don't exist from the agent's perspective — `ToolSearch` against `mcp__linear-server__*` returns nothing and any direct call would fail with an unknown-tool error.

Confirm status before doing anything else:

```bash
claude mcp list 2>&1 | grep -i linear
# Pre-auth:  linear-server: https://mcp.linear.app/mcp (HTTP) - ! Needs authentication
# Post-auth: linear-server: https://mcp.linear.app/mcp (HTTP) - ✓ Connected
```

If it says `Needs authentication`, stop and ask the operator to authenticate. You cannot complete OAuth on their behalf. Instructions for them:

1. Run `/mcp` in the Claude Code CLI.
2. Select `linear-server`.
3. Complete OAuth in the browser that opens.
4. Confirm `✓ Connected` in the CLI before telling the agent to proceed.

Once connected, load the tool schemas you'll need with a single `ToolSearch` `select:…` call rather than guessing names.

## MCP tools you'll use

- `mcp__linear-server__list_teams` — enumerate teams; needed to pick the right team identifier (the uppercase short code, e.g., `ENG`, `OPS`).
- `mcp__linear-server__list_projects` — accepts `team` + optional `query`. Always check for an existing project before creating to avoid duplicates.
- `mcp__linear-server__list_issue_statuses` — team-scoped. Capture the status IDs you'll need (`Done`, `In Progress`, etc.) before any create that sets a state.
- `mcp__linear-server__list_issue_labels` — team-scoped. Pass `limit: 250` to get the full set in one call.
- `mcp__linear-server__create_issue_label` — create labels before applying them; the `save_issue` `labels` parameter won't auto-create them.
- `mcp__linear-server__save_project` — **unified create + update**. Create returns the project UUID; capture it for later issue assignment.
- `mcp__linear-server__save_issue` — **unified create + update**. Pass `id` to update an existing issue, omit it to create. Also the tool you use to wire `blockedBy` after the fact.
- `mcp__linear-server__get_issue` — accepts `id` (string) plus optional `includeRelations: true`. The `id` parameter **must be a string** — passing `{query: "TEAM-8"}` returns a Zod validation error (`"expected": "string", "code": "invalid_type"`). Use `{id: "TEAM-8"}`.
- `mcp__linear-server__list_issues` — for projects with 50+ issues, the payload exceeds the agent's output token cap. See the gotcha section below.

## Description field — markdown is native, no separate field

Linear's `save_issue` and `save_project` accept markdown directly in `description`. There is **no** equivalent of ClickUp's `markdown_description` vs. `description` distinction — just write markdown. Tables, checklists, links, headings, and code blocks all render correctly in the Linear UI.

## Status workflow — team-scoped, look up before setting

Linear statuses belong to a team's workflow (not the issue, not the project). Always call `list_issue_statuses` for the target team first and capture the status IDs.

The Linear defaults are `Backlog`, `Todo`, `In Progress`, `In Review`, `Done`, `Canceled`, `Duplicate`, but customized workflows are common — don't hardcode these.

Setting status at create time works — `save_issue` with `status: "Done"` succeeds in one shot for already-complete tickets, no separate update pass needed (unlike ClickUp). If you're unsure of the exact state name on the operator's workflow, create without `state` first and update with a second `save_issue` call.

## Dependency direction — `blockedBy` on the child

Linear's relation parameter on `save_issue` is `blockedBy: [<issue_id>, …]`. Semantics:

- `mcp__linear-server__save_issue: {"id": "TEAM-6", "blockedBy": ["TEAM-5"]}` reads as "TEAM-6 is blocked by TEAM-5".
- Fire the call against the **child** (blocked) issue, not the parent (blocker).
- This matches the natural reading of `Blocked by:` in plan docs (child blocked by parent).
- The inverse relation appears automatically in the Linear UI — "Blocked by" on the child and "Blocking" on the parent. You don't have to wire both directions.

Use issue identifiers (`TEAM-6`) rather than UUIDs — both work, but identifiers are stable, human-readable in `ids.csv`, and survive renames.

## Subtask / parent-child semantics — true sub-issues

Linear has **true sub-issues** (not just project hierarchy). When creating a child issue, set the parent-issue field on `save_issue` (verify the exact parameter name — likely `parentId` — from the loaded tool schema before relying on it).

Sub-issues nest visually under the parent in the Linear UI. There is no enforced depth limit, but convention is two levels (phase → task).

For plan-to-tickets, the natural mapping is:

- Plan → Linear **Project** (1)
- Phase → top-level **Issue** with no parent, member of the project
- Task → **sub-Issue** under its phase issue

A typical multi-phase plan produces one project, a handful of phase issues, and a few dozen task sub-issues.

## Project assignment — required at create time for top-level issues

When creating a top-level issue, pass:

- `team` — team identifier (the uppercase short code, e.g., `ENG`) or UUID.
- `project` — project UUID (or name) returned by `save_project`.

Sub-issues automatically inherit the parent's team and project — you can omit both when passing the parent-issue field.

Linear doesn't require a project — issues can exist team-scoped without one — but for plan-to-tickets, always create the project first. It gives the operator a single landing page for the import.

## Labels — create first, then apply

Labels are team-scoped. Workflow:

1. `list_issue_labels` with `limit: 250` to see what already exists.
2. `create_issue_label` for any missing labels.
3. Pass label names to `save_issue` via the `labels` parameter at create time.

Confirm whether labels need to be passed by name or by ID against the tool's actual schema before relying on either.

## Project icon validation — known footgun

`save_project` rejects invalid icon values:

```
Argument Validation Error - icon is not a valid icon.
```

Safe default: **omit the `icon` field** unless the operator explicitly specifies one. If they do, verify the icon name against Linear's icon set before sending.

## Rate limits and parallel batches

`save_issue` creates parallelize cleanly when fired per-phase. Recommended ceiling: **10–15 parallel `save_issue` calls per turn**. Dependency wires (updates by `id`) are even safer to parallelize — they never contend.

## `list_issues` output-token gotcha

For projects with 50+ issues, `mcp__linear-server__list_issues` returns a payload that exceeds the agent's output token cap. The harness saves the full result to a tool-results file and returns the path in the error:

```
/Users/<user>/.claude/projects/<session>/tool-results/mcp-linear-server-list_issues-<ts>.txt
```

**Don't retry the call** — the file already has the data. Parse it with Python:

```python
import json
from collections import Counter

data = json.load(open(file_path))
issues = data.get('issues', [])
print(Counter(i['status'] for i in issues))
```

Use the same pattern for verifying parent assignment, dependency wiring, or any other bulk check.

## `save_issue` is unified create + update

There is no separate `update_issue` tool. `save_issue` creates when called without `id` and updates when called with `id`. Same shape as `save_project`.

Practical implication: **dependency wiring uses the same tool as create**, just with `id` set:

```
mcp__linear-server__save_issue: {"id": "TEAM-6", "blockedBy": ["TEAM-5"]}
```

Note this is different from ClickUp's separate `add_task_dependency` call — don't reach for a separate "update" tool that doesn't exist.

## URL → ID extraction

- Issue URL: `https://linear.app/<workspace>/issue/<TEAM>-<N>` — the identifier `<TEAM>-<N>` is what most tools accept.
- Project URL: `https://linear.app/<workspace>/project/<slug>-<short-uuid>` — the trailing 12 hex chars are the prefix of the project UUID, but the full UUID is needed for API calls. Capture the return value from `save_project` rather than parsing it from the URL.
- Workspace slug comes from the team URL; the team identifier is the uppercase short code shown next to issue IDs.

## Known quirks

Things that waste time on the first run and are worth catching up front:

- **OAuth blocker** — the agent has zero Linear visibility until the operator authenticates; `ToolSearch` returns nothing for `mcp__linear-server__*` until then.
- **`save_project` icon validation** — invalid icon names hard-fail the create. Omit by default.
- **`get_issue` argument typing** — `id` must be a string; passing `{query: "TEAM-8"}` returns a cryptic Zod error.
- **`list_issues` output-cap** — payloads for 50+ issues are saved to a file; read the file, don't retry the call.
- **Default workflow state names vary per team** — don't hardcode `"Done"`; call `list_issue_statuses` first.
- **Issue identifiers (`TEAM-5`) are auto-assigned sequentially per team** — they survive renames and are stable across the team's history. Prefer them over UUIDs in `ids.csv` for readability.
