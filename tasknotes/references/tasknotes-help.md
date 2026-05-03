# TaskNotes - Setup and Configuration Guide

Setup instructions, full API reference, and troubleshooting for all TaskNotes access methods. The tasknotes skill reads this file when connection or setup problems arise and uses it to guide users through fixes.

**Plugin docs:** https://tasknotes.dev/
**HTTP API docs:** https://tasknotes.dev/HTTP_API/
**MCP / integrations:** https://tasknotes.dev/features/integrations/

---

## Access methods overview

| Method | Requires | Body content | Best for |
|---|---|---|---|
| MCP server | Obsidian running + toggles on | Frontmatter only (see note) | Primary - full task ops |
| HTTP API | Obsidian running + API toggle on | Frontmatter only (see note) | Fallback or scripting |
| Filesystem | Any filesystem tool | Full file including body | Body reads; Obsidian closed |
| `mtn` CLI | Node.js + npm | Full file including body | Headless; scripting |

**Body content note:** Both MCP and HTTP API read from Obsidian's metadata cache, which only indexes frontmatter. Task body content (checklists, notes, acceptance criteria) is not returned by any GET operation. Body reads require a direct file read. This is a known upstream gap (GitHub issue #1858).

---

## Enabling the MCP server and HTTP API

Both run inside the TaskNotes Obsidian plugin (v4.5.1+). They are only available while Obsidian is running.

**In Obsidian:**
1. Settings → TaskNotes → Integrations
2. Enable the **HTTP API** toggle
3. Enable the **MCP Server** toggle
4. Note the port (default: 8080)
5. Restart Obsidian

**Verify:** Open `http://localhost:8080/api/health` in a browser. Should return `{"status":"ok","vault":"...","version":"..."}`.

---

## Connecting an agent to the MCP server

The MCP server exposes an SSE endpoint at `http://localhost:{port}/mcp`. How you connect depends on your agent platform.

### Claude Desktop

Edit `claude_desktop_config.json`:

- **Windows:** `C:\Users\{username}\AppData\Roaming\Claude\claude_desktop_config.json`
- **macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`

```json
{
  "mcpServers": {
    "tasknotes": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "http://localhost:8080/mcp"]
    }
  }
}
```

Restart Claude Desktop after editing.

### Other agent platforms

MCP setup varies by platform. The transport is HTTP/SSE; point your agent at `http://localhost:{port}/mcp`.

- **Claude Code:** https://docs.claude.ai/en/docs/claude-code/mcp
- **Cursor:** https://docs.cursor.com/advanced/model-context-protocol
- **Windsurf / Codex CLI / other agents:** check your platform's MCP or tool-server documentation
- **Generic MCP spec:** https://modelcontextprotocol.io/

All platforms that support MCP can connect using the same SSE URL. The `mcp-remote` npm package (used above for Claude Desktop) is a convenience wrapper; native SSE support does not need it.

---

## Finding the configured port

If port 8080 is in use, change it in Obsidian: Settings → TaskNotes → Integrations → HTTP API → Port.

Update the port in your agent config to match.

---

## Default folder locations and filesystem scope

**Where TaskNotes creates its folders by default:** The plugin creates `TaskNotes/Tasks/` and `TaskNotes/Views/` directly under the Obsidian vault root - not inside any subfolder.

Settings → TaskNotes → General → **Default tasks folder** controls where new task files are written. The plugin's default value is `TaskNotes/Tasks`.

**Auto-create task folder:** Settings → TaskNotes → General has an option to automatically create the task folder on startup if it does not exist. When enabled, Obsidian will recreate the folder at vault root even if you have moved it elsewhere. **If you move your TaskNotes folder inside a narrower filesystem scope, disable this toggle** - otherwise the plugin will recreate an empty folder at vault root and task creation may silently route to the wrong location.

Reference: https://tasknotes.dev/settings/general/

### If your agent's filesystem scope is narrower than the vault root

Many agent setups scope the filesystem tool to a subdirectory rather than the full vault (e.g., an `Agent Access/` folder rather than the full `Obsidian/` vault root). In this case, the default `TaskNotes/` folder at vault root is outside the agent's reach and the filesystem fallback will not work.

