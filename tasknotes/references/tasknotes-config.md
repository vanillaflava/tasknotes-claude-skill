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

## Configuration Guide

**tasks_folder** - Path to the folder where task files are stored, relative to the directory containing this config file. `TaskNotes/Tasks` means a `TaskNotes/Tasks` folder sitting alongside this file. Both this config and your TaskNotes folder must be inside the scope your filesystem MCP allows the agent to reach.

**default_status / default_priority** - Applied by the agent to every task it creates directly. These are independent of the TaskNotes plugin's own default settings; the plugin defaults only apply to tasks created through the GUI (modal, NLP, instant conversion). Keep both in sync so tasks look consistent regardless of how they were created. Edit directly in Obsidian Properties.

**default_tags** - Tags applied to every agent-created task. The `task` tag is required; without it the task is invisible to all TaskNotes views and Bases queries. Additional tags are optional. Edit directly in Obsidian Properties.

Note: named `default_tags` rather than `tags` to avoid Obsidian treating this config file itself as a task.

**projects and contexts** - These sections appear orange in Obsidian Properties because they contain nested or mapped values that Obsidian cannot render as simple fields. To edit them, switch to Source mode: click the `</>` icon in the top-right corner of the note. Switch back to Reading or Live Preview when done.

**projects** - Pre-registered project notes for agent use. Maps a domain label to a vault note wikilink written into the task's `projects:` frontmatter field. The linked note displays a live Kanban of all tasks pointing to it via the Relationships Widget. The note must exist before tasks link to it. Multiple labels can map to the same note (e.g. health and tax both pointing to `[[Personal - Home]]`). Full reference: https://tasknotes.dev/core-concepts/

**contexts** - Optional semantic sub-domain strings. Maps a domain label to a context string written into the task's `contexts:` frontmatter field. Used to filter within a project's Bases views. Values accumulate in the vault and become available in the TaskNotes GUI `@` autosuggest as the task base grows. Full reference: https://tasknotes.dev/core-concepts/

**tags** - The `task` tag is the only required tag. Configure which tag identifies tasks in Settings -> General -> Task Tag. Full reference: https://tasknotes.dev/settings/task-properties/

---

## Task Frontmatter Reference

Canonical field order for every agent-created task. Field order matters for Obsidian Properties rendering.

```yaml
---
title: Task title here
status: open
priority: normal
scheduled: YYYY-MM-DD
completedDate:
projects:
  - "[[Project Note Title]]"
dateCreated: YYYY-MM-DD
dateModified: YYYY-MM-DD
tags:
  - task
contexts:
  - "@context-label"
---
```

All dates use `YYYY-MM-DD`. No timestamps, no timezone offsets. `completedDate` is present but empty on open tasks; set it when status moves to `done`.

**Writable fields - agent-safe:**

| Field | Type | Required | Notes |
|---|---|---|---|
| `title` | text | yes | Plain string; avoid colons (triggers YAML quoting issues in some parsers) |
| `status` | text | yes | `open`, `in-progress`, `done` |
| `priority` | text | yes | `low`, `normal`, `high` |
| `scheduled` | date | recommended | When to work on it. `YYYY-MM-DD` |
| `completedDate` | date | on completion | Set to today when status goes to `done`. `YYYY-MM-DD` |
| `projects` | list | recommended | Must be `[[wikilinks]]` - plain strings are silently broken |
| `dateCreated` | date | yes | Set once on creation, never modified. `YYYY-MM-DD` |
| `dateModified` | date | yes | Update on every edit. `YYYY-MM-DD` |
| `tags` | list | yes | Must include `task`; without it the file is invisible to all plugin views |
| `contexts` | list | optional | Sub-domain filter labels, e.g. `@meetings` |
| `due` | date | optional | Deadline. `YYYY-MM-DD` |
| `timeEstimate` | number | optional | Minutes, e.g. `30` |
| `recurrence` | text | optional | RFC 5545 RRULE with embedded DTSTART. See Workflow 10 in the skill |
| `recurrence_anchor` | text | optional | `scheduled` or `completion` - pair with `recurrence` |
| `blockedBy` | list | optional | List of dependency objects; see structure below |

**`blockedBy` structure:**

```yaml
blockedBy:
  - uid: "relative/path/to/blocking-task.md"
    reltype: "FINISHTOSTART"
```

`uid` is the vault-relative path to the blocking task file. `reltype` is always `FINISHTOSTART` for agent-created dependencies.

**Not writable by the agent** (plugin-managed lifecycle): `timeEntries`, `complete_instances`, `skipped_instances`, `icsEventId`, `reminders`. Do not write these fields.
