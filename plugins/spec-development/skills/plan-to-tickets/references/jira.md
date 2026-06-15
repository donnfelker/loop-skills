# Jira — platform-specific notes

*Stub — fill this in the first time you run the plan-to-tickets skill against Jira.*

When you do, capture these specifics (mirror the structure of `clickup.md`):

- **MCP tools you'll use**: likely `mcp__atlassian_*` or similar. Confirm `create_issue`, `edit_issue`, `create_issue_link`, `delete_issue` etc. tool names.
- **Issue type vs. status**: Jira distinguishes issue types (Epic / Story / Task / Sub-task) from workflow status (To Do / In Progress / Done / etc.). Document both — issue type is set on create; status is changed via a transition, not a direct field update.
- **Transitions, not status updates**: Jira status changes go through workflow transitions (`transition_issue` with a transition ID), not direct field writes. The available transitions depend on the current status. Document the transition IDs for your project's workflow.
- **Hierarchy model**: Jira's hierarchy is Epic → Story → Sub-task, and the cardinality / nesting rules are project-config-dependent. Confirm with the operator before mapping plan phases.
- **Description format**: Jira Cloud supports Atlassian Document Format (ADF), not raw markdown. The MCP may accept markdown and convert it, or may require ADF directly — verify with a test create.
- **Issue links (dependencies)**: Jira uses issue link types (`Blocks` / `is blocked by` / `Relates` / etc.). The MCP's `create_issue_link` takes a link type ID and two issue keys — document which link type and which direction.
- **Required fields**: many Jira projects have required custom fields (assignee, priority, components). A create will fail if any required field is missing. Get the project's required field list before bulk-creating.
- **Project key prefix**: Jira issue keys have a project prefix (e.g., `ENG-1234`). Document the project key for the import.
- **Known quirks**: anything that wasted time on the first run.

Until this file is filled in, fall back to the generic SKILL.md guidance and verify each step against Jira's actual behavior — especially the transitions-vs-status-field distinction, which is the biggest gotcha for someone coming from a flat-status tracker.
