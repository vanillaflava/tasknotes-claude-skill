# tasknotes-claude-skill

A Claude agent skill for basic task management against an Obsidian vault with the [TaskNotes](https://tasknotes.dev/) plugin. This was my first learning project for understanding and implementing agent skills, specifically with Claude.

**Not affiliated with or endorsed by the TaskNotes project.** This skill was built by me, for personal use, and shared as-is. The TaskNotes plugin is developed and maintained independently by [callumalpass](https://github.com/callumalpass/tasknotes).

## What this is

A single installable skill that lets you talk to your tasks. Ask what's open, create a new one from conversation, mark something done, or let the agent manage its own Kanban board - all backed by plain Markdown files in your Obsidian vault. No API, no server, no HTTP calls.

Tasks are `.md` files with YAML frontmatter. The skill reads a config file to know where they live and how to route them to the right project. It coexists cleanly with the TaskNotes plugin - tasks the agent creates sit in the same folder as tasks you create via the GUI, using the same schema.

I use this for personal projects, work tracking, and the meta-work of running agent sessions themselves: the agents file their own tasks, track what needs doing across sessions, and hand off cleanly to whoever picks up the work next. If the filesystem MCP can reach your vault, it just works.

## What a task actually is

TaskNotes follows the "one note per task" principle. That means each task is a full Markdown document - YAML frontmatter for structured metadata (status, priority, due date, project) and a body that can hold anything: notes, decisions, sub-steps, links, accumulated context. Tasks grow and deepen over time.

This skill treats tasks accordingly. Creating a task is creating a document, not just a to-do entry. A task for a complex project might accumulate research, open questions, and interim decisions in its body across multiple agent sessions - the same way any note grows richer the more you return to it.

## What this does not cover

TaskNotes is a full-featured plugin. This skill covers a small slice of it.

Features this skill does not support:

- **HTTP API** - TaskNotes exposes a local REST API (`localhost:8080`) covering task queries, time tracking, Pomodoro, calendar integration, webhooks, and NLP parsing. This skill does not use any of it. If you need programmatic access to those features, the [HTTP API](https://tasknotes.dev/HTTP_API/) is the right path.
- **Recurring tasks** - the skill can create a task file with recurrence fields, but it cannot complete individual instances or manage the recurrence lifecycle the way the plugin does.
- **Time tracking and Pomodoro** - not accessible via file writes alone.
- **Calendar integration** - Google Calendar, Outlook, and ICS subscription features are plugin-only.
- **Reminders** - reminder scheduling is handled by the plugin at runtime, not in the file.
- **Inline task conversion** - the NLP-based checkbox-to-task workflow is GUI-only.
- **Webhooks** - plugin-only.
- **Status workflow configuration** - the plugin supports configurable status transition workflows; the skill writes status values directly without validating against your configured workflow.

If you need any of these, look at the [HTTP API](https://tasknotes.dev/HTTP_API/), the [Obsidian CLI](https://tasknotes.dev/obsidian-cli/), or the [mdbase-tasknotes CLI](https://tasknotes.dev/mdbase-tasknotes-cli/).

## Requirements

- **Obsidian** with the **TaskNotes plugin v4+** installed and enabled
- **Bases** core plugin enabled (required by TaskNotes v4)
- **Claude Desktop or Claude Code** (required - Claude.ai web and mobile cannot connect to local filesystem tools)
- **A filesystem MCP tool.** The minimum is [Anthropic's filesystem MCP](https://github.com/modelcontextprotocol/servers/tree/main/src/filesystem), available in Claude Desktop under **Customize -> Connectors**. Many alternatives exist - the MCP ecosystem moves quickly. When choosing one, look for directory scoping, per-tool permissions, and shell access restrictions; these matter for security and privacy (see below). [Desktop Commander](https://github.com/wonderwhy-er/DesktopCommanderMCP), also installable via Customize -> Connectors, has all three and is what this skill was built and tested with.

> **Note on Claude.ai web:** Completely untested with this skill, but Claude.ai's Google Drive connector could in principle support a vault stored in Google Drive. If you try this, please open an issue.

This skill has a hard dependency on [Obsidian](https://obsidian.md/) and the [TaskNotes](https://tasknotes.dev/) plugin (which must be installed separately via the community plugin settings inside Obsidian). Unlike my general [llm-wiki-skills](https://github.com/vanillaflava/llm-wiki-claude-skills) (see below), it is not portable to other Markdown environments.

**Other LLM providers.** Agent skills are a [published open spec](https://agentskills.io/specification) and this skill was built and tested against Claude Desktop. Other providers that implement the skill spec should work with minor adaptation - primarily the trigger description and slash command behaviour, which vary by implementation. The core file operations are provider-agnostic.

## Installation

1. Download `tasknotes.skill` from this repository
2. In Claude Desktop, go to **Customize -> Skills**
3. Upload the `.skill` file

The skill activates automatically for task-related conversational requests or via `/tasknotes`.

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

[llm-wiki-claude-skills](https://github.com/vanillaflava/llm-wiki-claude-skills) - my personal implementation of the [llm wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) pattern, and the reason this skill exists in the form it does.

The two together form a proper memory layer. The wiki carries accumulated knowledge - synthesised, crystallised, cross-linked. The tasks carry current intent: what is open, what was decided but not yet executed, what the next session needs to pick up. A fresh agent - whether that is a new chat cycle or a completely different domain agent picking up related work - reads the Kanban and knows exactly where things stand without anyone briefing them. The knowledge does not live in the chat history (that evaporates). It lives in the wiki and the tasks.

Domain home pages in the wiki make natural project anchors for the Relationships Widget. Tasks the agent files feed into the same compounding system. Neither requires the other, but the combination is where the interesting stuff happens.

## License

MIT
