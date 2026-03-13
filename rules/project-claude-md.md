# Project CLAUDE.md Addition — context-mode

Add this section to your project's `CLAUDE.md`, after the jCodeMunch and jDocMunch sections.

---

## Data File Navigation (context-mode)

For large JSON/HTML files produced by your project, use context-mode MCP tools instead of Read.

- **Large JSON (>100 lines):** `ctx_execute_file(path, language="python", code="...")` — file content available as `FILE_CONTENT` variable
- **Large HTML reports:** `ctx_execute_file` — same pattern, reference `FILE_CONTENT` in code
- **Index a file for search:** `ctx_index(path="file.json", source="label")` — indexes without reading into context
- **Pipeline command output:** `ctx_batch_execute(commands=[...], queries=[...])` — runs commands + searches results in one call
- **Search previous outputs:** `ctx_search(queries=["your terms"])`
- **Small config JSON** (package.json, tsconfig.json, etc.) uses direct Read — no sandboxing needed

Three-tier navigation summary:
1. Code files (.py/.ts/.tsx) → jCodeMunch
2. Doc files (.md/.mdx/.rst) → jDocMunch
3. Data files (.json/.html, large) → context-mode
