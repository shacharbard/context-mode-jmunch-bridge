# Project CLAUDE.md Addition — context-mode

Add this section to your project's `CLAUDE.md`, after the jCodeMunch and jDocMunch sections.

---

## context-mode MCP (MANDATORY when available)

context-mode saves tokens on **command outputs** and **data file reads** by running them in a sandbox — only filtered results enter your context window.

### Command Output Isolation (biggest savings)

Use `ctx_execute` instead of `Bash` for commands that produce large output:

- **Test suites:** `ctx_execute(language="shell", code="pytest ...")` — not `Bash("pytest ...")`
- **git log/diff (unbounded):** `ctx_execute(language="shell", code="git log ...")` — not `Bash("git log")`
- **Recursive search:** `ctx_execute(language="shell", code="find . -name ...")` — not `Bash("find ...")`
- **API calls:** `ctx_execute(language="shell", code="curl ...")` — not `Bash("curl ...")`
- **Build output:** `ctx_execute(language="shell", code="make ...")` — not `Bash("make ...")`

Outputs >5KB are automatically filtered by intent — only relevant portions enter context (98% token savings).

**When Bash IS correct:** git status/add/commit/push, file management (ls/mkdir/mv/cp), package installs, inline one-liners, commands with output redirected to a file (`> file`).

### Data File Navigation

- **Large JSON (>100 lines):** `ctx_execute_file(path, language="python", code="...")` — file content available as `FILE_CONTENT` variable
- **Large HTML reports:** `ctx_execute_file` — same pattern, reference `FILE_CONTENT` in code
- **Index a file for search:** `ctx_index(path="file.json", source="label")`
- **Search indexed content:** `ctx_search(queries=["your terms"])`
- **Batch operations:** `ctx_batch_execute(commands=[...], queries=[...])` — runs commands + searches in one call
- **Small config JSON** (package.json, tsconfig.json, etc.) uses direct Read

### Four-tier navigation summary

1. Code files (.py/.ts/.tsx) → jCodeMunch (`get_symbol`, `search_symbols`)
2. Doc files (.md/.mdx/.rst) → jDocMunch (`search_sections`, `get_section`)
3. Data files (.json/.html, large) → context-mode (`ctx_execute_file`)
4. Command outputs (tests, logs, builds) → context-mode (`ctx_execute`)
