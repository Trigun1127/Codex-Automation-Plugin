---
name: create-automation
description: "Create, inspect, update, delete, explain, or troubleshoot Codex automations and the @automation plugin visibility path from a plain-English request."
---

# Create Automation

Use this skill when the user wants Codex to create, inspect, update, delete, or explain a Codex automation.

## Core Principle

Prefer the native Codex automation API. Do not write raw automation files unless the native automation tool/bridge is unavailable and the user accepts a local fallback.

The native automation path is the only path that reliably:

- creates the TOML definition
- updates the scheduler database mirror
- invalidates the Automations tab cache
- avoids encoding/schema mistakes

## Goal

Turn a vague request like:

- `set up a morning analytics run`
- `make this fire every weekday at 10 pm`
- `what does my automation do`

into a concrete automation spec with:

- `name`
- `schedule`
- `prompt`
- `execution environment`
- `workspace`
- `model`
- `reasoning effort`
- `status`

## Required workflow

1. Determine whether the request is:
   - create
   - inspect
   - update
   - delete
   - explain

2. Infer or confirm the minimum required fields:
   - automation name
   - schedule or frequency
   - target workspace
   - task prompt
   - local vs cloud intent when the environment supports both

3. Prefer the native Codex automation surface when available in the session.
   - First search for an `automation_update` tool.
   - If the tool exists, use it for create/update/delete/inspect.
   - If the user invoked this through `@automation`, still prefer the native tool; the plugin is instruction packaging, not the scheduler itself.
   - If no first-class automation tool is available, inspect existing local automation definitions before proposing a fallback edit.

4. When falling back to local files:
   - inspect the current automation layout first
   - preserve the existing schema shape
   - avoid changing unrelated automations
   - treat writes outside the current workspace as higher-risk and require the normal approval flow
   - write UTF-8 without BOM
   - update the scheduler database mirror if the existing app version uses one
   - tell the user this is a fallback and may require a UI refresh

5. Explain the result in plain terms:
   - what it runs
   - when it runs
   - where it runs
   - what still needs user action, if anything

## Schedule rules

- Prefer exact times over vague wording.
- Convert `every day at 9:30 am` into a daily schedule with explicit minute and hour.
- If the user gives two run times, treat that as two schedules or two automation entries unless the native surface clearly supports multiple triggers in one object.
- If the user says `weekdays`, do not schedule weekends.
- If the user says `every day`, include all seven days.
- Use the user's local timezone unless they specify otherwise.
- In current Codex desktop TOML, a daily local-time schedule is represented as weekly RRULE with all seven days, for example:

```toml
rrule = "RRULE:FREQ=WEEKLY;BYHOUR=22;BYMINUTE=0;BYDAY=SU,MO,TU,WE,TH,FR,SA"
```

- Do not manually add jitter. The app scheduler may add a small run jitter internally for daily/weekly/hourly schedules.

## Prompt rules

- Keep the automation prompt task-focused and operational.
- Include the workspace or project context when relevant.
- Avoid vague prompts like `do analytics`.
- Prefer prompts that specify inputs, outputs, and where artifacts should be saved.

## Inspection rules

When explaining an existing automation, report:

- `id`
- `name`
- `kind`
- `status`
- `schedule`
- `execution environment`
- `workspace`
- `model`
- `reasoning effort`
- prompt summary

## Native Tool Notes

If the native tool is exposed, it is normally named `automation_update`.

Expected modes:

- `create`
- `update`
- `delete`
- possibly `view` or `inspect`, depending on the exposed schema

Use the tool schema exactly as exposed in the session. Do not guess field names if the tool is available; inspect the schema first.

Native create/update data is usually camelCase, for example:

- `executionEnvironment`
- `reasoningEffort`
- `targetThreadId`
- `localEnvironmentConfigPath`

Raw TOML uses snake_case, for example:

- `execution_environment`
- `reasoning_effort`
- `target_thread_id`
- `local_environment_config_path`

Do not mix the native tool shape with the TOML shape.

