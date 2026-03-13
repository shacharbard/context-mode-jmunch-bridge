# Global CLAUDE.md Addition — context-mode

Append this to your `~/.claude/CLAUDE.md`, **after** the jCodeMunch and jDocMunch sections.

---

## Data File Navigation — context-mode (when available)

When context-mode MCP tools (`mcp__context-mode__*`) are available in a project, use them for **large data files** (JSON >100 lines, HTML >100 lines) and **large command outputs** instead of letting raw content flood the context window.

### Rules

- **Large JSON/HTML files (>100 lines):** Use `ctx_execute_file(path, language, code)` — file content available as `FILE_CONTENT` variable in code, raw content never enters context
- **Index a file for search:** Use `ctx_index(path="file.json", source="label")` — indexes without reading into context
- **Large Bash output:** Use `ctx_batch_execute(commands=[...], queries=[...])` — runs commands AND searches results in one call (queries is required)
- **Search previous outputs:** Use `ctx_search(queries=["terms"])` to find data from earlier in the session
- **Index external docs:** Use `ctx_fetch_and_index` for URLs, then `ctx_search` to query
- **Small config JSON** (package.json, tsconfig.json, <100 lines): Direct Read is fine
- context-mode does NOT replace jCodeMunch for code or jDocMunch for docs — it handles the data files and outputs they don't cover

### When Read is correct (not context-mode)

- Small JSON/HTML files (<100 lines)
- Config files (package.json, tsconfig.json, etc.)
- Files that need full context for editing (e.g., append-only files)
- Code files → use jCodeMunch instead
- Doc files → use jDocMunch instead
