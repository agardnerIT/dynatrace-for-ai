# Task Conditions

A task's `conditions:` block gates whether it runs. Two sub-fields, combined with AND:

```yaml
conditions:
  states:
    upstream_task: OK              # only run if upstream finished OK
  custom: "{{ result('upstream_task').records | length > 0 }}"
```

If either fails, the task is `DISCARDED` (not `ERROR`). A discarded downstream does **not** fail the workflow.

## states:

```yaml
conditions:
  states:
    task_a: OK                     # task_a must have succeeded
    task_b: ERROR                  # task_b must have errored — useful for "run on failure" paths
```

Allowed values:

| State | Meaning |
|---|---|
| `OK` | Predecessor succeeded |
| `ERROR` | Predecessor errored (useful for `on_error` handlers) |
| `ANY` | Any terminal state (default if you don't specify) |

Use to model "send pager only if the cleanup task failed", etc.

## custom:

A Jinja-style template that must evaluate to truthy. Common patterns:

```yaml
custom: "{{ result('query_task').records | length > 0 }}"
custom: "{{ result('query_task').records[0].count > 100 }}"
custom: "{{ event()['event.category'] == 'AVAILABILITY' }}"
```

### `result('<task>')` shape

For `execute-dql-query` tasks:

```yaml
result('query_task'):
  records:                         # array — main payload
    - { field1: ..., field2: ... }
  types: [...]
  metadata: { grail: { ... } }
```

So:
- `result('q').records` — full array
- `result('q').records | length` — row count
- `result('q').records[0].field1` — first row's field
- `result('q').records | map(attribute='name') | list` — pluck a field

For `run-javascript` tasks, `result('js_task')` is whatever the function `return`ed.

### `event()` (event-triggered workflows only)

The firing Davis event, with bracket access for dotted keys:

```yaml
custom: "{{ event()['event.category'] in ['ERROR', 'AVAILABILITY'] }}"
```

## Reading the DISCARDED state

`dtctl exec workflow <id> --wait --show-results` reports each task. A `[DISCARDED]` line means the conditions blocked it — usually correct behavior. Two ways to debug:

1. Re-read the `custom:` expression. Is the path right (`records` vs `records[0]`)? Is the type right (string vs number)?
2. Temporarily lower the threshold or weaken the condition to force a `SUCCESS`, then revert. We did this when testing the cert-expiry workflow: relaxed `< 2` to `< 100` to force the email path to fire, confirmed it succeeded, then reverted.

## Common Gotchas

- Forgetting `conditions:` entirely — the task runs every time the workflow does, even if the upstream returned nothing useful
- Writing `result('q').records.length` (JS-style) instead of `result('q').records | length` (Jinja-style) — both can fail silently
- Templating in `subject:` / `content:` follows the same Jinja rules as `custom:` — `{{ ... }}` for expressions, `{% for x in ... %}` for loops
