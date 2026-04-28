# Claude Instructions For The Automation Plugin

This folder packages the working Codex automation plugin and the lessons learned from debugging Codex desktop automations on Windows.

## Purpose

Help an agent turn plain-English automation requests into reliable Codex desktop automations.

Examples:

- Create an automation every day at 9 AM.
- Delete an existing automation.
- Explain why an automation is not visible in the Automations tab.
- Recreate automations with a specific model and reasoning effort.

## Important Concept

The plugin is an instruction and skill package. It does not directly provide the scheduler.

When installed, users call it as:

```text
@automation
```

The agent should then follow the skill at:

```text
automation/skills/create-automation/SKILL.md
```

## Install Instructions For Users

1. Open Codex desktop.
2. Open the plugin store.
3. Choose the local marketplace/folder entry for this package. With the included marketplace file, it should appear as `Automation Local`.
4. Select the local `automation` plugin from this package.
5. If Codex asks for a folder, choose the package root, not only the nested `automation` folder.
6. Install it.
7. Use `@automation` in chat.

## Best Path

Always prefer the native Codex automation tool if available.

Search for:

```text
automation_update
```

If exposed, use it. It handles storage, scheduler state, and UI invalidation correctly.

## Fallback Path

If no native tool is exposed, use the local fallback documented in `SKILL.md`.

Current local storage:

```text
%USERPROFILE%\.codex\automations\<automation-id>\automation.toml
%USERPROFILE%\.codex\sqlite\codex-dev.db
```

The TOML file controls what appears in the Automations tab. The database row mirrors scheduler state.

## The Key Bug We Found

A manually created automation did not appear because `automation.toml` started with a UTF-8 BOM.

The file existed. The DB row existed. The Automations tab still ignored it because the parser silently rejected the TOML.

Fix:

- Rewrite TOML as UTF-8 without BOM.
- Verify first bytes.

Expected:

```text
76 65 72 73 69 6f 6e 20
```

Bad:

```text
ef bb bf
```

## UI Refresh Notes

The Automations tab refreshes by query invalidation and refetching, not pure filesystem watching.

Native create/update/delete causes a proper refresh. Manual fallback edits may require navigation, refetch, or restart, although a valid no-BOM file can appear without restart after the tab refetches.

## Maintenance Notes

Keep these files aligned:

- `README.md`
- `AGENTS.md`
- `CLAUDE.md`
- `automation/skills/create-automation/SKILL.md`

If the skill changes, update the documentation and any installed plugin cache copy.
