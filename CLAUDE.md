# Claude Instructions For The Automation Skill

This repository packages a Codex skill for working with Codex desktop automations.

## Purpose

Help an agent turn plain-English automation requests into reliable Codex desktop automations.

Examples:

- Create an automation every day at 9 AM.
- Delete an existing automation.
- Explain why an automation is not visible in the Automations tab.
- Recreate automations with a specific model and reasoning effort.

## Important Concept

This is a skill package, not an app/tool plugin. It does not directly provide the scheduler.

Codex builds that distinguish skills from plugins may show this package through the skill picker rather than the `@` plugin picker.

The agent should follow:

```text
skills/create-automation/SKILL.md
```

## Install Instructions For Users

1. Open the local Codex skills directory.
2. Copy `skills/create-automation/` into `$CODEX_HOME/skills/create-automation/`.
3. Restart Codex or open a fresh thread.
4. Invoke the skill from the skill picker or by skill mention if supported.

## Best Path

Always prefer the native Codex automation tool if available.

Search for:

```text
automation_update
```

If exposed, use it. It handles storage, scheduler state, and UI invalidation correctly.

## Fallback Path

If no native tool is exposed, use the local fallback documented in `SKILL.md`.

Fallback storage:

```text
%USERPROFILE%\.codex\automations\<automation-id>\automation.toml
%USERPROFILE%\.codex\sqlite\codex-dev.db
```

Write TOML as UTF-8 without BOM.

## Safety

- Do not edit broad automation directories.
- Do not delete anything unless the user explicitly asks.
- Do not store secrets in automation `memory.md`.
- Do not rely on the SQLite row alone for visibility.
- Prefer exact schedule wording and clear output expectations.
