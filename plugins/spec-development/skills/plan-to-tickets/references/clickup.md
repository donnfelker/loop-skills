# ClickUp — platform-specific notes

Use when the target tracker is ClickUp.

## MCP tools you'll use

- `mcp__clickup__clickup_create_task` — create a ticket. `parent` field nests it as a subtask.
- `mcp__clickup__clickup_update_task` — update fields (status, description, etc.).
- `mcp__clickup__clickup_add_task_dependency` — wire `waiting_on` or `blocking` edges.
- `mcp__clickup__clickup_delete_task` — explicitly requires user confirmation before invoking; the tool description says so.
- `mcp__clickup__clickup_get_list` — has a known response-shape validation bug. May error with "Output validation error" even when the list exists. Treat this specific error as inconclusive (the list probably *does* exist) and proceed to a single test `create_task` call. If the create succeeds, the list ID is good. If the create errors with a list-not-found code (404 or similar), stop and ask the operator to re-verify the list ID — don't retry on assumption.

## Description field — use `markdown_description`, not `description`

Both fields exist. The plain `description` field is rendered as literal text (you'll see `#` characters in the UI instead of headings). The `markdown_description` field is rendered as actual markdown. Always use `markdown_description` for anything with structure.

## Status strings — exact format matters

ClickUp errors with `{"error":"Status not found"}` if you pass a status string the list doesn't recognize. The strings vary by list workflow, but the defaults are:

- `to do` (default — usually omit to use this)
- `in progress` (with a space — not `in_progress`)
- `complete` (not `completed`)
- `closed`

If you guess wrong, the *entire create call fails*. Safer pattern: create the ticket without a status, then update the status separately with `clickup_update_task`. The update can fail without taking the create with it.

## Dependency direction

ClickUp's `add_task_dependency` takes a `type` of either `waiting_on` or `blocking`.

- `task_id=A, depends_on=B, type=waiting_on` means "A is waiting on B" (A is the child/blocked task; B is the parent/blocker)
- `task_id=A, depends_on=B, type=blocking` means "A is blocking B" (A is the parent/blocker; B is the child/blocked task)

Use `waiting_on` consistently: `child_id waiting_on parent_id`. This matches how `Blocked by:` reads in most planning docs.

## Subtask creation

Pass `parent` (a ClickUp task ID) on `create_task` to nest the new ticket as a subtask. The subtask inherits the list ID from the parent automatically — you can pass either `list_id` or `parent`, but if you pass both they must agree.

ClickUp's web UI shows subtasks as collapsible children under the parent. Cards in board view show subtask counts. Subtasks can have their own subtasks (no enforced max depth in practice), but UX gets noisy past 3 levels.

## Parallelizing dependency calls

Dependency edges have no inter-edge dependencies — they're all independent. Fire 15–20 in parallel per turn using multiple tool-use blocks in a single message. The ClickUp API accepts these comfortably; we've done 272 deps across ~14 parallel batches without rate-limiting issues.

Don't try to parallelize *create* calls in the same way — children depend on parent IDs, so you need the parent's ID back before creating children. Create calls stay sequential within a parent.

## Tags

ClickUp tags must be created at the space level before they can be applied. The `tags` parameter on `create_task` only references existing tags by name; it won't create new ones. If you need tags, either:

1. Have the operator pre-create the tag set in the ClickUp UI
2. Skip tags and use the hierarchy/name conventions instead

## Status set timing

If you need to mark some tickets as already-done or in-progress (e.g., prereqs that have closed), the safest pattern is:

1. Create without `status`
2. Then call `update_task` with the desired status string
3. If the update errors, log it and continue — don't let one status mismatch tank the whole import

## Known list-validation quirk

`clickup_get_list` may return a Zod validation error like:

```
MCP error -32602: Output validation error: ... Required at content
```

This happens when the list exists but has no `content` field set (which is common for lists created via the UI). Ignore the error and proceed with `create_task` directly using the list ID. The create will succeed.

## URL → ID extraction

ClickUp URLs look like `https://app.clickup.com/<workspace_id>/v/l/li/<list_id>`. The numeric suffix after `/li/` is the list ID. Ticket URLs look like `https://app.clickup.com/t/<task_id>` — task IDs are alphanumeric (e.g., `86ahjxrh2`).
