# Integrating context-mode with jCodeMunch & jDocMunch

A step-by-step guide to add [context-mode](https://github.com/mksglu/context-mode) alongside an existing [jCodeMunch](https://github.com/jgravelle/jcodemunch-mcp) + [jDocMunch](https://github.com/jgravelle/jdocmunch-mcp) setup in Claude Code.

## Prerequisites

You should already have jCodeMunch and jDocMunch working. If not, start with [jmunch-claude-code-setup](https://github.com/shacharbard/jmunch-claude-code-setup) first.

---

## Why Add context-mode?

jCodeMunch and jDocMunch save tokens by never requesting full file content — they return individual functions and doc sections. But they don't cover:

- **Large JSON/HTML files** — pipeline outputs, generated reports, data files
- **Bash command output** — test results, pipeline runs, log analysis
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

## Step 3: Install the Bridge Hook

Copy `context-mode-nudge.sh` to your project:

```bash
cp hooks/context-mode-nudge.sh .claude/hooks/
chmod +x .claude/hooks/context-mode-nudge.sh
```

### What this hook does

When Claude tries to `Read` a large JSON or HTML file (100+ lines), the hook blocks the read and tells Claude to use context-mode instead. It gives file-type-specific instructions:

- **JSON files** → suggests `ctx_execute_file` with `json.loads(FILE_CONTENT)`
- **HTML files** → suggests `ctx_execute_file` with heading extraction code

### What this hook allows through

| File | Reason |
|------|--------|
| `package.json`, `tsconfig.json`, etc. | Small config files |
| Any JSON/HTML under 100 lines | Too small to benefit from sandboxing |
| Files in `.claude/`, `.vbw-planning/`, `node_modules/` | Config/planning directories |

### How it coexists with jCodeMunch/jDocMunch hooks

Each hook checks **different file extensions** — no overlaps, no conflicts:

```
jcodemunch-nudge.sh   -> .py, .ts, .tsx    (code files)
jdocmunch-nudge.sh    -> .md, .mdx, .rst   (doc files)
context-mode-nudge.sh -> .json, .html, .htm (data files)
```

All three hooks fire on every `Read` call, but only one (at most) will block for any given file.

---

## Step 4: Register the Hook

Add `context-mode-nudge.sh` to the `Read` matcher in your `.claude/settings.json`:

```json
{
  "matcher": "Read",
  "hooks": [
    { "type": "command", "command": "bash .claude/hooks/jcodemunch-nudge.sh" },
    { "type": "command", "command": "bash .claude/hooks/jdocmunch-nudge.sh" },
    { "type": "command", "command": "bash .claude/hooks/context-mode-nudge.sh" }
  ]
}
```

> See `rules/settings-hook-example.json` for the full example.

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

> The key rule: "Code → jCodeMunch, Docs → jDocMunch, Data → context-mode."

---

## Step 7: Update Subagent Prompts

If you use the jCodeMunch/jDocMunch subagent instruction template from [jmunch-claude-code-setup](https://github.com/shacharbard/jmunch-claude-code-setup), add this block after it:

```
**Data file navigation (MANDATORY for large files):** Use context-mode MCP tools for large JSON/HTML files.
- Large JSON (>100 lines): mcp__context-mode__ctx_execute_file instead of Read
- Large HTML: mcp__context-mode__ctx_execute_file instead of Read
- Pipeline output analysis: mcp__context-mode__ctx_batch_execute with queries parameter
- ctx_execute_file provides file content as FILE_CONTENT variable — do NOT use open() or sys.argv
```

---

## Step 8: Verify the Integration

### Test 1: Small config JSON (should be allowed)

Ask Claude to read `package.json`. The hook should let it through.

### Test 2: Large JSON (should be blocked)

Ask Claude to read a JSON file with 100+ lines. The hook should block with context-mode instructions.

### Test 3: jCodeMunch still works

Ask Claude to read a `.py` file. The jCodeMunch hook should fire, not the context-mode hook.

### Test 4: context-mode tools work

Ask Claude to run:
```
ctx_execute(language="python", code="print('hello from sandbox')")
```

### Run the automated test

```bash
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
```

Expected output:
```
package.json: exit 0
tsconfig.node.json: exit 0
pipeline.py: exit 0
Large JSON: exit 2
```

---

## context-mode Tool Reference

These are the tools context-mode provides. All tool names are prefixed with `mcp__context-mode__` by Claude Code.

| Tool | Parameters | Purpose |
|------|-----------|---------|
| `ctx_execute` | `language` (required), `code` (required), `timeout`, `intent` | Run code in sandbox (11 languages) |
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

---

## How the Three Tools Divide the Work

```
Read request arrives
  |
  ├── .py / .ts / .tsx ?
  |     └── jcodemunch-nudge.sh → BLOCKED, use get_symbol()
  |
  ├── .md / .mdx / .rst (large) ?
  |     └── jdocmunch-nudge.sh → BLOCKED, use search_sections()
  |
  ├── .json / .html (large) ?
  |     └── context-mode-nudge.sh → BLOCKED, use ctx_execute_file()
  |
  └── anything else ?
        └── ALLOWED (direct Read)
```

No gaps. No conflicts. Each tool handles what it's best at:

- **jCodeMunch** — AST-level code navigation (returns one function, ~10-50 lines)
- **jDocMunch** — heading-level doc navigation (returns one section)
- **context-mode** — sandboxed data file processing + output compression + session persistence

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

**Hook blocks a file you need to edit**
Add the filename to the allowlist in `context-mode-nudge.sh` (the `BASENAME` check near line 20). Or use Read with a specific line range (`offset` and `limit` parameters) which bypasses the hook.

**Permission pattern doesn't match**
The tool prefix depends on your MCP server key. If you named it `contextmode` (no hyphen), the tools would be `mcp__contextmode__ctx_execute` etc. Match your permissions to your server key.
