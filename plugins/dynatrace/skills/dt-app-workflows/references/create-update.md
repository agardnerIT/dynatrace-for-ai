# Workflow Create & Update Workflow

## Mandatory 6-Step Order

1. Define purpose: trigger type, tasks, audience for any notification
2. **Introspect the tenant** for action names and input shapes (`dtctl get workflows`, `dtctl get workflow <id> -o yaml`)
3. Validate any DQL the workflow will run with `dtctl query '<DQL>' --plain`
4. **(Update only)** Download the existing YAML before editing: `dtctl get workflow <id> -o yaml --plain > workflow.yaml`
5. Author the YAML (see structure in `SKILL.md`)
6. Deploy with `dtctl apply -f workflow.yaml [--write-id]` and test with `dtctl exec workflow <id> --wait --show-results`

---

## Deploy Patterns

### First-time create

```bash
dtctl apply -f workflow.yaml --write-id -o yaml --plain
```

The `--write-id` flag stamps the generated UUID back into the file as `id: <uuid>` at the top. Every subsequent apply then updates that workflow in place.

If `--write-id` reports "could not write ID back to file" (happens when the file is YAML but dtctl can't round-trip cleanly), manually prepend:

```yaml
id: <uuid-from-the-apply-output>
title: ...
```

### Update existing

```bash
dtctl get workflow <id> -o yaml --plain > workflow.yaml   # always download first
# edit workflow.yaml
dtctl apply -f workflow.yaml -o yaml --plain
```

Downloading first preserves any edits made through the Dynatrace UI since the last local apply. Skipping it silently overwrites UI changes.

### Dry-run

```bash
dtctl apply -f workflow.yaml --dry-run -o yaml --plain
```

Validates schema only — does not catch action-not-installed or input-validation errors that only surface at execution time.

---

## Testing Pattern

```bash
dtctl exec workflow <id> --wait --show-results --timeout 5m -o yaml --plain
```

The output shows each task with one of:

| State | Meaning |
|---|---|
| `SUCCESS` | Task ran and completed without error |
| `DISCARDED` | Task was skipped because its `conditions` evaluated to false (usually expected, not a bug) |
| `ERROR` | Task failed — read the error message carefully |

### Common error patterns and what they mean

| Error | Cause | Fix |
|---|---|---|
| `Function \`/api/<x>\` not found. The app containing the action may have been uninstalled or the action no longer exists.` | Wrong action name **or** the app is not installed | Introspect an existing workflow to find the correct identifier; if no workflow uses it, install the app from Hub |
| `Error: Validation failed: - cc: Invalid input - bcc: Invalid input` | The action requires explicit input fields that you omitted | Pass `cc: []` / `bcc: []` (etc.) even when empty |
| `Must be a valid UUID.` on `trigger.schedule.rule` | Put a cron string into `rule:` | Set `rule: null` and use `trigger.schedule.trigger.cron` instead |

---

## DQL in Workflows: Time Bounds

DQL inside a workflow's `execute-dql-query` task has **no implicit timeframe** the way dashboards do. Always specify one:

```yaml
input:
  query: |
    timeseries v=avg(my.metric), by:{name}, from: now() - 10m
    | filter ...
```

Without it, the query defaults to a wide window (often hours or days) which can include stale series — leading to false-positive alerts. The cert-monitoring example caught a phantom alert this way: a series that stopped reporting still appeared in the 2h window with `arrayLast(...) = null`, which after arithmetic became a huge negative number that satisfied a `< 2 days` filter.

### Null-safety in workflow queries

For metrics that may have gaps or stale series:

```dql
| filter isNotNull(arrayLast(my_value))
```

---

## Anti-Patterns

- Guessing action names from memory — always introspect first
- Skipping the download step on update — overwrites UI edits
- No `from: now() - <window>` on workflow DQL queries — risks stale data
- Treating `DISCARDED` as failure — re-read the `conditions.custom` expression; a falsy result is usually correct
- Manually injecting an `id:` into a freshly-constructed file you intend as a "fresh create" — that flips it into an update
- Hardcoded secrets in `input:` — use the Credential Vault and `credentialVaultClient.getCredentialsDetails({id})` from a JS task
