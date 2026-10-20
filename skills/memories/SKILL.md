---
name: memories
description: Session journaling — save, list, open, search, and delete session memories. Use when user types /memories with any arguments.
argument-hint: "[--add [\"description\"]] [--open N] [--delete N] [--tag TAG] [--search \"text\"] [--project NAME] [--global]"
allowed-tools: [Read, Write, Bash, Glob, Grep]
---

# Memories

Session journaling skill. Captures what was done in a Claude Code session — description, commands, files touched, and auto-generated tags.

## Argument Parsing

Parse the arguments string provided by the user. The following flags are supported:

| Flag | Value | Description |
|------|-------|-------------|
| `--add` | optional `"description"` | Save current session as a memory. If a quoted string follows, use it as description. Otherwise auto-generate. |
| `--open` | `N` (integer) | Display the full session memory with ID N. |
| `--delete` | `N` (integer) | Delete the session memory with ID N (with confirmation). |
| `--tag` | `TAG` (string) | Filter memory list by tag (exact match). |
| `--search` | `"text"` (string) | Filter memory list by case-insensitive substring match on description. |
| `--project` | `NAME` (string) | Filter memory list by project name substring. Meaningful in global scope. |
| `--global` | (no value) | Use global storage (`~/.claude/memories/`) instead of project-scoped. |

**No flags at all** = list mode (show all memories for current scope).

**Flag combinations:**
- `--add` and `--global` can combine (save to global).
- `--tag`, `--search`, `--project`, `--global` can all combine for filtered listing.
- `--open` and `--delete` are standalone (only combined with `--global` for scope).

## Storage Path Resolution

Determine the storage directory based on scope:

**Project-scoped (default):**
1. Identify the current project key. The project key is derived from the current working directory path with slashes and colons replaced by hyphens. For example, `C:/Projects/Guideline/dev` becomes `C--Projects-Guideline-dev`.
2. Storage path: `C:/Users/4rneo/.claude/projects/<project-key>/memories/`

**Global (`--global` flag):**
1. Storage path: `C:/Users/4rneo/.claude/memories/`

**Initialization:**
1. Check if the storage directory exists. If not, create it:
   ```bash
   mkdir -p <storage-path>
   ```
2. Check if `index.json` exists in the storage directory using Read tool.
3. If `index.json` does not exist, create it with Write tool:
   ```json
   {
     "nextId": 1,
     "sessions": []
   }
   ```

## Flow: `--add`

When the user invokes `/memories --add` or `/memories --add "description"`:

### Step 1: Resolve storage path
Follow the Storage Path Resolution section above.

### Step 2: Read index.json
Use the Read tool to read `<storage-path>/index.json`. Parse the JSON content.

### Step 3: Collect session data
Review the entire conversation context and collect:

**Files touched** — gather full absolute paths from all tool calls in this conversation:
- Read tool calls → files read
- Edit tool calls → files modified
- Write tool calls → files created
- Bash tool calls that reference file paths → files involved

Deduplicate the list. Only include actual project files, not system/tool files.

**Commands & Responses** — for each tool call in the conversation, create a one-line summary:
- Format: `**ToolName** target — short result summary`
- Examples:
  - `**Read** C:/Projects/Guideline/dev/API/Program.cs — Reviewed startup configuration`
  - `**Edit** C:/Projects/Guideline/dev/API/Program.cs — Added Redis cache registration`
  - `**Bash** dotnet build — Build succeeded with 0 warnings`
  - `**Bash** dotnet test — 15 tests passed`
- Keep summaries under 80 characters each.
- Skip redundant consecutive reads of the same file.

**Description:**
- If the user provided a quoted description after `--add`, use it verbatim.
- If no description provided, auto-generate a 1-2 sentence summary of the session's work. Focus on WHAT was accomplished, not HOW.

**Tags** — auto-detect from session activity (max 5 tags):

| Signal | Tag |
|--------|-----|
| New files created via Write tool | `feature` |
| Existing files modified via Edit tool | `refactor` |
| `dotnet test`, `npm test`, `npm run test` in Bash commands | `test` |
| `dotnet build`, `npm run build` in Bash commands | `build` |
| Error investigation, reading stack traces, debugging | `debug` |
| `.md` files edited or created | `docs` |
| Docker, docker-compose, `.gitlab-ci.yml`, Dockerfile touched | `devops` |
| `appsettings`, `.env`, config files touched | `config` |
| Git commands (commit, merge, rebase, push) in Bash | `git` |
| Bug-related conversation context (fix, bug, issue, broken) | `bugfix` |

### Step 4: Assign ID and filename
- `id` = value of `nextId` from `index.json`
- `filename` = `session-` + id zero-padded to 3 digits + `.md` (e.g., `session-001.md`, `session-012.md`, `session-100.md`)

### Step 5: Show preview for confirmation
Display the generated memory to the user in this format:

```
**Memory Preview:**

**ID:** {id}
**Date:** {YYYY-MM-DD HH:mm}
**Project:** {project name}
**Tags:** {comma-separated tags}
**Description:** {description}
**Files:** {count} files touched
**Commands:** {count} entries

Save this memory? (yes/no)
```

