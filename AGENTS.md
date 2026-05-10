# Agent Instructions For The Automation Skill

Use this skill package when a user asks to create, inspect, update, delete, or explain Codex desktop automations.

## Primary Rule

Use the native Codex automation tool if it is exposed. Search for `automation_update` first.

Only use local TOML/SQLite fallback when the native tool is unavailable and fallback file edits are appropriate.

## Installation Context

This repository is a skill package, not an app/tool plugin. Codex builds that distinguish skills from plugins may show it through the skill picker, not the `@` plugin picker.

Install the skill folder at:

```text
$CODEX_HOME/skills/create-automation/
```

The skill provides automation instructions. It does not add a scheduler by itself.

## Workflow

1. Determine the action: create, inspect, update, delete, or explain.
2. Identify schedule, workspace, model, reasoning effort, execution environment, and task prompt.
3. Prefer native `automation_update`.
4. If native tooling is unavailable, inspect local storage.
5. For fallback writes, update TOML first, then SQLite mirror if this Codex version uses one.
6. Verify the automation appears or explain what refresh is needed.

## Local Storage

Current local Codex desktop automation storage:

```text
%USERPROFILE%\.codex\automations\<automation-id>\automation.toml
%USERPROFILE%\.codex\sqlite\codex-dev.db
```

The TOML file is the source of truth for listing. The SQLite row is scheduler-state support.

## Critical Windows Encoding Rule

Write `automation.toml` as UTF-8 without BOM.

Do not use old Windows PowerShell `Set-Content -Encoding utf8` unless you verify the resulting bytes.

Expected valid first bytes for a file starting with `version = 1`:

```text
76 65 72 73 69 6f 6e 20
```

Invalid BOM first bytes:

```text
ef bb bf
```

If a BOM exists, Codex may silently ignore the automation.

## Fallback Safety

When deleting, remove only the exact automation folder and exact matching DB row.

Before recursive delete, resolve the absolute path and verify:

```text
target.parent == %USERPROFILE%\.codex\automations
```

Never delete broad paths.

## Output To User

Keep final answers concise:

```md
Automation: <name>
Action: <created|updated|deleted|explained>
Schedule: <human-readable schedule>
Workspace: <path>
Execution: <local|cloud|unknown>
Model: <model>
Reasoning: <reasoning effort>
Verification: <what was checked>
```
