# Asana — platform-specific notes

*Stub — fill this in the first time you run the plan-to-tickets skill against Asana.*

When you do, capture these specifics (mirror the structure of `clickup.md`):

- **MCP tools you'll use**: confirm exact tool names for create / update / dependency / delete.
- **Hierarchy**: Asana has Project → Section → Task → Subtask. Decide how to map plan Phases (sections vs. parent tasks vs. separate projects).
- **Description field**: HTML notes vs. plain text — verify markdown handling.
- **Custom fields**: Asana uses custom fields heavily; document any required fields for the target project.
- **Dependencies**: Asana supports task dependencies (`add_dependencies` / `add_dependents`). Document direction and which call to use.
- **Status / sections**: Asana doesn't have ClickUp-style statuses; "done" is a boolean and progress lives in sections or custom fields. Document the convention for the target project.
- **Multi-project assignment**: an Asana task can live in multiple projects simultaneously. Useful if the plan spans projects.
- **Known quirks**: anything that wasted time on the first run.

Until this file is filled in, fall back to the generic SKILL.md guidance and verify each step against Asana's actual behavior.