Wait for the user to confirm.

### Step 6: Save on confirmation
If the user confirms:

1. Write the session markdown file to `<storage-path>/session-NNN.md` using the Write tool:

```markdown
# Session {id} — {description}

**Date:** {YYYY-MM-DD HH:mm}
**Project:** {project-name}
**Tags:** {comma-separated tags}

## Description
{description — expanded to a short paragraph if auto-generated}

## Files Touched
- `{full/path/to/file1}`
- `{full/path/to/file2}`

## Commands & Responses
1. {command-summary-1}
2. {command-summary-2}
...
```

2. Write the updated `index.json` using the Write tool — append the new session entry and increment `nextId`:

```json
{
  "nextId": {previous nextId + 1},
  "sessions": [
    ...existing sessions,
    {
      "id": {id},
      "timestamp": "{ISO 8601 timestamp}",
      "project": "{project-key}",
      "description": "{description}",
      "tags": [{tags array}],
      "files": [{files array}],
      "file": "session-NNN.md"
    }
  ]
}
```

3. Confirm to the user: `Memory #{id} saved.`

## Flow: List Memories

When the user invokes `/memories` with no action flags (no `--add`, `--open`, `--delete`), or with only filter flags (`--tag`, `--search`, `--project`, `--global`):

### Step 1: Resolve storage path
Follow the Storage Path Resolution section. Use global path if `--global` is present.

### Step 2: Read index.json
Use Read tool to read `<storage-path>/index.json`. Parse the JSON content.
If the file doesn't exist, display: `No memories found. Use /memories --add to save your first session.`

### Step 3: Apply filters
Start with the full `sessions` array, then filter in order:

1. **`--tag TAG`** — keep only sessions where `tags` array contains TAG (exact, case-insensitive match).
2. **`--search "text"`** — keep only sessions where `description` contains the text (case-insensitive substring).
3. **`--project NAME`** — keep only sessions where `project` contains NAME (case-insensitive substring).

### Step 4: Display results
If no sessions remain after filtering:
```
No memories found matching your filters.
```

Otherwise, display as a formatted table:

```
## Memories ({scope})

| ID | Date | Tags | Description |
|----|------|------|-------------|
| #1 | 2026-04-09 | refactor, auth | Implemented auth middleware refactor |
| #2 | 2026-04-08 | bugfix | Fixed tenant routing in ViewAPI |

{count} memories found. Use /memories --open N to view details.
```

Sort by ID descending (most recent first).

## Flow: `--open N`

When the user invokes `/memories --open N`:

### Step 1: Resolve storage path
Follow the Storage Path Resolution section. Use global path if `--global` is present.

### Step 2: Read index.json
Use Read tool to read `<storage-path>/index.json`. Parse the JSON.

### Step 3: Find the session
Look for a session entry where `id` equals N (as integer).

If not found, display:
```
Memory #{N} not found. Available IDs: {comma-separated list of all IDs}
```

### Step 4: Display the session
Use Read tool to read `<storage-path>/{session.file}`.
Display the full markdown contents in the conversation.

## Flow: `--delete N`

When the user invokes `/memories --delete N`:

### Step 1: Resolve storage path
Follow the Storage Path Resolution section. Use global path if `--global` is present.

### Step 2: Read index.json
Use Read tool to read `<storage-path>/index.json`. Parse the JSON.

### Step 3: Find the session
Look for a session entry where `id` equals N (as integer).

If not found, display:
```
Memory #{N} not found. Available IDs: {comma-separated list of all IDs}
```

### Step 4: Confirm deletion
Display:
```
Delete memory #{N}: "{session.description}"? This cannot be undone. (yes/no)
```

Wait for user confirmation.

### Step 5: Delete on confirmation
If the user confirms:

1. Remove the session entry from the `sessions` array in `index.json`.
2. Write the updated `index.json` using Write tool. Do NOT change `nextId` — IDs are never reused.
3. Delete the session markdown file:
   ```bash
   rm <storage-path>/{session.file}
   ```
4. Confirm: `Memory #{N} deleted.`

## Edge Cases

- **Missing storage directory** — always create it with `mkdir -p` before any read/write.
- **Missing `index.json`** — create fresh `{ "nextId": 1, "sessions": [] }` before proceeding.
- **Empty conversation on `--add`** — if no tool calls were made in the conversation, warn: `No commands detected in this session. Please provide a description: /memories --add "your description"`. If user provided a description, save it with empty commands/files lists.
- **ID gaps after deletion** — this is normal. `nextId` only increments, deleted IDs are not reused.
- **Large index** — if `sessions` array has 100+ entries in list mode, show only the 20 most recent and note: `Showing 20 most recent. Use --tag or --search to filter.`

## Routing

When this skill is invoked, determine the action from the arguments:

1. If `--add` is present → execute **Flow: `--add`**
2. If `--open N` is present → execute **Flow: `--open N`**
3. If `--delete N` is present → execute **Flow: `--delete N`**
4. Otherwise → execute **Flow: List Memories**

Execute exactly ONE flow per invocation. If conflicting action flags are present (e.g., `--add` and `--delete`), use the first one found in the order above.
