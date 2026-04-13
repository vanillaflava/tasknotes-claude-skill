---
name: tasknotes
description: Create, read, update, and complete tasks using the Obsidian TaskNotes plugin. Use when the user mentions tasks, to-dos, action items, open items, backlog, or says /tasknotes. Also use for casual mentions like "add that to my list", "don't forget to", "mark X as done", "what's on my plate", "what's open for X", or "what's in progress". Requires filesystem read/write access to the vault.
compatibility: Requires Obsidian with the TaskNotes plugin (v4+) and Bases core plugin enabled.
metadata:
  version: "2.5"
---

# TaskNotes

Provides basic task CRUD — create, read, update, complete — for users with the TaskNotes plugin installed who want local agent operations without an HTTP API or MCP server. All operations work directly against task files on disk. This skill is a complement to the plugin, not a replacement: tasks created here coexist in the same folder with GUI-created tasks and use the same frontmatter schema, but the skill does not expose the full capabilities the plugin offers through its interface. Where the plugin's GUI and agent behaviour diverge — particularly around project assignment and filename conventions — the agent-creation path is the more reliable one for purposes of this skill.

---

## Configuration

**Finding your config:** Search for `tasknotes-config.md` by filename across accessible directories. Do not assume a path. If found, read it and extract:
- `tasks_folder` — absolute path to the TaskNotes Tasks folder
- `default_status` — default status for new tasks
- `default_priority` — default priority for new tasks
- `default_tags` — list of tags applied to every task; must include `task`
- `projects` — pre-registered project notes: maps domain labels to wikilinks
- `contexts` — optional: maps domain labels to context strings

**If `tasknotes-config.md` is not found — run init:**
1. Ask the user: *"I couldn't find tasknotes-config.md. Please provide the absolute path to your vault root."*
2. Construct the default tasks folder as `<vault_root>/TaskNotes/Tasks`
3. Write `tasknotes-config.md` to the vault root using the template below
4. Create `TaskNotes/Tasks/` if it does not exist
5. Confirm what was created and proceed

**Config template — write this verbatim when initialising, substituting the user's tasks folder path.**

Write the frontmatter block first:

```
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

Then write this body immediately after the closing `---`:

```markdown
## Configuration Guide

**tasks_folder** — Absolute path to the folder where task files are stored.
Windows example: `C:\Users\yourname\vault\TaskNotes\Tasks`
macOS/Linux example: `/Users/yourname/vault/TaskNotes/Tasks`

**default_status / default_priority** — Applied by the agent to every task it creates directly.
These are independent of the TaskNotes plugin's own default settings — the plugin defaults
only apply to tasks created through the GUI (modal, NLP, instant conversion). Keep both
in sync so tasks look consistent regardless of how they were created.
Edit directly in Obsidian Properties.

**default_tags** — Tags applied to every agent-created task. The `task` tag is required —
without it the task is invisible to all TaskNotes views and Bases queries. Additional tags
are optional. Edit directly in Obsidian Properties.

Note: named `default_tags` rather than `tags` to avoid Obsidian treating this config file
itself as a task.

**projects and contexts** — These sections appear orange in Obsidian Properties because they
contain nested or mapped values that Obsidian cannot render as simple fields. To edit them,
switch to Source mode: click the `</>` icon in the top-right corner of the note. Switch back
to Reading or Live Preview when done.

**projects** — Pre-registered project notes for agent use.
Maps a domain label to a vault note wikilink written into the task's `projects:` frontmatter
field. The linked note displays a live Kanban of all tasks pointing to it via the
Relationships Widget. The note must exist before tasks link to it. Multiple labels can map
to the same note (e.g. health and tax both pointing to `[[Personal - Home]]`).
Full reference: https://tasknotes.dev/core-concepts/

**contexts** — Optional semantic sub-domain strings.
Maps a domain label to a context string written into the task's `contexts:` frontmatter field.
Used to filter within a project's Bases views. Values accumulate in the vault and become
available in the TaskNotes GUI `@` autosuggest as the task base grows.
Full reference: https://tasknotes.dev/core-concepts/

**tags** — The `task` tag is the only required tag. Configure which tag identifies tasks
in Settings → General → Task Tag.
Full reference: https://tasknotes.dev/settings/task-properties/
```


---

## Capability Requirements

This skill requires **filesystem read and write access**. If running on a surface without filesystem tools (web, mobile), inform the user and stop.

---

## Data Model

Each task is a `.md` file in the configured `tasks_folder`. Filename convention: `YYYY-MM-DD-slug.md` where slug is the title lowercased with spaces as hyphens.

**Required fields — every agent-created task must include all of these:**

```yaml
---
title: Task title here
status: open
priority: normal
tags:
  - task
