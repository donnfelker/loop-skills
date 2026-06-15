# Notion — platform-specific notes

*Stub — fill this in the first time you run the plan-to-tickets skill against Notion.*

When you do, capture these specifics (mirror the structure of `clickup.md`):

- **Database vs. page model**: Notion tasks live in a database (rows). The plan-to-tickets work is fundamentally creating database rows with properties, not creating standalone pages. Confirm the target is a database.
- **MCP tools you'll use**: confirm `notion-create-pages` (or equivalent) for inserting rows into a database, `notion-update-page` for edits.
- **Property types**: Notion properties are richly typed (title, text, select, multi-select, status, date, person, relation, rollup, formula). Phase / Effort / Blocked-by are usually `select` or `relation` properties. Document the property names and types for the target database.
- **Relations for dependencies**: Notion's native way to express "blocked by" is a `relation` property pointing at another row in the same database. This requires creating the rows first (Step 5) and then setting relation values (Step 6) — matches the skill's general pattern.
- **Rich text blocks vs. property values**: the "description" of a Notion task is often the page body (block children), not a property. To populate it, create the page with properties, then append blocks. Bulk markdown → blocks may require splitting on headings.
- **Status workflow**: Notion's `status` property type has named groups (To-do / In progress / Complete). Strings vary per database.
- **No native subtask field**: Notion subtasks are usually modeled as a `relation` to a parent row, sometimes with a rollup. Document the convention for the target database.
- **Known quirks**: anything that wasted time on the first run.

Until this file is filled in, fall back to the generic SKILL.md guidance and verify each step against Notion's actual behavior. The block-children API for descriptions is the biggest deviation from a typical task tracker.
