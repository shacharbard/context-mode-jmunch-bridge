# context-mode + jCodeMunch/jDocMunch Bridge for Claude Code

Hook, rules, and integration guide for running [context-mode](https://github.com/mksglu/context-mode) alongside [jCodeMunch](https://github.com/jgravelle/jcodemunch-mcp) and [jDocMunch](https://github.com/jgravelle/jdocmunch-mcp) in Claude Code — three MCP servers that together cover code, docs, and data files with zero conflicts.

> **Credit where it's due:**
> - [context-mode](https://github.com/mksglu/context-mode) is built and maintained by [mksglu](https://github.com/mksglu). It provides the sandbox execution, FTS5 indexing, and session persistence that make large data files and command outputs manageable.
> - [jCodeMunch](https://github.com/jgravelle/jcodemunch-mcp) and [jDocMunch](https://github.com/jgravelle/jdocmunch-mcp) are built and maintained by [J. Gravelle (jgravelle)](https://github.com/jgravelle). They provide the AST-level code navigation and section-level doc navigation that save 85-95% of tokens.
>
> This repo does not contain any of those MCP servers — it provides a bridge hook and configuration that makes all three work together. All the clever indexing, sandboxing, and symbol extraction is the work of their respective authors.

## The Problem

If you use [jmunch-claude-code-setup](https://github.com/shacharbard/jmunch-claude-code-setup), Claude Code already enforces jCodeMunch for code files and jDocMunch for doc files. But there's a gap:

| File type | What handles it | Token savings |
|-----------|----------------|---------------|
| `.py`, `.ts`, `.tsx` | jCodeMunch | ~85-95% |
| `.md`, `.mdx`, `.rst` | jDocMunch | ~90-95% |
| **`.json`, `.html` (large)** | **Nothing — full Read** | **0%** |
| **Bash output (large)** | **Nothing — floods context** | **0%** |

context-mode fills these gaps with sandboxed execution and FTS5 search — keeping raw data out of the context window entirely.

## What Each Tool Does

- **[jCodeMunch](https://github.com/jgravelle/jcodemunch-mcp)** (by jgravelle) — indexes code (Python, TypeScript, 15 languages) and returns individual functions via AST parsing. You get one function instead of the whole file.
- **[jDocMunch](https://github.com/jgravelle/jdocmunch-mcp)** (by jgravelle) — indexes docs (.md, .mdx, .rst) and returns specific sections by heading hierarchy. You get one section instead of the whole document.
- **[context-mode](https://github.com/mksglu/context-mode)** (by mksglu) — sandboxes file reads and command outputs into SQLite with FTS5 search. Raw content stays in the sandbox; only your analysis output enters the context window.

These are **complementary, not competing** tools:

| Capability | jCodeMunch | jDocMunch | context-mode |
|---|---|---|---|
| Code symbol navigation (AST) | **Best** | — | — |
| Doc section navigation (headings) | — | **Best** | — |
| Large JSON/HTML processing | — | — | **Best** |
| Bash output sandboxing | — | — | **Best** |
| FTS5 search (BM25 ranking) | — | — | **Best** |
| Session persistence (SQLite) | — | — | **Best** |

## How It Works

This repo provides one hook — `context-mode-nudge.sh` — that completes the coverage:

```
Read request arrives
  |
  ├── .py / .ts / .tsx ?
  |     └── jcodemunch-nudge.sh → BLOCKED, use get_symbol()
  |
  ├── .md / .mdx / .rst (large) ?
  |     └── jdocmunch-nudge.sh → BLOCKED, use search_sections()
  |
  ├── .json / .html / .htm (large) ?
  |     └── context-mode-nudge.sh → BLOCKED, use ctx_execute_file()
  |
  └── anything else ?
        └── ALLOWED (direct Read)
```

Each hook checks **different file extensions**. No overlaps. No hook ordering issues. No conflicts.

## Quick Start

**Prerequisite:** You should already have jCodeMunch + jDocMunch working via [jmunch-claude-code-setup](https://github.com/shacharbard/jmunch-claude-code-setup).

```bash
# 1. Register context-mode as MCP server (global)
# Add to ~/.claude/mcp.json under mcpServers:
#   "context-mode": { "type": "stdio", "command": "npx", "args": ["-y", "context-mode"] }

# 2. Copy the bridge hook
cp hooks/context-mode-nudge.sh .claude/hooks/
chmod +x .claude/hooks/context-mode-nudge.sh

# 3. Register the hook (merge into .claude/settings.json Read matcher)
# See rules/settings-hook-example.json

# 4. Allow context-mode tools (add to .claude/settings.local.json)
# See rules/allowed-tools.txt

# 5. Add CLAUDE.md rules
# See rules/global-claude-md.md and rules/project-claude-md.md
```

See [docs/setup-guide.md](docs/setup-guide.md) for the full step-by-step walkthrough.

## Repository Structure

```
hooks/
  context-mode-nudge.sh            # PreToolUse:Read — blocks Read on large .json/.html files
rules/
  global-claude-md.md              # CLAUDE.md rules for ~/.claude/CLAUDE.md
  project-claude-md.md             # CLAUDE.md rules for project CLAUDE.md
  settings-hook-example.json       # Hook registration (merge with existing settings)
  mcp-example.json                 # MCP config with all three servers
  allowed-tools.txt                # Context-mode tool allowlist
docs/
  setup-guide.md                   # Full step-by-step walkthrough
```

## What the Hook Allows Through

The `context-mode-nudge.sh` hook is not a blanket block. It allows:

| File | Why |
|------|-----|
| `package.json`, `tsconfig.json`, `config.json`, etc. | Small config files — always allowed |
| Any JSON/HTML under 100 lines | Too small to benefit from sandboxing |
| Files in `.claude/`, `.vbw-planning/`, `node_modules/` | Config/planning directories |

## Important: `FILE_CONTENT` Variable

When context-mode's `ctx_execute_file` processes a file, it injects the content as a variable called `FILE_CONTENT`. Do **not** use `open()` or `sys.argv` — the content is already available:

```python
# Correct — FILE_CONTENT is pre-loaded by context-mode
ctx_execute_file(path="data.json", language="python",
  code="import json; data=json.loads(FILE_CONTENT); print(len(data), 'entries')")

# Wrong — will fail
ctx_execute_file(path="data.json", language="python",
  code="import json; data=json.load(open('data.json')); print(len(data))")
```

## Subagent Instructions

If you use the jCodeMunch/jDocMunch subagent template from [jmunch-claude-code-setup](https://github.com/shacharbard/jmunch-claude-code-setup), add this block after it:

```
**Data file navigation (MANDATORY for large files):** Use context-mode MCP tools for large JSON/HTML files.
- Large JSON (>100 lines): mcp__context-mode__ctx_execute_file instead of Read
- Large HTML: mcp__context-mode__ctx_execute_file instead of Read
- Pipeline output analysis: mcp__context-mode__ctx_batch_execute with queries parameter
- ctx_execute_file provides file content as FILE_CONTENT variable — do NOT use open() or sys.argv
```

## Why Not Just Use context-mode for Everything?

context-mode sandboxes entire files — it reads the full content into a subprocess. For a 500-line Python file, that's 500 lines loaded into the sandbox just to extract one function.

jCodeMunch returns just that one function (~10-50 lines) via AST parsing. That's 10x fewer tokens even compared to context-mode's sandbox approach.

Similarly, jDocMunch returns one doc section instead of the whole document. For code and docs, the specialized tools are simply cheaper.

context-mode shines where jCodeMunch/jDocMunch have no coverage: JSON, HTML, command outputs, session persistence, and FTS5 search.

## Requirements

- [Claude Code](https://claude.ai/claude-code) CLI
- [jmunch-claude-code-setup](https://github.com/shacharbard/jmunch-claude-code-setup) (prerequisite)
- Node.js / npm (for `npx context-mode`)
- `python3` (hook input parsing)

## Credits

- [context-mode](https://github.com/mksglu/context-mode) by [mksglu](https://github.com/mksglu)
- [jCodeMunch MCP](https://github.com/jgravelle/jcodemunch-mcp) by [jgravelle](https://github.com/jgravelle)
- [jDocMunch MCP](https://github.com/jgravelle/jdocmunch-mcp) by [jgravelle](https://github.com/jgravelle)
- [jmunch-claude-code-setup](https://github.com/shacharbard/jmunch-claude-code-setup) for the companion enforcement stack

## License

The hook, rules, and documentation in this repository are licensed under the [MIT License](LICENSE).

This repo does **not** include context-mode, jCodeMunch, or jDocMunch — only configuration and a bridge hook that makes them work together. The MCP servers are separate projects by their respective authors and are subject to their own licenses:
- context-mode: [ELv2 (Elastic License v2)](https://github.com/mksglu/context-mode/blob/main/LICENSE)
- jCodeMunch: see [jcodemunch-mcp](https://github.com/jgravelle/jcodemunch-mcp)
- jDocMunch: see [jdocmunch-mcp](https://github.com/jgravelle/jdocmunch-mcp)
