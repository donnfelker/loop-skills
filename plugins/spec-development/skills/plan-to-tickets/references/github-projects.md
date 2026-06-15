# GitHub Projects — platform-specific notes

*Stub — fill this in the first time you run the plan-to-tickets skill against GitHub Projects.*

When you do, capture these specifics (mirror the structure of `clickup.md`):

- **GraphQL, not REST**: GitHub Projects v2 is GraphQL-only. The MCP wraps this, but field names, item types, and mutation patterns differ from the REST issues API.
- **Items vs. issues**: a project item can wrap an issue, a PR, or be a draft item that lives only in the project. Decide which to create — usually issues (so they're linked to the repo) unless the plan is project-internal.
- **Custom fields**: GitHub Projects v2 supports single-select, text, number, date, and iteration custom fields. Phase / Effort / Status are usually modeled as single-select fields with pre-defined options. Document the field IDs and option IDs for the target project.
- **No native dependencies**: GitHub Projects doesn't have first-class dependency edges. The options are: (a) put `Blocked by: #123` in the issue body and rely on humans/agents reading it, (b) use task lists (`- [ ] #123`) which GitHub auto-links, or (c) use a third-party app like ZenHub. Document which approach this project uses.
- **Hierarchy via task lists**: GitHub renders nested issue references in task lists as a kind of parent-child. Useful for phase→task mapping if no other hierarchy field exists.
- **Repo-scoped vs. org-scoped projects**: org projects span multiple repos; repo projects don't. Affects which repo to create issues in.
- **Known quirks**: anything that wasted time on the first run — especially around GraphQL field/option ID lookup, which is verbose.

Until this file is filled in, fall back to the generic SKILL.md guidance and verify each step against GitHub Projects' actual behavior.