dateCreated: 2026-04-11T10:00:00.000+02:00
dateModified: 2026-04-11T10:00:00.000+02:00
---
```

Use `default_status`, `default_priority`, and `tags` from config for the values above.

**Optional fields — add only as needed:**

```yaml
projects:
  - "[[Project Note Title]]"
contexts:
  - "@context-label"
due: 2026-04-20
scheduled: 2026-04-15
completedDate: 2026-04-11
timeEstimate: 30
blockedBy: []
```

`projects:` must be a wikilink to a real vault note. A plain string (e.g. `- work`) creates no backlink, triggers no Relationships Widget, and is silently broken.

**Timestamp format:** ISO 8601 with milliseconds and timezone offset — `2026-04-11T10:00:00.000+02:00`. Match the format of existing task files in the vault.

**Task body:** Write enough context that any agent or human can pick up the task without the originating chat. Include: what needs doing and why, acceptance criteria as inline checklist items, constraints, links to related notes.

---

## Organisation: Projects, Contexts, and Tags

TaskNotes provides three organisational properties. They are all optional beyond the mandatory `task` tag, but using them unlocks filtering, views, and the Relationships Widget.

**`projects:` — project note link**
A wikilink to a vault note that owns this task. When a task's `projects:` field points to a note, the Relationships Widget on that note automatically shows a live Kanban of all tasks pointing to it — no setup required beyond the link existing. One task can belong to multiple projects. Any vault note can serve as a project note: a dedicated project page, a domain home note, or a simple "Work tasks" note you create for the purpose.

The agent uses the `projects:` map in `tasknotes-config.md` as a lookup table to know which note to link without asking every time. Register a note there before creating tasks for that domain.

**`contexts:` — context label**
A free-form string describing where or how the task gets done (GTD-style: `@home`, `@computer`, `@meetings`). Does not drive the Relationships Widget. Used to filter within a project's Bases views. Multiple contexts per task are allowed. As context strings accumulate in the vault they become available in the TaskNotes GUI `@` autosuggest.

**`tags:` — Obsidian tags**
The `task` tag (set in defaults) is the only required tag — it is how TaskNotes identifies task files. Additional tags are optional. Configure which tag identifies tasks in Settings → General → Task Tag.

Full reference: https://tasknotes.dev/core-concepts/

---

## Relationships Widget

The Relationships Widget is a TaskNotes v4 feature that renders relationship data directly on vault notes.

The relevant tab for project notes is **Subtasks (Kanban)**: shows all tasks whose `projects:` field contains a wikilink to the current note, rendered as a Kanban grouped by status. This works on any vault note — not just task files.

The widget is entirely automatic and reactive. When you open a note, TaskNotes queries the vault for tasks pointing to it and renders the widget. If no tasks reference the note, the widget does not appear.

**Configuration is global, not per-note:**
- On/off: Settings → TaskNotes → Appearance → UI Elements → Relationships Widget
- Position: Settings → TaskNotes → Appearance → UI Elements → Relationships Position (top or bottom)

Other tabs (Projects, Blocked By, Blocking) appear when dependency relationships exist between task files.

Full reference: https://tasknotes.dev/features/inline-tasks/


---

## Project Routing Discipline

On every task creation:
1. Read `tasknotes-config.md` and locate the `projects` map
2. Identify the right project note for this task's domain
3. Verify the project note exists in the vault — do not create a task linking to a non-existent note
4. If the note does not exist: stop, inform the user, offer to create it first
5. If no entry matches the domain: present the known project list and ask which applies, or whether a new entry should be added to config first
6. Never write a task with a plain string in `projects:` — always a wikilink

**Adding a new project:**
1. Confirm the note name and location with the user
2. Create the note if it does not exist (a minimal stub is sufficient)
3. Add the entry to `tasknotes-config.md` under `projects:`
4. Then create the task

**If the user has not configured projects:** tasks can be created without a `projects:` field. Inform the user that without it the task will not appear in any project view or trigger the Relationships Widget, then proceed if they confirm.

---

## Subtask Model

**Tier 1 — Inline checklists:** Steps or acceptance criteria within a single task. Bases views show checklist progress automatically.

```markdown
## Acceptance criteria
- [ ] Step one
- [ ] Step two
```

**Tier 2 — Child task files:** When a subtask warrants its own status and notes, create it as a normal task file and set its `projects:` to a wikilink of the parent task title. The Relationships Widget surfaces child tasks in the parent's Subtasks tab automatically.

Rule of thumb: inline checklists for steps within one task; child task files for work substantial enough to need its own status tracking.

---

## Workflows

### 1. Create a task

1. Read `tasknotes-config.md`
2. Infer domain from conversation; look up project note and context
3. Verify project note exists (see Project Routing Discipline)
4. Generate filename: `YYYY-MM-DD-<slug>.md`. Append `-2`, `-3` if file exists.
5. Get current ISO timestamp for `dateCreated` and `dateModified`
6. Write file with required frontmatter + relevant optional fields + body
7. Confirm filename and key fields to the user

**GUI creation note:** Agent creation is the reliable path for correct project assignment and filename conventions. If the user creates tasks via the plugin GUI instead:
- Use **Show Detailed Options** in the modal to access the project picker — the abbreviated modal only inherits the plugin's configured default project
- At least one default project must be configured in plugin settings (Settings → TaskNotes) as a safety net; without it, tasks created without "Show Detailed Options" will be orphaned with no `projects:` field
- The plugin's task folder setting must match `tasks_folder` in `tasknotes-config.md` — verify both after any vault migration or reinstall; a mismatch silently routes GUI tasks to the wrong directory

### 2. Find / read tasks

**By project:** Search `tasks_folder` for files containing the project wikilink (e.g. `[[Work - Home]]`).
**By status:** Search for `status: open` or `status: in-progress`.
**Combined:** Search for project wikilink, filter to target status.
**By keyword:** Search task file contents.

Always read each candidate file in full before acting on it.

### 3. Update a task

Always read the file first. Use surgical edits. Always update `dateModified` alongside any other change.

### 4. Complete a task

1. Read the task file
2. Update `status: done`
3. Add `completedDate: YYYY-MM-DD`
4. Update `dateModified`

Never delete task files. Done tasks remain — TaskNotes views filter by status.

### 5. List tasks for a domain

1. Read `tasknotes-config.md` for the project wikilink
2. Search `tasks_folder` for files containing that wikilink. Let the search run to full completion — do not stop early. Paginate with `get_more_search_results` until the search session reports no more results before reading or filtering.
3. Read frontmatter of each match: title, status, priority, due
4. Present: `in-progress` first, then `open`; skip `done`

**Domain signal note:** The project wikilink is the only reliable cross-task domain signal. `contexts:` is a secondary filter within results — use it to narrow a result set, never as a substitute for the wikilink search. Some tasks may have no `contexts:` field at all; never rely on context alone.

### 6. Adopt a GUI-created task

Use when the user asks to fix or tidy up a task created via the plugin GUI that has incorrect or missing fields.

1. Read the task file
2. Check `projects:` — if absent, add the correct wikilink; if path-prefixed (e.g. `[[Vault/Folder/Note]]`), replace with the bare note title (e.g. `[[Note Title]]`)
3. Check `tags:` — add `task` if missing
4. Optionally rename the file to `YYYY-MM-DD-slug.md` convention for consistency — ask the user before renaming
5. Update `dateModified`
6. Confirm changes to the user

### 7. Extend a domain with custom properties

Document custom fields in `tasknotes-config.md` under a `domain_extensions:` key. Include them when creating tasks in that domain. Do not add domain-specific fields to tasks from other domains.

---

## Creating Domain Views

TaskNotes views are `.base` files in `TaskNotes/Views/`. Agents can read and write them but cannot execute them directly. Ask the user before creating new view files. See the default `kanban-default.base` for formula patterns (urgency scoring, overdue detection) reusable in custom views.

Full reference: https://tasknotes.dev/views/

---

## Safety Rules

1. Never delete task files — use `status: done` + `completedDate` instead
2. Never modify `dateCreated` — set once on creation, never changed
3. Always update `dateModified` on every edit
4. Always include `tags: - task` in task frontmatter — without it the task is invisible to all views; read `default_tags` from config to confirm the required value
5. `projects:` must be a wikilink — plain strings are silently broken
6. The project note must exist before a task links to it — verify or create it first
7. Read before editing — never overwrite from stale context
8. Unknown projects require a config update first

---

## Cloud-Synced Vaults

Vaults stored in cloud sync services may have files not locally downloaded, appearing as zero-byte placeholders. If a file read returns empty unexpectedly, flag it as a possible sync issue and ask the user to confirm before retrying.

Erratic plugin behaviour (settings not persisting, widget position jumping) is environment-specific — inform the user and do not attempt to fix it via file edits.
