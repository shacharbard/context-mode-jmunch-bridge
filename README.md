# context-mode + jCodeMunch/jDocMunch Bridge for Claude Code

Hooks, rules, and integration guide for running [context-mode](https://github.com/mksglu/context-mode) alongside [jCodeMunch](https://github.com/jgravelle/jcodemunch-mcp) and [jDocMunch](https://github.com/jgravelle/jdocmunch-mcp) in Claude Code — three MCP servers that together cover code, docs, data files, and command outputs with zero conflicts.

> **Credit where it's due:**
> - [context-mode](https://github.com/mksglu/context-mode) is built and maintained by [mksglu](https://github.com/mksglu). It provides the sandbox execution, FTS5 indexing, and session persistence that make large data files and command outputs manageable.
> - [jCodeMunch](https://github.com/jgravelle/jcodemunch-mcp) and [jDocMunch](https://github.com/jgravelle/jdocmunch-mcp) are built and maintained by [J. Gravelle (jgravelle)](https://github.com/jgravelle). They provide the AST-level code navigation and section-level doc navigation that save 85-95% of tokens.
>
> This repo does not contain any of those MCP servers — it provides bridge hooks and configuration that makes all three work together. All the clever indexing, sandboxing, and symbol extraction is the work of their respective authors.

## The Problem

If you use [jmunch-claude-code-setup](https://github.com/shacharbard/jmunch-claude-code-setup), Claude Code already enforces jCodeMunch for code files and jDocMunch for doc files. But there are two gaps:

| What | What handles it | Token savings |
|------|----------------|---------------|
| Code files (.py/.ts/.tsx) | jCodeMunch | ~85-95% |
| Doc files (.md/.mdx/.rst) | jDocMunch | ~90-95% |
| **Data files (.json/.html, large)** | **Nothing — full Read** | **0%** |
| **Command outputs (tests, logs, builds)** | **Nothing — floods context** | **0%** |

context-mode fills both gaps — sandboxed file processing AND command output isolation — keeping raw data out of the context window entirely.

## What Each Tool Does

- **[jCodeMunch](https://github.com/jgravelle/jcodemunch-mcp)** (by jgravelle) — indexes code (Python, TypeScript, 15 languages) and returns individual functions via AST parsing. You get one function instead of the whole file.
- **[jDocMunch](https://github.com/jgravelle/jdocmunch-mcp)** (by jgravelle) — indexes docs (.md, .mdx, .rst) and returns specific sections by heading hierarchy. You get one section instead of the whole document.
- **[context-mode](https://github.com/mksglu/context-mode)** (by mksglu) — sandboxes file reads and command outputs into SQLite with FTS5 search. Raw content stays in the sandbox; only your filtered analysis enters the context window.

These are **complementary, not competing** tools:

| Capability | jCodeMunch | jDocMunch | context-mode |
|---|---|---|---|
| Code symbol navigation (AST) | **Best** | — | — |
| Doc section navigation (headings) | — | **Best** | — |
| Large JSON/HTML processing | — | — | **Best** |
| Command output isolation | — | — | **Best** |
| FTS5 search (BM25 ranking) | — | — | **Best** |
| Session persistence (SQLite) | — | — | **Best** |

## How It Works

This repo provides **two hooks** that complete the four-tier coverage:

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

Bash command arrives
  |
  ├── Small/safe command (git status, ls, mkdir, echo) ?
  |     └── ALLOWED (direct Bash)
  |
  ├── Large-output command (pytest, git log, curl, find, make) ?
  |     └── context-mode-bash-nudge.sh → BLOCKED, use ctx_execute()
  |
  └── Package installs, file redirects, infrastructure ?
        └── ALLOWED (direct Bash)
```

Each hook checks **different criteria**. No overlaps. No hook ordering issues. No conflicts.

## Quick Start

**Prerequisite:** You should already have jCodeMunch + jDocMunch working via [jmunch-claude-code-setup](https://github.com/shacharbard/jmunch-claude-code-setup).

```bash
# 1. Register context-mode as MCP server (global)
# Add to ~/.claude/mcp.json under mcpServers:
#   "context-mode": { "type": "stdio", "command": "npx", "args": ["-y", "context-mode"] }

# 2. Copy both bridge hooks
cp hooks/context-mode-nudge.sh .claude/hooks/
cp hooks/context-mode-bash-nudge.sh .claude/hooks/
chmod +x .claude/hooks/context-mode-nudge.sh
chmod +x .claude/hooks/context-mode-bash-nudge.sh

# 3. Register the hooks (merge into .claude/settings.json)
# See rules/settings-hook-example.json

# 4. Allow context-mode tools (add to .claude/settings.local.json)
# See rules/allowed-tools.txt

# 5. Add CLAUDE.md rules
# See rules/global-claude-md.md and rules/project-claude-md.md
```

See [docs/setup-guide.md](docs/setup-guide.md) for the full step-by-step walkthrough.

## Token Savings Tracking

context-mode does not natively report token savings. This repo includes a **PostToolUse hook** that measures genuine savings using an output-size proxy:

- For `ctx_execute_file`: compares actual file size vs. what entered context
- For `ctx_execute` (shell): compares a conservative baseline (~56KB) vs. output
- For `ctx_batch_execute`: compares ~240KB baseline vs. output
- For `ctx_fetch_and_index`: compares ~32KB baseline vs. output

Only genuine replacement tools are tracked (tools that replace a Read or Bash call). Utility tools like `ctx_stats`, `ctx_index`, and `ctx_search` are excluded.

### How it works

```
ctx_execute("pytest tests/")
  → Full output: ~56KB (stays in sandbox)
  → Filtered output entering context: ~2KB
  → Tokens saved: ~13,500

ctx_execute_file("data.json", ...)
  → File size: 180KB
  → Summary entering context: ~500 bytes
  → Tokens saved: ~44,875
```

Savings accumulate in `~/.code-index/_genuine_savings_ctx.json`:

```json
{
  "total_genuine_tokens_saved": 58375,
  "by_tool": {
    "mcp__context-mode__ctx_execute": 13500,
    "mcp__context-mode__ctx_execute_file": 44875
  },
  "call_counts": {
    "mcp__context-mode__ctx_execute": 3,
    "mcp__context-mode__ctx_execute_file": 2
  }
}
```

### Statusline integration

If you use the jmunch-claude-code-setup statusline, it displays savings as split counters:

```
Claude Opus 4 │ JCM:920.376K JDM:2.108K CTX:58.375K │ leaflet-analysis
```

- **JCM** — jCodeMunch savings (code files)
- **JDM** — jDocMunch savings (doc files)
- **CTX** — context-mode savings (data files + command outputs)

## Repository Structure

```
hooks/
  context-mode-nudge.sh            # PreToolUse:Read — blocks Read on large .json/.html files
  context-mode-bash-nudge.sh       # PreToolUse:Bash — blocks Bash on large-output commands
  track-genuine-savings-ctx.sh     # PostToolUse — tracks token savings from context-mode
rules/
  global-claude-md.md              # CLAUDE.md rules for ~/.claude/CLAUDE.md
  project-claude-md.md             # CLAUDE.md rules for project CLAUDE.md
  settings-hook-example.json       # Hook registration (merge with existing settings)
  mcp-example.json                 # MCP config with all three servers
  allowed-tools.txt                # Context-mode tool allowlist
docs/
  setup-guide.md                   # Full step-by-step walkthrough
```

## What the Read Hook Allows Through

The `context-mode-nudge.sh` hook is not a blanket block. It allows:

| File | Why |
|------|-----|
| `package.json`, `tsconfig.json`, `config.json`, etc. | Small config files — always allowed |
| Any JSON/HTML under 100 lines | Too small to benefit from sandboxing |
| Files in `.claude/`, `.vbw-planning/`, `node_modules/` | Config/planning directories |

## What the Bash Hook Allows Through

The `context-mode-bash-nudge.sh` hook allows all small/safe commands:

| Command type | Examples | Why allowed |
|-------------|----------|-------------|
| Git state changes | `git add`, `git commit`, `git push` | Side effects, not data output |
| Git queries (bounded) | `git log -5`, `git diff --stat` | Output is small and bounded |
| Filesystem | `ls`, `mkdir`, `mv`, `cp`, `chmod` | Predictable small output |
| Package management | `npm install`, `pip install`, `uv` | Progress output, not data |
| Build commands | `npm run build`, `npm run lint` | Side effects, not data |
| File redirects | Any command with `> file` | Output goes to file, not context |

It **blocks** commands with potentially large output:

| Command type | Examples | Why blocked |
|-------------|----------|-------------|
| Test suites | `pytest`, `npm test`, `jest`, `vitest` | Test output can be thousands of lines |
| Unbounded git | `git log`, `git diff` (no limits) | Full history/diff is huge |
| Recursive search | `find .`, `grep -r`, `rg` | Can return thousands of matches |
| API calls | `curl`, `wget` | Response bodies can be massive |
| Verbose builds | `make`, `cargo build`, `tsc` | Compile output can be very long |

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
**Command & data navigation (MANDATORY):** Use context-mode MCP tools for large outputs.
- Test/build/search commands: mcp__context-mode__ctx_execute(language="shell", code="...") instead of Bash
- Large JSON (>100 lines): mcp__context-mode__ctx_execute_file instead of Read
- Large HTML: mcp__context-mode__ctx_execute_file instead of Read
- Pipeline output analysis: mcp__context-mode__ctx_batch_execute with queries parameter
- ctx_execute_file provides file content as FILE_CONTENT variable — do NOT use open() or sys.argv
- Bash only for: git status/add/commit/push, ls/mkdir/mv, package installs, file redirects
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

The hooks, rules, and documentation in this repository are licensed under the [MIT License](LICENSE).

This repo does **not** include context-mode, jCodeMunch, or jDocMunch — only configuration and bridge hooks that make them work together. The MCP servers are separate projects by their respective authors and are subject to their own licenses:
- context-mode: [ELv2 (Elastic License v2)](https://github.com/mksglu/context-mode/blob/main/LICENSE)
- jCodeMunch: see [jcodemunch-mcp](https://github.com/jgravelle/jcodemunch-mcp)
- jDocMunch: see [jdocmunch-mcp](https://github.com/jgravelle/jdocmunch-mcp)
