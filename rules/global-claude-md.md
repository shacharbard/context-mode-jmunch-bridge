# Global CLAUDE.md Addition — context-mode

Append this to your `~/.claude/CLAUDE.md`, **after** the jCodeMunch and jDocMunch sections.

---

## Command Output & Data File Navigation — context-mode (when available)

When context-mode MCP tools (`mcp__context-mode__*`) are available, use them for **large command outputs** and **large data files** instead of letting raw content flood the context window.

### Command Output Isolation (primary use case)

Use `ctx_execute(language="shell", code="...")` instead of `Bash` for commands that produce large output:

- **Test suites:** `ctx_execute(language="shell", code="pytest ...")` — not `Bash("pytest ...")`
- **git log/diff (unbounded):** `ctx_execute(language="shell", code="git log ...")` — not `Bash("git log")`
- **Recursive search:** `ctx_execute(language="shell", code="find . -name ...")` — not `Bash("find ...")`
- **API calls:** `ctx_execute(language="shell", code="curl ...")` — not `Bash("curl ...")`
- **Build output:** `ctx_execute(language="shell", code="make ...")` — not `Bash("make ...")`

Outputs >5KB are automatically filtered by intent — only relevant portions enter context (98% savings).

**When Bash IS correct:** git status/add/commit/push, file management (ls/mkdir/mv/cp), package installs, inline one-liners, commands with output redirected to a file.

### Data File Rules

- **Large JSON/HTML files (>100 lines):** Use `ctx_execute_file(path, language, code)` — file content available as `FILE_CONTENT` variable in code, raw content never enters context
- **Index a file for search:** Use `ctx_index(path="file.json", source="label")` — indexes without reading into context
- **Batch operations:** Use `ctx_batch_execute(commands=[...], queries=[...])` — runs commands AND searches results in one call (queries is required)
- **Search previous outputs:** Use `ctx_search(queries=["terms"])` to find data from earlier in the session
- **Index external docs:** Use `ctx_fetch_and_index` for URLs, then `ctx_search` to query
- **Small config JSON** (package.json, tsconfig.json, <100 lines): Direct Read is fine

### Four-tier navigation

1. Code files (.py/.ts/.tsx) → jCodeMunch
2. Doc files (.md/.mdx/.rst) → jDocMunch
3. Data files (.json/.html, large) → context-mode (`ctx_execute_file`)
4. Command outputs (tests, logs, builds) → context-mode (`ctx_execute`)

### When Bash/Read is correct (not context-mode)

- Small commands with predictable output (git status, ls, pwd, echo)
- Git operations that modify state (add, commit, push, checkout)
- Package installs (npm install, pip install)
- Small JSON/HTML files (<100 lines) or config files
- Files that need full context for editing (e.g., append-only files)
- Code files → use jCodeMunch instead
- Doc files → use jDocMunch instead
