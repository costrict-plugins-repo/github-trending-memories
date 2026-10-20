# Memories - Claude Code Plugin

Session journaling for Claude Code. Captures what you did in each working session — descriptions, commands, files touched, and auto-generated tags. Browse, search, and filter your session history.

## Features

- **Auto-generated session summaries** — captures description, commands, files, and tags from your conversation
- **Manual descriptions** — provide your own summary when you prefer
- **Project-scoped & global** — memories default to the current project, or use `--global` for cross-project storage
- **Filtering** — search by tag, text, or project name
- **Tag auto-detection** — automatically tags sessions as `feature`, `refactor`, `bugfix`, `debug`, `test`, `build`, `docs`, `devops`, `config`, or `git`

## Installation

### From the Claude Code marketplace (recommended)

```
/plugin install memories
```

### Manual installation

1. Clone this repo:
   ```bash
   git clone https://github.com/4rneorist/memories-claude-plugin.git ~/.claude/plugins/local/memories
   ```

2. Add to `~/.claude/plugins/installed_plugins.json` inside the `"plugins"` object:
   ```json
   "memories@local": [
     {
       "scope": "user",
       "installPath": "<absolute-path-to-cloned-repo>",
       "version": "1.0.0",
       "installedAt": "<current-ISO-timestamp>"
     }
   ]
   ```

3. Add to `~/.claude/settings.json` inside `"enabledPlugins"`:
   ```json
   "memories@local": true
   ```

4. Restart Claude Code.

## Usage

### Save a session

```
/memories --add                          # Auto-generate description
/memories --add "Fixed auth bug"         # Provide your own description
/memories --add --global                 # Save to global (cross-project) storage
```

### List memories

```
/memories                                # List all project memories
/memories --global                       # List all global memories
/memories --tag bugfix                   # Filter by tag
/memories --search "auth"                # Search descriptions
/memories --project Guideline            # Filter by project (global scope)
/memories --global --tag bugfix --search "auth"   # Combine filters
```

### View & manage

```
/memories --open 3                       # View full session details by ID
/memories --delete 3                     # Delete a memory (with confirmation)
```

## Storage

Memories are stored as a JSON index + individual Markdown files:

**Project-scoped (default):**
```
~/.claude/projects/<project-key>/memories/
  index.json
  session-001.md
  session-002.md
```

**Global:**
```
~/.claude/memories/
  index.json
  session-001.md
```

Each session file is human-readable Markdown containing the description, files touched, and a command log.

## Auto-detected Tags

| Activity | Tag |
|----------|-----|
| New files created | `feature` |
| Existing files modified | `refactor` |
| Running tests | `test` |
| Building | `build` |
| Error investigation | `debug` |
| Markdown files edited | `docs` |
| Docker/CI files touched | `devops` |
| Config files touched | `config` |
| Git operations | `git` |
| Bug-related work | `bugfix` |

## License

MIT
