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

### Step 4: Copy and rewrite (MUST use Python binary mode + external script)

```bash
# Create target directory
mkdir -p ~/.claude/projects/<target-sanitized>/

# Copy session file
cp ~/.claude/projects/<source-sanitized>/<session-id>.jsonl \
   ~/.claude/projects/<target-sanitized>/<session-id>.jsonl
```

**CRITICAL: Do NOT use `sed` for path replacement.** JSONL files contain JSON-escaped paths with double backslashes, and non-ASCII characters (Chinese, etc.) are stored as raw UTF-8 bytes. `sed` will fail silently or corrupt the file.

#### The backslash escape hell problem

In JSON files, Windows path `G:\Research` is stored as `G:\\Research` — that's two literal bytes `5c 5c` in the file. When you try to construct replacement paths in Python:

- `b'G:\\\\Research'` in Python bytes literal = `G:\\Research` = bytes `47 3a 5c 5c 52...` ✅ CORRECT
- `'G:\\\\Research'.encode('utf-8')` in Python str = `G:\\Research` = bytes `47 3a 5c 5c 52...` ✅ CORRECT
- `'G:\\Research'.encode('utf-8')` in Python str = `G:\Research` = bytes `47 3a 5c 52...` ❌ WRONG (single backslash, breaks JSON)
- Inline `-c "..."` with shell: extra layer of escaping makes it nearly impossible to get right

**The ONLY reliable approach: write to an external .py file with `cat << 'PYEOF'` (single-quoted heredoc prevents shell expansion), and construct paths using `bytes.fromhex()` for backslashes.**

#### Recommended approach: extract-and-replace

Instead of manually constructing the full new path (error-prone), **extract the old cwd from the file, then modify it** (e.g., trim a suffix or replace a substring):

```bash
cat > /tmp/migrate_session.py << 'PYEOF'
import re, json, sys

session_file = sys.argv[1]
# old_suffix and new_suffix as hex-safe bytes (see below)

with open(session_file, 'rb') as f:
    content = f.read()

pattern = rb'"cwd":"([^"]*)"'
cwds = set(re.findall(pattern, content))

# Step 1: Print what we found (for debugging)
sys.stdout.reconfigure(encoding='utf-8')
print("Found cwd paths:")
for c in cwds:
    print(f"  hex: {c.hex(' ')}")
    print(f"  str: {c.decode('utf-8')}")

# Step 2: Build the suffix/substring to replace using bytes.fromhex()
# NEVER use Python string escapes for backslashes!
#
# To build a path segment like \\投稿\\building simulation:
#   - \\ in file = 5c5c (two bytes)
#   - 投稿 in UTF-8 = e68a95 e7a8bf
#   - Use: bytes.fromhex('5c5c') + bytes.fromhex('e68a95e7a8bf') + bytes.fromhex('5c5c') + b'building simulation'
#
# Or for pure ASCII paths like \\subfolder:
#   - bytes.fromhex('5c5c') + b'subfolder'

old_suffix = bytes.fromhex('5c5c') + '投稿'.encode('utf-8') + bytes.fromhex('5c5c') + b'building simulation'

for old_cwd in sorted(cwds, key=len, reverse=True):
    if old_cwd.endswith(old_suffix):
        new_cwd = old_cwd[:-len(old_suffix)]
        content = content.replace(
            b'"cwd":"' + old_cwd + b'"',
            b'"cwd":"' + new_cwd + b'"'
        )
        print(f"Replaced -> {new_cwd.decode('utf-8')}")

with open(session_file, 'wb') as f:
    f.write(content)

# Step 3: Verify JSON integrity
with open(session_file, 'rb') as f:
    lines = f.readlines()
errors = 0
for line in lines:
    if not line.strip():
        continue
    try:
        json.loads(line)
    except:
        errors += 1
print(f"JSON parse errors: {errors} / {len(lines)}")

# Step 4: Verify new cwd
with open(session_file, 'rb') as f:
    new_cwds = set(re.findall(pattern, f.read()))
for c in new_cwds:
    print(f"Final cwd: {c.decode('utf-8')}")
PYEOF

python /tmp/migrate_session.py "<target-session-file>"
```

#### Alternative: full path replacement

If source and target paths share no common prefix, replace the entire cwd value:

```bash
cat > /tmp/migrate_session.py << 'PYEOF'
import re, json, sys
sys.stdout.reconfigure(encoding='utf-8')

session_file = sys.argv[1]

with open(session_file, 'rb') as f:
    content = f.read()

pattern = rb'"cwd":"([^"]*)"'
cwds = set(re.findall(pattern, content))

# Build new path using bytes.fromhex for backslashes + .encode('utf-8') for text
# Example: D:\\pythonPycharms\\新项目
# CRITICAL: use bytes.fromhex('5c5c') for each \\ separator, NOT b'\\\\'
new_path = (
    b'D:' + bytes.fromhex('5c5c') +
    b'pythonPycharms' + bytes.fromhex('5c5c') +
    '新项目'.encode('utf-8')
)

for old in sorted(cwds, key=len, reverse=True):
    content = content.replace(
        b'"cwd":"' + old + b'"',
        b'"cwd":"' + new_path + b'"'
    )

with open(session_file, 'wb') as f:
    f.write(content)

# Verify (same as above)
with open(session_file, 'rb') as f:
    vf = f.read()
errors = sum(1 for line in vf.split(b'\n') if line.strip() and _safe_json(line))
new_cwds = set(re.findall(pattern, vf))
for c in new_cwds:
    print(f"Final cwd: {c.decode('utf-8')}")

def _safe_json(line):
    try:
        json.loads(line)
        return False
    except:
        return True
PYEOF
```

#### Key rules for path byte construction

| What you want in the file | How to build it in Python |
|---|---|
| `\\` (JSON-escaped backslash, bytes `5c 5c`) | `bytes.fromhex('5c5c')` |
| ASCII text like `Research` | `b'Research'` |
| Chinese text like `投稿` | `'投稿'.encode('utf-8')` |
| Full path `G:\\Research\\投稿` | `b'G:' + bytes.fromhex('5c5c') + b'Research' + bytes.fromhex('5c5c') + '投稿'.encode('utf-8')` |

**NEVER use Python string escapes (`\\\\`, `b'\\\\'`) for backslashes in path construction.** The number of escape layers (shell → Python string → bytes → JSON) makes it nearly impossible to get right. `bytes.fromhex('5c5c')` is unambiguous.

#### Mandatory verification

After rewriting, ALWAYS verify:
1. **JSON integrity**: parse every line with `json.loads()`, report error count (must be 0)
2. **cwd correctness**: extract and print all cwd values, confirm they match the target path
3. If JSON errors > 0, the file is corrupted — re-copy from source and retry

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
