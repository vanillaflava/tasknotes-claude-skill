# tasknotes-claude-skill

A Claude agent skill for basic task management against an Obsidian vault with the [TaskNotes](https://tasknotes.dev/) plugin. This was my first learning project for understanding and implementing agent skills, specifically with Claude.

**Not affiliated with or endorsed by the TaskNotes project.** This skill was built by me, for personal use, and shared as-is. The TaskNotes plugin is developed and maintained independently by [callumalpass](https://github.com/callumalpass/tasknotes).

## What this is

A single installable skill that lets a Claude agent create, read, update, and complete tasks by reading and writing Markdown files directly on disk: no API, no server, no HTTP calls. Tasks are `.md` files with YAML frontmatter; the skill reads a config file to know where they live and how to route them to the right project.

This skill works at the file level only. It does not use the TaskNotes HTTP API, NLP engine, or any GUI workflow. What it does is narrow and deliberate: it reads and writes task files in the format TaskNotes expects, so tasks created by the agent coexist in the same folder with tasks created via the plugin GUI.

The reason to use this approach rather than the API: no setup overhead, no requirement for Obsidian to be running, no port configuration, and no dependency on desktop-only features. If the filesystem MCP can reach your vault folder, the skill works.

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

**1. Create tasknotes-config.md (optional)**

If `tasknotes-config.md` is not found, the skill will prompt you for your vault root path, create the config file there, and set up the Tasks folder automatically. You can skip this step and let it happen in conversation.

If you prefer to set it up beforehand, create `tasknotes-config.md` anywhere in your vault. A minimal config:

```yaml
---
tasks_folder: /absolute/path/to/TaskNotes/Tasks

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

Two things to get right in this config:

**`tasks_folder`** must be an absolute local path that your filesystem MCP can reach. If your vault lives on a cloud sync service (iCloud, OneDrive, Dropbox), use the local sync folder path, not a cloud URL. This path must also match the Task folder setting in the TaskNotes plugin (Settings -> TaskNotes -> Task folder) - if they differ, agent-created and GUI-created tasks will land in different places.

TaskNotes defaults to `YourVault/TaskNotes/` for its folder, with `Tasks/` and `Views/` subfolders inside it. The plugin can auto-create these on first run (Settings -> TaskNotes -> General -> Auto-create task folder). If you are happy with that default location, `tasks_folder` is just that path made absolute.

**Scope your agent's access carefully.** The skill only needs access to the folder where your task files live - it does not need access to your entire vault, let alone your entire filesystem. Before setting up, decide what scope you are comfortable exposing to an LLM and configure your filesystem MCP accordingly. A scoped path (e.g. pointing at `TaskNotes/Tasks` only) is preferable to vault root; vault root is preferable to full filesystem access. If you scope below vault root, make sure the plugin's Task folder setting points to the same location - the plugin's auto-create toggle assumes vault-level access and may not create folders outside that scope. Agents with native filesystem access (such as Claude Code) do not use MCP directory controls and must be scoped separately through their own configuration. The broader the access, the more of your data passes through the LLM - see Privacy below.

**`projects` values must be wikilinks** to existing notes - for example `"[[Work - Home]]"`. A plain string tag is silently accepted by the config but will break the Relationships Widget, which is what surfaces your tasks as a live Kanban on the linked note. Always use the `[[Note Title]]` format.

**2. Ask the agent to create a task**

> Add a task to review the Q2 report, work project, due Friday

The agent reads the config, finds the right project note, and writes the task file. Done.

## Limitations

Agent-created tasks use a `YYYY-MM-DD-slug.md` filename convention. Tasks created via the plugin GUI use the task title as filename and inherit the plugin's configured defaults. Both coexist in the same folder without conflict. If you care about filename consistency, create tasks via the agent or use "Show Detailed Options" in the plugin modal.

**Task accumulation.** All tasks - open, done, and archived - stay in the Tasks folder by default. Archiving a task in TaskNotes sets an `archived` field; it does not move the file. The plugin's views filter by status and archive state so the GUI stays clean, but the folder grows without bound and the agent scans every file in it. At a few dozen tasks this is imperceptible. At several hundred it becomes slow, and at a few thousand it will fail or time out.

TaskNotes has a built-in remedy: **Settings -> TaskNotes -> General -> Move Archived Tasks to Folder**. Enable it and point it at a subfolder like `Tasks/Archive/`. The plugin will then move files there automatically when you archive them, keeping your main task list tidy. TaskNotes scans recursively so those files remain visible to the plugin - but they stay out of your way in the GUI.

Before enabling this, verify that your plugin folder settings and `tasknotes-config.md` are pointing at the same locations. The plugin manages `TaskNotes/`, `TaskNotes/Tasks/`, and the archive subfolder independently from the skill's config - it will not infer from `tasks_folder` in your config file. If they are out of sync, archived tasks will move somewhere neither the plugin nor the agent expects. Check: Settings -> TaskNotes -> General -> Task folder matches the absolute path in `tasks_folder`, and that the archive subfolder you configure sits inside it.

This does not fully solve the agent scan problem, since `Tasks/Archive/` is still scanned by the agent. When the folder gets heavy, the skill will flag it and walk you through archiving done tasks via the plugin GUI - presenting a list sorted by age and linking you to the relevant plugin settings.

## Privacy

**Your vault is not private from your LLM provider.** Any task file the agent reads - title, body, project links, anything you have written in it - is sent to Anthropic (or whichever provider you use) as part of the conversation. This is true even if your vault is stored entirely on local disk. The only data boundary that matters is what the agent is allowed to access.

Think carefully about how much of your vault you want to expose before you start. The skill only needs to reach your Tasks folder - it does not need vault root, and vault root is not recommended. Use your filesystem MCP's access controls to limit scope. If you use Desktop Commander, its `allowedDirectories` setting is the right lever. Anthropic's own filesystem connector scopes to whatever directories you configure at setup. Agents with native filesystem access, such as Claude Code, are not constrained by MCP-level controls and must be scoped separately through their own configuration - this requires explicit attention if you use them.

The filesystem MCP itself may also send data. Desktop Commander, for example, sends anonymised telemetry by default unless you disable it in its config. Check the documentation and privacy settings of any MCP tool you install before connecting it to sensitive material.

What happens to data once it reaches your provider depends on their privacy policy and your account settings. Claude's data handling is described at [anthropic.com/privacy](https://www.anthropic.com/privacy). If you use a different provider, check their policy before proceeding.

## Works well with

[llm-wiki-claude-skills](https://github.com/vanillaflava/llm-wiki-claude-skills) - the llm wiki skills (my personal implementation of the [llm wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) pattern) and this skill were designed alongside each other. Domain home pages in the wiki make natural project anchors for the Relationships Widget, and tasks captured via the agent feed into the same compounding knowledge system. Neither requires the other.

## License

MIT
