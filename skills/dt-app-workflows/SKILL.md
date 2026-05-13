---
name: dt-app-workflows
description: Work with Dynatrace Workflows (Automation app) — create, modify, query, and analyze workflow YAML including tasks, triggers (schedule/event), conditions, and the action catalog.
license: Apache-2.0
---

# Dynatrace Workflows Skill

## Overview

Dynatrace Workflows are YAML/JSON documents in the Automation engine that wire together **tasks** (actions like DQL queries, JavaScript, email, HTTP calls) on a directed graph and fire them on a **trigger** (cron schedule, Davis event, manual run).

**When to use:** Creating, modifying, debugging, or executing workflows.

## Workflow YAML Structure

```yaml
id: <uuid>                          # present when updating; omit on first create
title: My Workflow
description: ...
private: false
tasks:
  task_a:
    name: task_a
    action: dynatrace.automations:execute-dql-query
    description: ...
    input: { query: "..." }
    position: { x: 0, y: 1 }
    predecessors: []
  task_b:
    name: task_b
    action: dynatrace.email:send-email
    input: { to: [...], cc: [], bcc: [], subject: "...", content: "..." }
    conditions:
      states: { task_a: OK }
      custom: "{{ result('task_a').records | length > 0 }}"
    position: { x: 0, y: 2 }
    predecessors: [task_a]
trigger:
  schedule:
    isActive: true
    isFaulty: false
    nextExecution: null
    rule: null                      # Business Calendar UUID — NOT a cron string
    timezone: UTC
    trigger:
      type: cron
      cron: "0 * * * *"
```

- Task IDs in `tasks` are also the names used in `predecessors` and inside `result('<name>')` expressions
- `position` is for the UI graph; only required to be present, exact values rarely matter
- See [assets/ExampleWorkflow.yaml](assets/ExampleWorkflow.yaml) for a complete working example

## Mandatory Workflow Creation Rules

1. **Discover before guessing.** Before using any unfamiliar action name or input shape, find an existing workflow in the tenant that uses it:
   ```bash
   dtctl get workflows --plain | jq -r '.[] | .id + " " + .title'
   dtctl get workflow <id> -o yaml --plain
   ```
   Action names and input schemas vary across Dynatrace versions and which Hub apps are installed — memory is unreliable.
2. **Validate DQL before embedding** — same rule as dashboards. Run each query with `dtctl query '<DQL>' --plain` first.
3. **Workflows need explicit time bounds.** Unlike dashboards (UI time picker) or DQL run interactively, workflow queries have no implicit timeframe. Add `from: now() - 10m` (or similar) to the query or use the action's `timeframeSelector` input.
4. **First apply uses `--write-id`**; subsequent applies use the embedded `id`:
   ```bash
   dtctl apply -f workflow.yaml --write-id   # create + stamp id back
   dtctl apply -f workflow.yaml              # update in place
   ```
5. **Test end-to-end before trusting it.** `dtctl exec workflow <id> --wait --show-results --timeout 5m -o yaml --plain` — confirm each task's state (`SUCCESS` / `DISCARDED` / `ERROR`) makes sense.

Full workflow detailed in [references/create-update.md](references/create-update.md).

## Key References

| File | When to Load |
|------|-------------|
| [create-update.md](references/create-update.md) | Building/updating workflows, deploy/test patterns, anti-patterns |
| [actions.md](references/actions.md) | Finding the right action; common action catalog and input shapes |
| [triggers.md](references/triggers.md) | Schedule (cron/time) vs event (Davis problem, custom event) triggers |
| [conditions.md](references/conditions.md) | Task-level conditions: `states:`, `custom:` Jinja, `result()` accessor |

## Token Scopes

- `automation:workflows:read` — list and get workflows
- `automation:workflows:write` — create and update workflows
- `automation:workflows:run` — execute workflows (manual + scheduled)
- Plus whatever the **tasks** need at runtime — e.g. `storage:metrics:read` for a DQL task, an installed Email connector for `dynatrace.email:send-email`

## Quick Anti-Patterns

- Guessing action names from memory instead of introspecting the tenant
- Putting a cron string in `trigger.schedule.rule` (it expects a Business Calendar UUID — use `trigger.schedule.trigger.cron` instead)
- Forgetting `cc: []` / `bcc: []` on `dynatrace.email:send-email` — they're required even when empty
- Omitting timeframe bounds on `execute-dql-query` (defaults to a wide window that may include stale series)
- Treating `DISCARDED` as failure — it means a `conditions.custom` expression evaluated to false; usually that's correct behavior
