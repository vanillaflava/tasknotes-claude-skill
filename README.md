# tasknotes-skill

[![Release](https://img.shields.io/github/v/release/vanillaflava/tasknotes-skill?style=flat-square)](https://github.com/vanillaflava/tasknotes-skill/releases/latest) [![License](https://img.shields.io/badge/license-MIT-green?style=flat-square)](https://github.com/vanillaflava/tasknotes-skill/blob/master/LICENSE) [![Agent Skills](https://img.shields.io/badge/Agent_Skills-compatible-brightgreen?style=flat-square)](https://agentskills.io/specification) [![Claude](https://img.shields.io/badge/Claude-D97757?style=flat-square&logo=claude&logoColor=white)](https://claude.ai) [![Obsidian](https://img.shields.io/badge/Obsidian-%23483699?style=flat-square&logo=obsidian&logoColor=white)](https://obsidian.md) [![TaskNotes](https://img.shields.io/badge/TaskNotes-plugin-blue?style=flat-square)](https://tasknotes.dev) [![MCP](https://img.shields.io/badge/MCP-compatible-blue?style=flat-square)](https://modelcontextprotocol.io)

Talk to your tasks. Ask what's open, file something from conversation, mark it done. Tasks are plain Markdown files in your Obsidian vault - the skill works with filesystem access alone, and gets richer query and tracking capabilities when the TaskNotes MCP server or HTTP API are also available.

**Not affiliated with or endorsed by the TaskNotes project.** This skill was built by me, for personal use, and shared as-is. The TaskNotes plugin is developed and maintained independently by [callumalpass](https://github.com/callumalpass/tasknotes).

## What this is

A single installable skill for task management: ask what's open, create a task from conversation, update or close it, check what's blocking you, track time, run a schema diagnostic. Tasks are `.md` files with YAML frontmatter. The skill reads a config file to know where they live and how to route them to the right project.

The skill adapts to the environment. With filesystem access alone it covers task CRUD, file-based querying, body and checklist reads, and schema diagnostics - and works even when Obsidian is closed. When the TaskNotes MCP server or HTTP API are also available (Obsidian running, integration toggles enabled), it uses those as the primary path for frontmatter operations and gains access to time tracking, Pomodoro, calendar events, and per-instance recurring task completion. The surface lookup table in the skill body documents exactly what is available on each path.

It coexists cleanly with the TaskNotes plugin - tasks the agent creates sit in the same folder as tasks you create via the GUI, using the same schema.

I use this for personal projects, work tracking, and the meta-work of running agent sessions themselves. The satisfying part: the agents file their own tasks, track what needs doing across sessions, and hand off cleanly to whoever picks up the work next.

**Tasks orient a fresh agent the same way a wiki Domain Home does.** When an agent starts a new chat, it reads the open tasks for a domain and immediately knows what's in progress, what was decided but not yet executed, what's blocked. No briefing needed. This is not incidental - it is the architecture. The agent loses its working memory at the end of every session by design, and tasks are part of what survives.

**The Kanban board and the agent are looking at the same data differently.** In Obsidian, any task linked to a domain home via `projects: [[Domain Home]]` is rendered as a live Kanban card through the Relationships Widget - automatically, no configuration required beyond the link. This is a visual aid for you as the human reader. The agent never sees the Kanban itself; it discovers tasks by scanning the task files directly. Both mechanisms are reading the same underlying Markdown files. The Kanban is the surface; the files are the truth.

## What a task actually is

TaskNotes follows the "one note per task" principle. That means each task is a full Markdown document - YAML frontmatter for structured metadata (status, priority, due date, project) and a body that can hold anything: notes, decisions, sub-steps, links, accumulated context. Tasks grow and deepen over time.

This skill treats tasks accordingly. Creating a task is creating a document, not just a to-do entry. A task for a complex project might accumulate research, open questions, and interim decisions in its body across multiple agent sessions - the same way any note grows richer the more you return to it.

## What this covers depends on your setup

Capabilities vary by environment. The skill probes what is available at the start of each session and routes accordingly.

**Filesystem access alone** (minimum - works even when Obsidian is closed):
- Task CRUD, file-based querying, schema diagnostics
- Full body and checklist reads (filesystem is the only path that can return body content)
- Recurring task creation (sets the RRULE and first scheduled date)

**HTTP API or MCP available** (Obsidian running, integration toggles enabled - adds):
- Reliable filtered queries without scanning all files
- Time tracking and Pomodoro sessions
- Calendar event reads
- Per-instance recurring task completion

**Body content gap:** Both MCP and HTTP API return frontmatter only - this is an upstream limitation of the plugin's metadata cache (GitHub issue [#1858](https://github.com/callumalpass/tasknotes/issues/1858)). Body content always requires a direct filesystem read.

**Not covered regardless of setup:**

- **Reminders** - scheduling and firing are runtime-only plugin behavior; not accessible via any external path
- **Inline task conversion** - the NLP checkbox-to-task workflow is GUI-only
- **Status workflow validation** - the skill writes status values directly without checking your configured workflow transitions
- **Webhooks** - no dedicated skill workflow; use the [HTTP API](https://tasknotes.dev/HTTP_API/) directly if you need this
- **Plugin configuration** - settings, `.json` config files, and any feature that requires writing to Obsidian's internal state are out of scope

For time tracking, Pomodoro, calendar, and recurring task instance management without MCP or HTTP API access, the [mdbase-tasknotes CLI](https://tasknotes.dev/mdbase-tasknotes-cli/) is a standalone alternative that reads vault files directly.

## Requirements

**Required:**

- **Obsidian** with the **TaskNotes plugin v4+** installed and enabled
- **Bases** core plugin enabled (required by TaskNotes v4)
- **An agent with filesystem access.** The skill reads and writes files on your local disk. How you get access depends on your platform:
  - **Claude Desktop** has no native filesystem access. Add a filesystem MCP - [Desktop Commander](https://github.com/wonderwhy-er/DesktopCommanderMCP) or the [official Anthropic filesystem MCP](https://github.com/modelcontextprotocol/servers/tree/main/src/filesystem), both installable via **Customize → Connectors**. When choosing, look for directory scoping, per-tool permissions, and shell access restrictions; these matter for security and privacy (see below). Desktop Commander has all three and is what this skill was built and tested with.
  - **Claude Code, Gemini CLI, OpenAI Codex CLI, and GitHub Copilot** have filesystem access built in. No MCP required.
  - **Claude.ai web and mobile** cannot reach files on your disk. Completely untested with this skill; Claude.ai's Google Drive connector could in principle support a vault stored in Google Drive - if you try this, please open an issue.

This skill has a hard dependency on [Obsidian](https://obsidian.md/) and the [TaskNotes](https://tasknotes.dev/) plugin. Unlike my general [llm-wiki-skills](https://github.com/vanillaflava/llm-wiki-skills), it is not portable to other Markdown environments.

**Optional (unlocks MCP and HTTP API paths):**

- **TaskNotes HTTP API:** enable in Settings → TaskNotes → Integrations → HTTP API. Adds reliable filtered queries, time tracking, Pomodoro, calendar events, and recurring task instance completion via REST calls to `localhost:8080`.
- **TaskNotes MCP server:** enable in Settings → TaskNotes → Integrations → MCP Server (requires HTTP API toggle also on). Exposes 24 tools directly to the agent - the preferred path when Obsidian is running. See `references/tasknotes-help.md` in the skill for setup config and multi-platform instructions.

Both require Obsidian to be running. The skill falls back to filesystem access automatically if either is unavailable.

## Installation

### Claude Desktop

Download [**tasknotes.skill**](https://github.com/vanillaflava/tasknotes-skill/releases/latest/download/tasknotes.skill) and upload it under **Customize → Skills**.

The skill activates automatically for task-related conversational requests or via `/tasknotes`.

### Other agents (Claude Code, Gemini CLI, Codex CLI, GitHub Copilot, and more)

Use the `skills` CLI:

```bash
npx skills add vanillaflava/tasknotes-skill
```

Or install manually - copy the `tasknotes/` folder into your agent's skills directory:

```
Claude Code:     ~/.claude/skills/tasknotes/
Codex CLI:       ~/.codex/skills/tasknotes/
Gemini CLI:      ~/.gemini/skills/tasknotes/
GitHub Copilot:  configure via chat.agentSkillsLocations in VS Code
```

Copy the **entire `tasknotes/` folder**, not just `SKILL.md` - the skill bundles reference files in `references/` that it reads at runtime.

To update, re-run the install command - it always fetches the latest version from master. `npx skills update` exists but has known issues with remote change detection.

## Getting started

**Where things need to live**

The skill only works on files the agent can reach. Your filesystem MCP's allowed scope is the boundary. Everything the skill needs - the config file, the TaskNotes folder, task files themselves - must live inside that scope.

TaskNotes defaults to placing its folder at `YourVault/TaskNotes/` with `Tasks/` and `Views/` subfolders inside it. If your MCP scope is your whole vault, that's fine as-is. If your MCP scope is a subfolder of your vault (recommended - exposing less to the LLM), you need to move the TaskNotes folder into that subfolder. The plugin will not follow your scoping decisions; no matter what the config says, if the tasks folder sits outside MCP scope, the skill cannot reach it.

Set it up so the layout inside your MCP scope looks like this:

```
your-mcp-scope/          ← the folder your MCP is scoped to
├── tasknotes-config.md  ← drop this here
└── TaskNotes/
    ├── Tasks/           ← your task files live here
    └── Views/           ← .base views
```

Two plugin settings to get right so the plugin and the skill agree on where tasks live:

- **Settings -> TaskNotes -> General -> Task folder:** point it at the absolute path of the `Tasks/` folder above
- **Settings -> TaskNotes -> General -> Auto-create task folder:** disable it. Otherwise the plugin may recreate `TaskNotes/` at your vault root (outside your scope), which defeats the scoping work you did

**1. Create tasknotes-config.md (optional)**

If `tasknotes-config.md` is not found, the skill will ask you where to set it up, create the config file, and set up the Tasks folder. You can skip this step and let it happen in conversation.

If you prefer to set it up beforehand, create `tasknotes-config.md` in your MCP scope root. A minimal config:

```yaml
---
tasks_folder: TaskNotes/Tasks

default_status: open
default_priority: normal
default_tags:
  - task

projects:
  work: "[[Work - Home]]"
  personal: "[[Personal - Home]]"

contexts:
  meetings: "@meetings"
  admin: "@admin"
---
```

`tasks_folder` is relative to the directory containing this config file. `TaskNotes/Tasks` means "a `TaskNotes/Tasks` folder sitting next to this config file". Keep them together.

`projects` values must be wikilinks to existing notes - for example `"[[Work - Home]]"`. A plain string is silently accepted but will break the Relationships Widget, which is what surfaces your tasks as a live Kanban on the linked note. Always use the `[[Note Title]]` format.

**2. Ask the agent to create a task**

> Add a task to review the Q2 report, work project, due Friday

The agent reads the config, finds the right project note, and writes the task file. Done.

## Limitations

Agent-created tasks use a `YYYY-MM-DD-slug.md` filename convention. Tasks created via the plugin GUI use the task title as filename and inherit the plugin's configured defaults. Both coexist in the same folder without conflict. If you care about filename consistency, create tasks via the agent or use "Show Detailed Options" in the plugin modal.

**Task accumulation.** All tasks - open, done, and archived - stay in the Tasks folder by default. Archiving a task in TaskNotes sets an `archived` field; it does not move the file. The plugin's views filter by status and archive state so the GUI stays clean, but the folder grows without bound and the agent scans every file in it. At a few dozen tasks this is imperceptible. At several hundred it becomes slow, and at a few thousand it will fail or time out.

TaskNotes has a built-in remedy: **Settings -> TaskNotes -> General -> Move Archived Tasks to Folder**. Enable it and point it at a subfolder like `Tasks/Archive/`. The plugin will then move files there automatically when you archive them, keeping your main task list tidy. TaskNotes scans recursively so those files remain visible to the plugin - but they stay out of your way in the GUI.

Before enabling this, verify that your plugin folder settings and `tasknotes-config.md` are pointing at the same locations. The plugin manages `TaskNotes/`, `TaskNotes/Tasks/`, and the archive subfolder independently from the skill's config; it will not infer from `tasks_folder` in your config file. If they are out of sync, archived tasks will move somewhere neither the plugin nor the agent expects. Check: Settings -> TaskNotes -> General -> Task folder resolves to the same folder as `tasks_folder` in your config, and the archive subfolder you configure sits inside it.

This does not fully solve the agent scan problem, since `Tasks/Archive/` is still scanned by the agent. When the folder gets heavy, the skill will flag it and walk you through archiving done tasks via the plugin GUI - presenting a list sorted by age and linking you to the relevant plugin settings.

## Privacy

**Your vault is not private from your LLM provider.** Any task file the agent reads - title, body, project links, anything you have written in it - is sent to Anthropic (or whichever provider you use) as part of the conversation. This is true even if your vault is stored entirely on local disk. The only data boundary that matters is what the agent is allowed to access.

Before you start, it helps to think in two scopes:

```
Your machine
├── Everything else on your filesystem
└── Your knowledge space (Obsidian vault, markdown reader, notes folder, etc.)
    ├── Private/          ← medical, financial, personal
    ├── Sensitive/        ← work confidential, legal, credentials
    ├── Archive/          ← legacy notes you don't want an LLM near
    └── Agent Access/     ← the ONLY folder the agent needs
        ├── tasknotes-config.md
        └── TaskNotes/
            ├── Tasks/
            └── Views/
```

Give the agent-accessible folder an obvious name - `Agent Access` makes the boundary legible when you set things up and when you revisit later. Everything outside it should be unreachable by the agent. Everything inside it should be something you are comfortable sending to your LLM provider.

Scope your filesystem MCP to that folder. If you use Desktop Commander, its `allowedDirectories` setting is the right lever. Anthropic's own filesystem MCP scopes to whatever directories you configure at setup. Agents with native filesystem access, such as Claude Code, are not constrained by MCP-level controls and must be scoped separately through their own configuration.

The TaskNotes plugin does not respect your MCP scoping decisions. If the plugin's Task folder setting points outside your MCP scope, or auto-create recreates `TaskNotes/` at vault root, the skill cannot reach your tasks. After setting up, verify that both the plugin's Task folder and your `tasknotes-config.md` resolve to the same location inside your MCP scope, and disable Settings -> TaskNotes -> General -> Auto-create task folder so the plugin does not silently undo your setup.

The filesystem MCP you use may also send data of its own. Desktop Commander, for example, sends anonymised telemetry by default unless you disable it in its config. Check the documentation and privacy settings of any MCP tool before connecting it to sensitive material.

What happens to data once it reaches your provider depends on their privacy policy and your account settings. Claude's data handling is described at [anthropic.com/privacy](https://www.anthropic.com/privacy). If you use a different provider, check their policy before proceeding.

**The skill stays in its lane.** Agent skills are instructions, not firewalls; what actually gets done depends on what the agent decides in the moment. This skill is written to confine itself to reading and writing task files in your `tasks_folder` (plus `.base` view files and its own config). For anything that cannot be done with those file operations - changing plugin defaults, adjusting the task tag, configuring archive behaviour, toggling widget settings - the skill is instructed to send you to Obsidian Settings rather than terminal out and edit `data.json` (which would silently fail anyway). This is the correct lane for a task CRUD skill to operate in: TaskNotes is a feature-rich plugin, the skill is a narrow complement, and plugin configuration belongs in the plugin's own GUI.

## Works well with

[llm-wiki-skills](https://github.com/vanillaflava/llm-wiki-skills) - my personal implementation of the [Karpathy LLM wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f), and the reason this skill exists in the form it does.

The two together form a complete memory layer. The wiki carries accumulated knowledge - synthesised, crystallised, interlinked pages that compound over time. The tasks carry current intent: what is open, what was decided but not yet executed, what the next session needs to pick up. Neither requires the other, but the combination is where the interesting stuff happens.

**The session cycling problem and how they solve it together.** Agents lose their working memory. The context window fills, things slow down, you start a fresh chat. Without structure, you spend the first ten minutes re-briefing the agent on where things stand. With the wiki and tasks in place, a fresh agent reads the Domain Home (current knowledge state) and open tasks (current work state) and picks up almost immediately.

This works in three layers:

- **Personal Preferences** (Claude Settings, system prompt elsewhere) carry your permanent working style and preferences into every conversation automatically.
- **Project Instructions** (Claude's term; most providers have an equivalent) inject domain context before the chat starts. Point them at the Domain Home or paste a summary.
- **Domain Home + open tasks** are the final layer. The Domain Home carries what is known; the open tasks carry what needs doing. A fresh agent reads both and is oriented without a briefing.

**The Kanban board sits at the intersection.** Domain home pages in the wiki make natural project anchors. Any task filed with `projects: [[Domain Home]]` shows up as a live Kanban card in Obsidian via the Relationships Widget - you see the board without setting anything up beyond the link. The agent reads the same tasks from the files directly; the Kanban is your view of the same data, not a separate thing to maintain.

The result: the wiki holds the knowledge, the tasks hold the intent, and a fresh agent can arrive at either without a handoff note from the previous session. That is the system working as designed.

## License

MIT
