---
name: session-migrator
description: |
  Migrate Claude Code sessions across directories so you can /resume conversations in a different project folder.
  Use this skill when the user wants to continue a conversation from one directory in another directory.

  Trigger on: /migrate-session, "迁移会话", "把会话移过去", "migrate session", "复制会话到",
  "在新目录resume", "把对话带过去", "move session", "copy session to", "带着对话去",
  "换目录继续", "把记忆带过去", "session搬过去", "想在那边resume", "那边能resume吗",
  "怎么在新项目继续对话", "会话怎么迁移"
---

# Session Migrator

Migrate Claude Code sessions between project directories so the user can `/resume` conversations in a different folder.

## How It Works

Claude Code stores sessions as `.jsonl` files under `~/.claude/projects/<sanitized-path>/`. Each session file contains `"cwd"` fields pointing to the original working directory. To migrate:

1. **Identify source** — Find the session file in the source project's directory
2. **Copy** — Copy the `.jsonl` file to the target project's directory
3. **Rewrite paths** — Replace all `"cwd":"<old-path>"` references with the new path
4. **Copy memory** — Also copy memory files (MEMORY.md, *.md) if they exist

## Implementation

### Step 1: Determine source and target

Ask the user if not clear:
- **Source**: The directory where the session was originally created (current directory if not specified)
- **Target**: The directory where they want to resume the session

### Step 2: Find session files

Session files are at: `~/.claude/projects/<sanitized-path>/`

Claude's sanitization rule: `:` becomes `--`, then all `/`, `\`, `_`, spaces, and other non-alphanumeric characters become a single `-`. Examples:
- `C:\Users\ASUS` → `C--Users-ASUS`
- `D:\pythonPycharms\Zync` → `D--pythonPycharms-Zync`
- `G:\Research_20250121\24AI for urban scientist` → `G--Research-20250121-24AI-for-urban-scientist`

Note: underscores `_` and spaces become `-`, NOT `--`. Only `:` becomes `--`.

**CRITICAL: Do NOT try to compute the sanitized path manually.** The safest approach is to first `cd` into the target directory and launch `claude` once (then exit). This forces Claude to create the correct sanitized directory. Then find it:

```bash
# Find source project dir by partial match
ls -d ~/.claude/projects/*keyword* 2>/dev/null

# Example: find the project dir for "057邮件处理"
ls -d ~/.claude/projects/*057* 2>/dev/null
```

For the **target** sanitized path, first launch claude in the target dir to let it create the correct directory, then find it:
```bash
ls -d ~/.claude/projects/*keyword* 2>/dev/null
```

If the target doesn't exist yet, create it by inferring the pattern from existing dirs. For ASCII-only paths, the rule is simple (`\` and `/` become `--`, `:` becomes `--`). For paths with Chinese/Unicode characters, the sanitization may encode them differently per platform — always verify by `ls`.

### Step 3: Select which session to migrate

List sessions sorted by modification time (newest first):
```bash
ls -lt ~/.claude/projects/<source-sanitized>/*.jsonl | head -10
```

By default, migrate the **most recent** session. If user asks for a specific one, let them choose.

### Step 4: Copy and rewrite (MUST use Python binary mode)

```bash
# Create target directory
mkdir -p ~/.claude/projects/<target-sanitized>/

# Copy session file
cp ~/.claude/projects/<source-sanitized>/<session-id>.jsonl \
   ~/.claude/projects/<target-sanitized>/<session-id>.jsonl
```

**CRITICAL: Do NOT use `sed` for path replacement.** JSONL files contain JSON-escaped paths with double backslashes, and non-ASCII characters (Chinese, etc.) are stored as raw UTF-8 bytes. `sed` will fail silently or corrupt the file. **Always use a Python script with binary mode (`'rb'`/`'wb'`):**

```python
import sys, re

session_file = sys.argv[1]

with open(session_file, 'rb') as f:
    content = f.read()

# Find all unique cwd values (as raw bytes)
pattern = rb'"cwd":"([^"]*)"'
cwds = set(re.findall(pattern, content))

# New path must be JSON-escaped (use \\\\ for each backslash)
new_path = b'G:\\\\Research_20250121\\\\24AI for urban scientist'

# Replace longest paths first to avoid partial matches
for old in sorted(cwds, key=len, reverse=True):
    old_full = b'"cwd":"' + old + b'"'
    new_full = b'"cwd":"' + new_path + b'"'
    content = content.replace(old_full, new_full)

with open(session_file, 'wb') as f:
    f.write(content)
```

**Why binary mode is required:**
- JSON files on Windows store paths like `D:\\pythonPycharms\\工具开发\\057邮件处理`
- The `\\` are literal two-character sequences in the file (backslash + backslash)
- Chinese characters are raw UTF-8 bytes (e.g., `工` = `\xe5\xb7\xa5`)
- Python text mode + string escaping = double-encoding chaos
- Binary mode (`rb`/`wb`) avoids all encoding issues by treating the file as raw bytes

### Step 5: Copy memory files (optional)

```bash
# Copy memory directory if it exists
if [ -d ~/.claude/projects/<source-sanitized>/memory ]; then
  cp -r ~/.claude/projects/<source-sanitized>/memory \
        ~/.claude/projects/<target-sanitized>/memory
fi

# Copy MEMORY.md if it exists at the project level
if [ -f ~/.claude/projects/<source-sanitized>/MEMORY.md ]; then
  cp ~/.claude/projects/<source-sanitized>/MEMORY.md \
     ~/.claude/projects/<target-sanitized>/MEMORY.md
fi
```

### Step 6: Verify

```bash
# Check the session exists in target
ls -la ~/.claude/projects/<target-sanitized>/*.jsonl

# Verify paths were rewritten (use Python to avoid encoding issues)
python -c "
import re, sys
with open(sys.argv[1], 'rb') as f:
    cwds = set(re.findall(rb'\"cwd\":\"([^\"]*)\"', f.read()))
for c in cwds:
    print(c.decode('utf-8', errors='replace'))
" ~/.claude/projects/<target-sanitized>/<session-id>.jsonl
```

Tell the user:
```
Session migrated! You can now:
  cd <target-path>
  claude
  /resume
```

## Edge Cases

- **Chinese/Unicode paths**: MUST use Python binary mode for replacement. Never use sed/awk/bash string replacement.
- **Multiple sessions**: If user wants all sessions, copy all `.jsonl` files and rewrite each
- **Already exists**: If a session with the same ID exists in target, warn before overwriting
- **Same directory**: If source and target are the same, no migration needed
- **Memory conflicts**: If target already has memory files, ask whether to merge or overwrite
- **Can't find sanitized dir**: Use `ls ~/.claude/projects/` and grep for keywords from the path

## Example Usage

User: "把当前会话迁移到 G:\Research\24AI for urban scientist"
Action: Find source dir via `ls *057*`, copy .jsonl, rewrite with Python binary script

User: "migrate session to /home/user/new-project"
Action: Same process for Unix paths (simpler, no backslash issues)

User: "我想在 Zync 目录继续这个对话"
Action: Detect current session, migrate to D--pythonPycharms-Zync
