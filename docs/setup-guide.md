# Integrating context-mode with jCodeMunch & jDocMunch

A step-by-step guide to add [context-mode](https://github.com/mksglu/context-mode) alongside an existing [jCodeMunch](https://github.com/jgravelle/jcodemunch-mcp) + [jDocMunch](https://github.com/jgravelle/jdocmunch-mcp) setup in Claude Code.

## Prerequisites

You should already have jCodeMunch and jDocMunch working. If not, start with [jmunch-claude-code-setup](https://github.com/shacharbard/jmunch-claude-code-setup) first.

---

## Why Add context-mode?

jCodeMunch and jDocMunch save tokens by returning individual code symbols and doc sections. But they don't cover:

- **Command outputs** — test results, git log, build output, API responses flood the context window
- **Large JSON/HTML files** — pipeline outputs, generated reports, data files
- **Session persistence** — surviving context compaction with SQLite event storage
- **FTS5 search** — searching previous command outputs by relevance (BM25 ranking)

context-mode fills these gaps without conflicting with jCodeMunch/jDocMunch.

---

## Step 1: Register context-mode as MCP Server

### Global (all projects) — recommended

Add to `~/.claude/mcp.json`:

```json
{
  "context-mode": {
    "type": "stdio",
    "command": "npx",
    "args": ["-y", "context-mode"]
  }
}
```

> Merge this into your existing `mcpServers` object alongside jCodeMunch and jDocMunch.

### Project-level

Add to `.mcp.json` in your project root:

```json
{
  "mcpServers": {
    "context-mode": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "context-mode"]
    }
  }
}
```

> See `rules/mcp-example.json` for the full config with all three servers.

### Verify it connects

Restart Claude Code, then check:

```
claude mcp list
```

You should see `context-mode: npx -y context-mode - Connected`.

---

## Step 2: Disable context-mode's Own Hooks

**This is critical.** context-mode ships with PreToolUse hooks that intercept Read, Bash, WebFetch, and Grep. These will conflict with jCodeMunch/jDocMunch hooks.

**If you installed via `npx` (MCP only):** You're safe. MCP servers don't install hooks — only the Claude Code plugin does.

**If you installed as a Claude Code plugin:** Disable hook enforcement:

```bash
export CONTEXT_MODE_HOOKS_DISABLED=true
```

Or in plugin settings, disable "Enforce Hook Interception".

### Verify no context-mode hooks are registered

```bash
grep -r "context-mode" .claude/settings.json .claude/settings.local.json 2>/dev/null | grep -i hook
```

Should return empty. If it shows context-mode hooks, remove them.

---

## Step 3: Install the Bridge Hooks

Copy both hooks to your project:

```bash
cp hooks/context-mode-nudge.sh .claude/hooks/
cp hooks/context-mode-bash-nudge.sh .claude/hooks/
chmod +x .claude/hooks/context-mode-nudge.sh
chmod +x .claude/hooks/context-mode-bash-nudge.sh
```

### What each hook does

| Hook | Event | Effect |
|------|-------|--------|
| `context-mode-nudge.sh` | PreToolUse:Read | Blocks `Read` on large JSON/HTML files (>100 lines). Redirects to `ctx_execute_file`. |
| `context-mode-bash-nudge.sh` | PreToolUse:Bash | Blocks `Bash` on commands likely to produce large output (tests, unbounded git log/diff, curl, find, builds). Redirects to `ctx_execute`. |

### What `context-mode-nudge.sh` allows through

| File | Reason |
|------|--------|
| `package.json`, `tsconfig.json`, etc. | Small config files |
| Any JSON/HTML under 100 lines | Too small to benefit from sandboxing |
| Files in `.claude/`, `.vbw-planning/`, `node_modules/` | Config/planning directories |

### What `context-mode-bash-nudge.sh` allows through

| Command type | Examples |
|-------------|----------|
| Git state changes | `git add`, `git commit`, `git push`, `git checkout` |
| Git queries (bounded) | `git log -5`, `git log --oneline`, `git diff --stat` |
| Filesystem | `ls`, `mkdir`, `mv`, `cp`, `chmod`, `pwd` |
| Package management | `npm install`, `pip install`, `uv`, `npx` |
| Build/lint | `npm run build`, `npm run lint`, `npm run typecheck` |
| One-liners | `python -c "..."`, `echo`, `date`, `which` |
| File redirects | Any command with `> file` or `>> file` |

### What `context-mode-bash-nudge.sh` blocks

| Command type | Examples | Why |
|-------------|----------|-----|
| Test suites | `pytest`, `npm test`, `jest`, `vitest` | Output can be thousands of lines |
| Unbounded git | `git log`, `git diff` (no limits) | Full history/diff can be huge |
| Recursive search | `find .`, `grep -r`, `rg` | Thousands of matches |
| API calls | `curl`, `wget` | Response bodies can be massive |
| Verbose builds | `make`, `cargo build`, `tsc` | Compile output can be very long |

### How the hooks coexist with jCodeMunch/jDocMunch

Each hook checks **different criteria** — no overlaps, no conflicts:

```
jcodemunch-nudge.sh        -> .py, .ts, .tsx        (code files)
jdocmunch-nudge.sh         -> .md, .mdx, .rst       (doc files)
context-mode-nudge.sh      -> .json, .html, .htm    (data files)
context-mode-bash-nudge.sh -> large-output commands  (command outputs)
```

---

## Step 4: Register the Hooks

Add both hooks to your `.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Read",
        "hooks": [
          { "type": "command", "command": "bash .claude/hooks/jcodemunch-nudge.sh" },
          { "type": "command", "command": "bash .claude/hooks/jdocmunch-nudge.sh" },
          { "type": "command", "command": "bash .claude/hooks/context-mode-nudge.sh" }
        ]
      },
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "bash .claude/hooks/context-mode-bash-nudge.sh" }
        ]
      }
    ]
  }
}
```

> See `rules/settings-hook-example.json` for the full example. **Merge** with your existing hooks — don't replace them.

---

## Step 5: Allow context-mode Tools

Add these to your `.claude/settings.local.json` under `permissions.allow`:

```
mcp__context-mode__ctx_execute
mcp__context-mode__ctx_execute_file
mcp__context-mode__ctx_batch_execute
mcp__context-mode__ctx_search
mcp__context-mode__ctx_index
mcp__context-mode__ctx_fetch_and_index
mcp__context-mode__ctx_stats
mcp__context-mode__ctx_doctor
mcp__context-mode__ctx_upgrade
```

> See `rules/allowed-tools.txt` for the full list.

**Note:** The tool prefix uses a hyphen (`context-mode`), matching the server key in your MCP config. If you named the server differently, adjust accordingly.

---

## Step 6: Add CLAUDE.md Rules

### Global rules (all projects)

Append the contents of `rules/global-claude-md.md` to your `~/.claude/CLAUDE.md`, **after** the jCodeMunch and jDocMunch sections.

### Project rules

Append the contents of `rules/project-claude-md.md` to your project's `CLAUDE.md`.

> The key rule: "Code → jCodeMunch, Docs → jDocMunch, Data files → ctx_execute_file, Command outputs → ctx_execute."

---

## Step 7: Update Subagent Prompts

If you use the jCodeMunch/jDocMunch subagent instruction template from [jmunch-claude-code-setup](https://github.com/shacharbard/jmunch-claude-code-setup), add this block after it:

```
**Command & data navigation (MANDATORY):** Use context-mode MCP tools for large outputs.
- Test/build/search commands: mcp__context-mode__ctx_execute(language="shell", code="...") instead of Bash
- Large JSON (>100 lines): mcp__context-mode__ctx_execute_file instead of Read
- Large HTML: mcp__context-mode__ctx_execute_file instead of Read
- Pipeline output analysis: mcp__context-mode__ctx_batch_execute with queries parameter
- ctx_execute_file provides file content as FILE_CONTENT variable — do NOT use open() or sys.argv
- Bash only for: git status/add/commit/push, ls/mkdir/mv, package installs, file redirects
```

---

## Step 8: Verify the Integration

### Test 1: Small config JSON (should be allowed through Read)

Ask Claude to read `package.json`. The hook should let it through.

### Test 2: Large JSON (should be blocked from Read)

Ask Claude to read a JSON file with 100+ lines. The hook should block with context-mode instructions.

### Test 3: Small Bash command (should be allowed)

Ask Claude to run `git status`. Should execute normally via Bash.

### Test 4: Large-output Bash command (should be blocked)

Ask Claude to run `pytest`. The hook should block and redirect to `ctx_execute`.

### Test 5: jCodeMunch still works

Ask Claude to read a `.py` file. The jCodeMunch hook should fire, not the context-mode hook.

### Test 6: context-mode tools work

Ask Claude to run:
```
ctx_execute(language="python", code="print('hello from sandbox')")
```

### Run the automated tests

```bash
# ── Read hook tests ──

# Allow cases
echo '{"file_path":"package.json"}' | bash .claude/hooks/context-mode-nudge.sh >/dev/null 2>&1
echo "package.json: exit $?"

echo '{"file_path":"tsconfig.node.json"}' | bash .claude/hooks/context-mode-nudge.sh >/dev/null 2>&1
echo "tsconfig.node.json: exit $?"

echo '{"file_path":"src/pipeline.py"}' | bash .claude/hooks/context-mode-nudge.sh >/dev/null 2>&1
echo "pipeline.py: exit $?"

# Block case (create a large temp JSON)
TMPJSON=$(mktemp /tmp/test.XXXXXX.json)
python3 -c "import json; json.dump([{'id':i} for i in range(200)], open('$TMPJSON','w'), indent=2)"
echo '{"file_path":"'$TMPJSON'"}' | bash .claude/hooks/context-mode-nudge.sh >/dev/null 2>&1
echo "Large JSON: exit $?"
rm "$TMPJSON"

# ── Bash hook tests ──

# Allow cases
echo '{"tool_input":{"command":"git status"}}' | bash .claude/hooks/context-mode-bash-nudge.sh >/dev/null 2>&1
echo "git status: exit $?"

echo '{"tool_input":{"command":"ls -la"}}' | bash .claude/hooks/context-mode-bash-nudge.sh >/dev/null 2>&1
echo "ls -la: exit $?"

echo '{"tool_input":{"command":"git log -5"}}' | bash .claude/hooks/context-mode-bash-nudge.sh >/dev/null 2>&1
echo "git log -5: exit $?"

echo '{"tool_input":{"command":"npm install"}}' | bash .claude/hooks/context-mode-bash-nudge.sh >/dev/null 2>&1
echo "npm install: exit $?"

# Block cases
echo '{"tool_input":{"command":"pytest tests/"}}' | bash .claude/hooks/context-mode-bash-nudge.sh >/dev/null 2>&1
echo "pytest: exit $?"

echo '{"tool_input":{"command":"git log"}}' | bash .claude/hooks/context-mode-bash-nudge.sh >/dev/null 2>&1
echo "git log: exit $?"

echo '{"tool_input":{"command":"curl https://api.example.com/data"}}' | bash .claude/hooks/context-mode-bash-nudge.sh >/dev/null 2>&1
echo "curl: exit $?"

echo '{"tool_input":{"command":"find . -name *.py"}}' | bash .claude/hooks/context-mode-bash-nudge.sh >/dev/null 2>&1
echo "find: exit $?"
```

Expected output:
```
package.json: exit 0
tsconfig.node.json: exit 0
pipeline.py: exit 0
Large JSON: exit 2
git status: exit 0
ls -la: exit 0
git log -5: exit 0
npm install: exit 0
pytest: exit 2
git log: exit 2
curl: exit 2
find: exit 2
```

---

## context-mode Tool Reference

These are the tools context-mode provides. All tool names are prefixed with `mcp__context-mode__` by Claude Code.

| Tool | Parameters | Purpose |
|------|-----------|---------|
| `ctx_execute` | `language` (required), `code` (required), `timeout`, `intent` | Run code/commands in sandbox (11 languages including shell) |
| `ctx_execute_file` | `path` (required), `language` (required), `code` (required), `timeout`, `intent` | Process a file in sandbox — content available as `FILE_CONTENT` |
| `ctx_index` | `content` OR `path`, `source` | Index text/file into FTS5 for later search |
| `ctx_search` | `queries` (array), `limit`, `source` | Search indexed content with BM25 ranking |
| `ctx_batch_execute` | `commands` (required), `queries` (required), `timeout` | Run multiple commands + search results in one call |
| `ctx_fetch_and_index` | `url` (required), `source` | Fetch URL, auto-detect type, index for search |
| `ctx_stats` | — | Show context savings statistics |
| `ctx_doctor` | — | Run diagnostics |
| `ctx_upgrade` | — | Upgrade context-mode |

### Important: `FILE_CONTENT` variable

When using `ctx_execute_file`, the file content is injected as a variable called `FILE_CONTENT` in your code. Do **not** use `open()`, `sys.argv`, or `sys.stdin` to read the file — it's already available.

```python
# Correct
ctx_execute_file(path="data.json", language="python",
  code="import json; data=json.loads(FILE_CONTENT); print(len(data))")

# Wrong — will fail
ctx_execute_file(path="data.json", language="python",
  code="import json; data=json.load(open('data.json')); print(len(data))")
```

### Important: `ctx_execute` for shell commands

When using `ctx_execute` for shell commands, use `language="shell"`:

```python
# Run a test suite — output stays sandboxed
ctx_execute(language="shell", code="pytest tests/ -v")

# Unbounded git log — only filtered results enter context
ctx_execute(language="shell", code="git log --all")

# API call — response stays in sandbox
ctx_execute(language="shell", code="curl -s https://api.example.com/data")
```

---

## How the Four Tools Divide the Work

```
                    ┌─────────────────────┐
                    │   Claude Code        │
                    └─────────┬───────────┘
                              │
            ┌─────────────────┼─────────────────┐
            │                 │                 │
      Read request      Bash command      Other tools
            │                 │                 │
    ┌───────┼───────┐   ┌─────┼─────┐          │
    │       │       │   │           │          │
  .py/ts  .md/rst  .json  Small    Large      OK
    │       │    .html   cmd     output
    │       │       │     │         │
    ▼       ▼       ▼     ▼         ▼
 jCode   jDoc   ctx_     Bash   ctx_
 Munch   Munch  execute         execute
                _file           (shell)
```

No gaps. No conflicts. Each tool handles what it's best at:

- **jCodeMunch** — AST-level code navigation (returns one function, ~10-50 lines)
- **jDocMunch** — heading-level doc navigation (returns one section)
- **context-mode** — sandboxed command execution + data file processing + FTS5 search

---

## Step 9: Token Savings Tracking (Optional)

context-mode doesn't natively report token savings. This repo includes a PostToolUse hook that tracks genuine savings using an output-size proxy.

### Install the savings tracker

```bash
cp hooks/track-genuine-savings-ctx.sh .claude/hooks/
chmod +x .claude/hooks/track-genuine-savings-ctx.sh
```

### Register the PostToolUse hook

Add to `.claude/settings.json` under `hooks.PostToolUse`:

```json
{
  "matcher": "mcp__context-mode__*",
  "hooks": [
    {
      "type": "command",
      "command": "bash .claude/hooks/track-genuine-savings-ctx.sh"
    }
  ]
}
```

> See `rules/settings-hook-example.json` for the full example including both PreToolUse and PostToolUse hooks.

### What it tracks

The hook only counts **genuine** tools — those that replace a Read or Bash call:

| Tool | Baseline | What it replaces |
|------|----------|-----------------|
| `ctx_execute` (shell) | ~56KB (14K tokens) | Bash with large output |
| `ctx_execute_file` | Actual file size | Read on large JSON/HTML |
| `ctx_batch_execute` | ~240KB (60K tokens) | Multiple Bash calls |
| `ctx_fetch_and_index` | ~32KB (8K tokens) | WebFetch + Read |

Non-shell `ctx_execute` calls (Python one-liners, etc.) and utility tools (`ctx_stats`, `ctx_index`, `ctx_search`) are excluded.

### Where savings are stored

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

### Statusline display

If you use the [jmunch-claude-code-setup](https://github.com/shacharbard/jmunch-claude-code-setup) statusline, update `statusline.js` to read this file and display `CTX:58.375K` alongside `JCM:` and `JDM:` counters.

### Requires

- `python3` (the hook uses an inline Python script for JSON processing)
- `~/.code-index/` directory (created automatically on first run)

---

## Troubleshooting

**"Tool not found" errors for context-mode**
The MCP server may not be connected. Run `claude mcp list` and verify `context-mode` shows as Connected. If not, check your `~/.claude/mcp.json` or `.mcp.json` config.

**context-mode hooks overriding jCodeMunch**
You installed context-mode as a Claude Code plugin AND its hooks are active. Disable them:
```bash
export CONTEXT_MODE_HOOKS_DISABLED=true
```

**Large JSON being allowed through (not blocked)**
The hook checks line count (`wc -l`). Single-line JSON (minified) will pass through since it's under 100 lines. If your pipeline produces minified JSON, adjust the threshold or add a file-size check.

**Bash command incorrectly blocked**
Add the command pattern to the "ALWAYS ALLOW" section of `context-mode-bash-nudge.sh`. The allowlist uses `case` pattern matching — add a new pattern like `"your-command"*) exit 0 ;;`.

**Bash command not blocked when it should be**
Add a new detection pattern in the "BLOCK" section of `context-mode-bash-nudge.sh`. Set a new `IS_*` variable and add it to the final check.

**Hook blocks a file you need to edit**
Add the filename to the allowlist in `context-mode-nudge.sh` (the `BASENAME` check near line 20). Or use Read with a specific line range (`offset` and `limit` parameters) which bypasses the hook.

**Permission pattern doesn't match**
The tool prefix depends on your MCP server key. If you named it `contextmode` (no hyphen), the tools would be `mcp__contextmode__ctx_execute` etc. Match your permissions to your server key.