## Plugin Mention Visibility

If the user can use the automation plugin in one thread but `@automation` does not appear in another thread's mention picker, treat it as plugin marketplace or composer-cache state before assuming the skill is unavailable.

Reliable cross-thread visibility requires both:

```toml
[marketplaces.automation-local]
source_type = "local"
source = 'C:\Users\you\path\to\automations plugin'

[plugins."automation@automation-local"]
enabled = true
```

The package root should include:

```text
.agents/plugins/marketplace.json
automation/.codex-plugin/plugin.json
automation/skills/create-automation/SKILL.md
```

The marketplace file should point to the plugin folder:

```json
{
  "name": "automation",
  "source": {
    "source": "local",
    "path": "./automation"
  }
}
```

Install from the package root folder, not only the nested `automation` folder. With the included marketplace file, the plugin store should show the registered local marketplace/folder as `Automation Local`. If the plugin was installed before `.agents/plugins/marketplace.json` existed, uninstall it, reselect the local package root in the plugin store, then reinstall it from the marketplace/folder entry.

After reinstalling or editing `config.toml`, restart Codex desktop or open a fresh thread so the composer `@` picker rebuilds its plugin list. A session may still have the automation capability loaded even while the picker cache is stale.

## Local Fallback Storage

Current Codex desktop local automations use:

```text
%USERPROFILE%\.codex\automations\<automation-id>\automation.toml
%USERPROFILE%\.codex\sqlite\codex-dev.db
```

The TOML file is the source of truth once any automation TOML exists. The SQLite row is a scheduler-state mirror used for fields such as `next_run_at` and `last_run_at`.

Important behavior:

- The app lists automations by reading `automation.toml` files first.
- The database alone is not enough to make an automation visible.
- If any TOML file exists, app startup does not migrate DB-only automations back into TOML.
- The parser silently ignores invalid TOML or schema-invalid automations.
- The directory name must match the TOML `id`.

## Automation Run Memory

When a scheduled automation runs, the automation runner may inject runtime instructions that tell the agent to use:

```text
$CODEX_HOME/automations/<automation_id>/memory.md
```

These instructions are run-context instructions, not fields in `automation.toml`.

The expected behavior is:

```text
Read memory.md first if present.
Create memory.md if missing.
Update memory.md before returning.
```

Use `memory.md` for continuity between recurring runs:

- last run time
- input source or target
- output artifacts
- previous snapshot or comparison baseline
- known quirks or failure notes
- next-run reminders

Do not store secrets, cookies, tokens, private keys, passwords, or unnecessary personal paths in `memory.md`. Do not treat `memory.md` as required for Automations tab visibility; `automation.toml` is still the scheduling and listing source of truth.

## Local Fallback TOML Schema

Cron automation example:

```toml
version = 1
id = "channel-analytics-10pm"
kind = "cron"
name = "Channel analytics 10 PM"
prompt = "Pull the latest public YouTube channel analytics, save the snapshot files, compare against the prior snapshot, and summarize what changed."
status = "ACTIVE"
rrule = "RRULE:FREQ=WEEKLY;BYHOUR=22;BYMINUTE=0;BYDAY=SU,MO,TU,WE,TH,FR,SA"
model = "gpt-5.5"
reasoning_effort = "medium"
execution_environment = "local"
cwds = ["C:\\Users\\you\\project"]
created_at = 1777351327467
updated_at = 1777351327467
```

Heartbeat automation example:

```toml
version = 1
id = "follow-up-heartbeat"
kind = "heartbeat"
name = "Follow-up heartbeat"
prompt = "Check whether this thread needs a follow-up."
status = "ACTIVE"
rrule = "RRULE:FREQ=HOURLY;INTERVAL=1"
model = "gpt-5.5"
reasoning_effort = "medium"
target_thread_id = "<thread-id>"
created_at = 1777351327467
updated_at = 1777351327467
```

Required cron fields:

