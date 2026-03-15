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

The sanitized path replaces path separators with `--` and removes drive colons. Examples:
- `C:\Users\ASUS` → `C--Users-ASUS`
- `D:\pythonPycharms\Zync` → `D--pythonPycharms-Zync`

To sanitize a path:
```bash
# On Windows (Git Bash)
echo "D:\pythonPycharms\Zync" | sed 's|\\|/|g' | sed 's|:|--|' | sed 's|/|--|g' | sed 's|^--||'
# Result: D--pythonPycharms-Zync
```

### Step 3: Select which session to migrate

List sessions sorted by modification time (newest first):
```bash
ls -lt ~/.claude/projects/<source-sanitized>/*.jsonl | head -10
```

By default, migrate the **most recent** session. If user asks for a specific one, let them choose.

### Step 4: Copy and rewrite

```bash
# Create target directory
mkdir -p ~/.claude/projects/<target-sanitized>/

# Copy session file
cp ~/.claude/projects/<source-sanitized>/<session-id>.jsonl \
   ~/.claude/projects/<target-sanitized>/<session-id>.jsonl

# Rewrite cwd paths (escape backslashes for JSON)
# Replace old path with new path in all "cwd":"..." fields
sed -i 's|"cwd":"<old-escaped>"|"cwd":"<new-escaped>"|g' \
   ~/.claude/projects/<target-sanitized>/<session-id>.jsonl
```

**Important**: JSON paths use double-escaped backslashes on Windows:
- `C:\Users\ASUS` in JSON is `C:\\Users\\ASUS`
- In sed, you need to escape again: `C:\\\\Users\\\\ASUS`

### Step 5: Copy memory files (optional)

```bash
# Copy memory directory if it exists
if [ -d ~/.claude/projects/<source-sanitized>/memory ]; then
  cp -r ~/.claude/projects/<source-sanitized>/memory \
        ~/.claude/projects/<target-sanitized>/memory
fi
```

### Step 6: Verify

```bash
# Check the session exists in target
ls -la ~/.claude/projects/<target-sanitized>/*.jsonl

# Verify paths were rewritten
grep -o '"cwd":"[^"]*"' ~/.claude/projects/<target-sanitized>/<session-id>.jsonl | sort -u
```

Tell the user:
```
Session migrated! You can now:
  cd <target-path>
  claude
  /resume
```

## Edge Cases

- **Multiple sessions**: If user wants all sessions, copy all `.jsonl` files and rewrite each
- **Already exists**: If a session with the same ID exists in target, warn before overwriting
- **Same directory**: If source and target are the same, no migration needed
- **Memory conflicts**: If target already has memory files, ask whether to merge or overwrite

## Example Usage

User: "把当前会话迁移到 D:\pythonPycharms\NewProject"
Action: Copy latest .jsonl from current project dir to D--pythonPycharms-NewProject, rewrite cwd paths

User: "migrate session to /home/user/new-project"
Action: Same process for Unix paths

User: "我想在 Zync 目录继续这个对话"
Action: Detect current session, migrate to D--pythonPycharms-Zync
