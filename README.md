# Codex Automation Skill

This repository contains a Codex skill for creating, inspecting, updating, deleting, and troubleshooting Codex desktop automations.

Codex recently changed how plugin and skill surfaces appear in the composer. App/tool-backed plugins such as Browser, Gmail, Vercel, and Chrome can appear in the `@` picker because they provide a connector, native tool, bundled script, or other app capability. This automation package is instruction-based, so it is now provided as a skill rather than an `@` plugin.

Use this skill when you want Codex to work with automations through the native automation tool when available, or to understand the local automation storage format when a fallback is needed.

## Contents

- `skills/create-automation/SKILL.md`: the Codex skill instructions.
- `AGENTS.md`: short operational guidance for Codex-style agents.
- `CLAUDE.md`: short operational guidance for Claude-style agents.
- `README.md`: end-user documentation.

## Install

Install the skill by copying this folder into your Codex skills directory:

```text
$CODEX_HOME/skills/create-automation/
```

If `CODEX_HOME` is not set, Codex commonly uses:

```text
%USERPROFILE%\.codex\skills\create-automation\
```

After installation, restart Codex or open a fresh thread so the skill list refreshes.

## Use

Invoke the skill from the skill picker or mention it directly if your Codex build supports skill mentions:

```text
$create-automation
```

Example requests:

```text
Use $create-automation to create a weekday 9 AM automation for this project.
Use $create-automation to inspect my existing automations.
Use $create-automation to explain why an automation is not visible.
```

Do not expect this package to appear as `@Automation`. The `@` picker is currently oriented around plugins with app/tool capabilities. This repository intentionally stays skill-only instead of adding a fake MCP server or fake app connector just for presentation.

## How The Skill Works

Codex automations have two practical paths:

1. Native path: use Codex's native automation tool when available.
2. Local fallback path: inspect or write local automation files only when native tooling is unavailable and fallback file edits are appropriate.

The native path is preferred because it handles:

- schema validation
- automation file creation
- scheduler state
- Automations tab refresh behavior
- Windows encoding details

The native automation tool is usually exposed as:

```text
automation_update
```

Agents should search for that tool before editing files manually.

## Local Automation Storage

Codex desktop automations can be stored locally under:

```text
%USERPROFILE%\.codex\automations\<automation-id>\automation.toml
%USERPROFILE%\.codex\sqlite\codex-dev.db
```

The TOML file is the main source of truth for listing. The SQLite database row is scheduler-state support for fields such as `next_run_at` and `last_run_at`. A database row alone is not enough to make an automation visible in the Automations tab.

## Automation Run Memory

When a scheduled automation runs, the automation runner may provide runtime instructions that tell the agent to maintain a per-automation memory file:

```text
$CODEX_HOME/automations/<automation_id>/memory.md
```

These instructions are runtime context, not fields inside `automation.toml`.

The runtime pattern is:

```text
Read $CODEX_HOME/automations/<automation_id>/memory.md first if it exists.
Create it if it is missing.
Update it before returning.
```

The purpose is continuity across recurring runs. A useful `memory.md` can record:

- last successful run time
- input sources used
- output files or folders created
- previous snapshot or comparison baseline
- known quirks for the next run
- unresolved follow-up items

Do not store secrets in `memory.md`. Avoid tokens, cookies, private keys, passwords, or unnecessary personal paths. The required file for automation visibility and scheduling remains `automation.toml`.

## Visibility Rules

For an automation to appear in the Automations tab:

- The directory must be `%USERPROFILE%\.codex\automations\<id>`.
- The file must be named `automation.toml`.
- The TOML field `id` must exactly match the directory name.
- The file must parse successfully.
- The schema must validate.
- `status` must not be `DELETED`.
- The file must be UTF-8 without BOM.

The parser can silently ignore invalid files. If something does not appear, check encoding and schema first.

## Windows Encoding Rule

On Windows, avoid old Windows PowerShell `Set-Content -Encoding utf8` for automation TOML unless you verify the bytes afterward. Some PowerShell versions write a UTF-8 BOM, and a BOM can cause the automation parser to ignore the file.

Bad first bytes:

```text
ef bb bf
```

Good first bytes when the file starts with `version = 1`:

```text
76 65 72 73 69 6f 6e 20
```

Safe Python write pattern:

```python
from pathlib import Path

path = Path(r"C:\Users\you\.codex\automations\example\automation.toml")
text = "version = 1\n..."
path.parent.mkdir(parents=True, exist_ok=True)
path.write_text(text, encoding="utf-8", newline="\n")
assert not path.read_text(encoding="utf-8").startswith("\ufeff")
```

## Cron TOML Example

```toml
version = 1
id = "channel-analytics-10pm"
kind = "cron"
name = "Channel analytics 10 PM"
prompt = "Pull the latest public channel analytics, save snapshot files, compare against the prior snapshot, and summarize what changed."
status = "ACTIVE"
rrule = "RRULE:FREQ=WEEKLY;BYHOUR=22;BYMINUTE=0;BYDAY=SU,MO,TU,WE,TH,FR,SA"
model = "gpt-5.5"
reasoning_effort = "medium"
execution_environment = "local"
cwds = ["C:\\Users\\you\\project"]
created_at = 1777351327467
updated_at = 1777351327467
```

Daily schedules may be represented as weekly RRULEs with all seven days.

## Troubleshooting

If an automation does not appear:

1. Check the TOML first bytes for a BOM.
2. Check the directory name equals TOML `id`.
3. Check required fields exist.
4. Check `kind` is either `cron` or `heartbeat`.
5. Check `cwds` exists for cron automations.
6. Check `target_thread_id` exists for heartbeat automations.
7. Check the app is using the expected `%USERPROFILE%\.codex`.
8. Check the SQLite row only after the TOML is valid.
9. Trigger a UI refetch by navigating away/back or restarting.
