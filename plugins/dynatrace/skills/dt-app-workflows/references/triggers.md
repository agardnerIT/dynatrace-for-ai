# Workflow Triggers

A workflow runs when its `trigger` fires. There are three trigger families: **schedule**, **event**, and **manual** (no trigger block — only run via `dtctl exec`).

## Schedule Trigger

```yaml
trigger:
  schedule:
    isActive: true                  # false = disabled
    isFaulty: false
    nextExecution: null             # set by the server; do not author
    rule: null                      # Business Calendar UUID — NOT a cron string
    timezone: UTC                   # IANA zone, e.g. Europe/Berlin
    trigger:
      type: cron
      cron: "0 * * * *"             # standard 5-field cron
```

### Common cron values

| Cron | Meaning |
|---|---|
| `*/5 * * * *` | every 5 minutes |
| `0 * * * *` | top of every hour |
| `0 9 * * 1-5` | 09:00 weekdays |
| `0 0 * * *` | midnight daily |

### Time-of-day form

For "once per day at HH:MM in the given timezone":

```yaml
trigger:
  schedule:
    isActive: true
    timezone: Europe/Berlin
    trigger:
      type: time
      time: "09:30"
```

### Business Calendar (rule:)

`trigger.schedule.rule` accepts the UUID of a **Business Calendar** — a configured "business hours" definition that limits when the schedule may fire. Leave `null` to fire on the cron alone. **Never put a cron string here** — it must be a UUID or null (`Must be a valid UUID.` error).

## Event Trigger

Fires on Davis events (problems, custom events, etc.):

```yaml
trigger:
  eventTrigger:
    isActive: true
    triggerConfiguration:
      type: davis-problem
      value:
        categories: [error, slowdown, availability, resource, custom]
        entityTagsMatch: any
        entityTags: []
        onProblemClose: false
```

Inside tasks, the firing event is available as `event()` (e.g. `{{ event()['event.name'] }}`). Useful for problem-routing workflows.

For DQL-based custom triggers:

```yaml
trigger:
  eventTrigger:
    isActive: true
    triggerConfiguration:
      type: event
      value:
        query: "fetch events | filter event.kind == 'CUSTOM_INFO' and event.name == 'deployment'"
```

## Manual / No Trigger

Omit the `trigger` block entirely. The workflow only runs via `dtctl exec workflow <id>` or the UI "Run" button. Useful for runbook-style workflows triggered by humans.

## Verifying What Fires When

```bash
dtctl get workflow <id> -o yaml --plain | sed -n '/^trigger:/,$p'
```

After applying, the server may compute `nextExecution:` for scheduled workflows — re-download to see when the next firing is.