**To bring TaskNotes into scope:**

1. In Obsidian, create a `TaskNotes/` folder inside your scoped directory (e.g., `Agent Access/TaskNotes/`)
2. Move your existing `TaskNotes/Tasks/` and `TaskNotes/Views/` contents into it
3. Update Settings → TaskNotes → General → Default tasks folder to the new path (e.g., `Agent Access/TaskNotes/Tasks`)
4. **Disable auto-create task folder** so the plugin does not recreate the folder at vault root
5. Update `tasks_folder` in `tasknotes-config.md` to match (see section below)

**Archive folder:** If Settings → TaskNotes → General → Move Archived Tasks to Folder is enabled, the plugin moves archived tasks to a subfolder. The default is typically a subfolder under the task folder (e.g., `TaskNotes/Tasks/Archive`). Verify this destination is also inside your filesystem scope, or change it to a path that is. The archive path must be updated in both Obsidian settings and in `tasknotes-config.md`.

### tasknotes-config.md implications

The skill reads `tasknotes-config.md` to find the `tasks_folder` path for filesystem operations. This value must match the actual path configured in Obsidian - they are not linked automatically.

**After moving your TaskNotes folder:**

1. Open `tasknotes-config.md` in your vault
2. Update `tasks_folder` to reflect the new location (use a path relative to the config file's own directory)
3. If you have an archive subfolder configured, also update any archive-related paths in the config
4. Confirm that the path resolves correctly: `tasks_folder` joined with the config file's directory should produce the full path to your tasks folder

**Quick check:** ask your agent to run a schema diagnostic (Workflow 9) after moving folders - it will either find your tasks or tell you the folder is unreachable, confirming whether the paths are aligned.

---

## Complete HTTP API endpoint reference

Base URL: `http://localhost:{port}/api`

Authentication: optional. If `apiAuthToken` is set in plugin settings, send `Authorization: Bearer {token}`. If empty, all requests are accepted.

Full docs: https://tasknotes.dev/HTTP_API/

### System

| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/health` | Server status and vault info |
| GET | `/api/docs` | OpenAPI spec (JSON) |
| GET | `/api/docs/ui` | Interactive API explorer |
| POST | `/api/nlp/parse` | Parse natural language into task fields (no creation) |
| POST | `/api/nlp/create` | Parse and create task from natural language |

### Tasks

| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/tasks` | List tasks (pagination only - no filtering; use query endpoint) |
| POST | `/api/tasks` | Create a task |
| GET | `/api/tasks/:id` | Get one task's frontmatter by vault-relative path |
| PUT | `/api/tasks/:id` | Update task with partial payload |
| DELETE | `/api/tasks/:id` | Delete task file |
| POST | `/api/tasks/:id/toggle-status` | Cycle status through configured workflow |
| POST | `/api/tasks/:id/archive` | Toggle archived state |
| POST | `/api/tasks/:id/complete-instance` | Complete one instance of a recurring task |
| POST | `/api/tasks/query` | Advanced filtering with AND/OR logic, sorting, grouping |
| GET | `/api/filter-options` | Available statuses, priorities, tags, contexts, projects |
| GET | `/api/stats` | Task counts by status, priority, overdue, etc. |

**`:id` format:** URL-encoded vault-relative path. Example: `Agent%20Access%2FTaskNotes%2FTasks%2Fmy-task.md`

**Query body format:**
```json
{
  "type": "group",
  "id": "root",
  "conjunction": "and",
  "children": [
    { "type": "condition", "id": "c1", "property": "status", "operator": "is", "value": "open" },
    { "type": "condition", "id": "c2", "property": "projects", "operator": "contains", "value": "[[My Project]]" }
  ],
  "sortKey": "due",
  "sortDirection": "asc"
}
```

**Create task body fields:** `title` (required), `details`, `status`, `priority`, `due`, `scheduled`, `tags`, `contexts`, `projects`, `recurrence`, `timeEstimate`

### Time tracking

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/tasks/:id/time/start` | Start a time tracking session |
| POST | `/api/tasks/:id/time/start-with-description` | Start session with description |
| POST | `/api/tasks/:id/time/stop` | Stop active session |
| GET | `/api/tasks/:id/time` | Time summary and entries for a task |
| GET | `/api/time/active` | All currently running sessions |
| GET | `/api/time/summary` | Aggregate time summary |

### Pomodoro

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/pomodoro/start` | Start a pomodoro (body: `{ "taskId": "path", "duration": 25 }`) |
| POST | `/api/pomodoro/stop` | Stop and reset |
| POST | `/api/pomodoro/pause` | Pause |
| POST | `/api/pomodoro/resume` | Resume |
| GET | `/api/pomodoro/status` | Current state, time remaining, stats |
| GET | `/api/pomodoro/sessions` | Session history |
| GET | `/api/pomodoro/stats` | Aggregate stats |

### Calendars

| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/calendars` | Provider connectivity overview |
| GET | `/api/calendars/google` | Google Calendar connection details |
| GET | `/api/calendars/microsoft` | Microsoft/Outlook connection details |
| GET | `/api/calendars/subscriptions` | ICS subscriptions |
| GET | `/api/calendars/events` | Merged event list from all providers |

### Webhooks

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/webhooks` | Register a webhook |
| GET | `/api/webhooks` | List webhooks |
| DELETE | `/api/webhooks/:id` | Remove a webhook |
| GET | `/api/webhooks/deliveries` | Delivery history |

Webhook docs: https://tasknotes.dev/webhooks/

---

## Troubleshooting

### MCP tools not loading

1. Confirm Obsidian is running
2. Confirm both toggles are enabled (HTTP API + MCP Server) in Settings → TaskNotes → Integrations
3. Confirm port matches between plugin settings and agent config
4. Check `http://localhost:8080/api/health` in a browser - if it returns JSON, the server is up but the agent connection is failing; if it returns nothing, the server is not running
5. Restart both Obsidian and the agent

### Connection refused on port 8080

- Another application may be using port 8080
- Change the port in plugin settings (Settings → TaskNotes → Integrations → HTTP API → Port)
- Update the port in your agent config and restart

### Windows Firewall prompt

When Obsidian first starts the server, Windows may show a firewall prompt. Allow access on private networks. If you declined previously, go to Windows Firewall settings and add an exception for Obsidian.

### Toggles reset after update

Plugin updates can reset settings. After any TaskNotes update, re-enable both toggles and restart Obsidian.

### `mcp-remote` errors

- Confirm the health endpoint works in a browser first
- Confirm `npx` is available in the environment (requires Node.js)
- Try running `npx -y mcp-remote http://localhost:8080/mcp` in a terminal to see the raw error

### Task body content not available via MCP or API

Both MCP and HTTP API only return frontmatter fields (metadata cache limitation). To read body content - checklists, notes, acceptance criteria - read the task `.md` file directly using any filesystem tool. This is a known upstream gap (GitHub issue #1858 on callumalpass/tasknotes).

---

## Filesystem access (Obsidian closed)

When Obsidian is not running, all API access is unavailable. Task files are standard `.md` files with YAML frontmatter - any filesystem tool can read and write them directly.

Default task folder location: `{vault_root}/TaskNotes/Tasks/`. If this folder has been moved into a narrower filesystem scope, see the **Default folder locations and filesystem scope** section above for where to find it and what to check in `tasknotes-config.md`.

---

## mdbase-tasknotes CLI (`mtn`)

A standalone Node.js CLI that works without Obsidian running. Reads the vault directly. Supports full body content - titles, notes, checklists.

```
npm install -g mdbase-tasknotes
mtn config --set collectionPath=/path/to/your/vault
```

Key commands:
```
mtn list                              # Open tasks sorted by due date
mtn list --status in-progress         # Filter by status
mtn list --tag work                   # Filter by tag
mtn show "Task title"                 # Full task including body content
mtn search quarterly                  # Full-text search across titles and body
mtn create "Task title @context"      # Natural language creation
mtn update "Task title" --status in-progress
mtn update "Task title" --priority urgent --due 2026-06-01
mtn complete "Task title"
mtn timer start "Task title"
mtn timer stop
mtn projects list --stats
```

Full docs: https://tasknotes.dev/mdbase-tasknotes-cli/