- `version`
- `id`
- `kind = "cron"`
- `name`
- `prompt`
- `status`
- `rrule`
- `execution_environment`
- `cwds`
- `created_at`
- `updated_at`

Required heartbeat fields:

- `version`
- `id`
- `kind = "heartbeat"`
- `name`
- `prompt`
- `status`
- `rrule`
- `target_thread_id`
- `created_at`
- `updated_at`

## Encoding Requirement

If writing TOML manually, it must be UTF-8 without BOM.

This matters on Windows. Windows PowerShell `Set-Content -Encoding utf8` may write a BOM. A BOM at the start of `automation.toml` can make the Codex parser silently reject the file, causing the automation to exist on disk but not appear in the Automations tab.

Safe Python write pattern:

```python
from pathlib import Path

path = Path(r"C:\Users\you\.codex\automations\example\automation.toml")
text = "version = 1\n..."
path.parent.mkdir(parents=True, exist_ok=True)
path.write_text(text, encoding="utf-8", newline="\n")
assert not path.read_text(encoding="utf-8").startswith("\ufeff")
```

Safe verification:

```python
from pathlib import Path

path = Path(r"C:\Users\you\.codex\automations\example\automation.toml")
print(path.read_bytes()[:8].hex(" "))
print(path.read_text(encoding="utf-8").startswith("\ufeff"))
```

Expected first bytes for a valid file beginning with `version`:

```text
76 65 72 73 69 6f 6e 20
False
```

If the first bytes are `ef bb bf`, remove the BOM.

## Refresh Behavior

The Automations tab does not simply watch the filesystem.

Known refresh triggers:

- The tab queries `list-automations`.
- Native create/update/delete invalidates the `list-automations` query.
- Automation run events dispatch `automation-runs-updated`, which also invalidates `list-automations`.
- A manual file edit may appear after the tab refetches, but it does not emit the same native invalidation event.

If a fallback-created automation does not appear:

1. Verify the TOML has no BOM.
2. Verify the directory name equals the TOML `id`.
3. Verify required fields are present for `cron` or `heartbeat`.
4. Verify `status` is not `DELETED`.
5. Verify `%USERPROFILE%\.codex\automations\<id>\automation.toml` is the path used by the running Codex app.
6. Verify the app is not using a different `CODEX_HOME`.
7. Trigger a refresh by navigating away/back, creating/updating through the native UI, or restarting if necessary.

## Scheduler Database Mirror

If a manual fallback is unavoidable, inspect whether this app version uses:

```text
%USERPROFILE%\.codex\sqlite\codex-dev.db
```

The `automations` table can contain:

- `id`
- `name`
- `prompt`
- `status`
- `next_run_at`
- `last_run_at`
- `cwds`
- `rrule`
- `model`
- `reasoning_effort`
- `created_at`
- `updated_at`

The database row should mirror the TOML. Store `cwds` as JSON, for example:

```json
["C:\\Users\\you\\project"]
```

Do not rely on the DB row alone. It will not make the automation visible if the TOML is missing or invalid.

## Safety rules

- Do not silently overwrite an unrelated automation.
- Do not assume a global automation should be changed when the request sounds project-specific.
- If the only workable fallback is editing local automation files outside the current workspace, say that clearly and follow the required approval flow.
- Make the difference between a native app automation and a draft/local workaround explicit.
- Do not write to `%USERPROFILE%\.codex` without normal filesystem approval.
- Do not use Windows PowerShell text writing that may introduce a BOM.
- Do not modify the SQLite DB unless the TOML has also been written and validated.
- Do not delete automation directories unless the user explicitly asks.

## Output shape

When creating or updating, present the result as:

```md
Automation: <name>
Action: <created|updated|drafted|explained>
Schedule: <human-readable schedule>
Workspace: <path>
Execution: <local|cloud|unknown>
Prompt: <short summary>
Notes: <follow-up or limitation if any>
```

For fallback-created automations, include:

```md
Storage: TOML + scheduler DB mirror
Encoding: UTF-8 without BOM verified
Visibility: Should appear after the Automations tab refetches
```
