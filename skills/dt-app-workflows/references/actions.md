# Workflow Actions

## Discovery is the rule, the catalog is the hint

Action identifiers (`dynatrace.<app>:<action>`) and their input schemas **vary by tenant** — which apps are installed, which versions, which connectors. Memorized lists go stale.

**Before using any action you haven't recently verified, find a workflow in the tenant that already uses it:**

```bash
# List workflows
dtctl get workflows --plain | jq -r '.[] | .id + " " + .title'

# Dump a candidate workflow to read its task definitions
dtctl get workflow <id> -o yaml --plain | sed -n '/^tasks:/,/^trigger:/p'

# Or grep all workflows for a keyword
for id in $(dtctl get workflows -o json --plain | jq -r '.[].id'); do
  dtctl get workflow "$id" -o yaml --plain 2>/dev/null \
    | grep -l "send-email\|send-slack\|http" >/dev/null \
    && echo "$id uses it"
done
```

The action's `input:` block in the existing workflow tells you exactly which fields are required.

## Common Actions (verify before using)

These are commonly available in Dynatrace SaaS but **the names can change** — treat as a starting hint, not gospel.

### DQL query

```yaml
action: dynatrace.automations:execute-dql-query
input:
  query: |
    timeseries avg(my.metric), by:{x}, from: now() - 10m
```

Task result accessed via `result('<task_name>').records` (an array of objects).

### JavaScript

```yaml
action: dynatrace.automations:run-javascript
input:
  script: |
    import { execution } from '@dynatrace-sdk/automation-utils';
    export default async function ({ executionId }) {
      const ex = await execution(executionId);
      const records = (await ex.result('query_task')).records;
      return { count: records.length };
    }
```

Use for: branching logic, calling external HTTP APIs, anything outside the built-in action catalog.

### Email

```yaml
action: dynatrace.email:send-email
input:
  to:      [user@example.com]
  cc:      []
  bcc:     []
  subject: "Subject with {{ result('q').records | length }} substitutions"
  content: |
    Plain-text body. Jinja templates work:
    {% for row in result('q').records %}
    - {{ row.name }}: {{ row.value }}
    {% endfor %}
```

**Requires the Email connector app installed in the tenant.** Watch out for:
- `cc:` and `bcc:` are required keys; pass `[]` if unused (omitting them gives `Validation failed: - cc: Invalid input`)
- Don't confuse with `dynatrace.automations:send-email` (different version/install) or `dynatrace.email:outgoing-email` (yet another older name) — both can 404 in tenants that have only one variant installed

### Davis ownership

```yaml
action: dynatrace.ownership:get-contact-details-from-owners
input: { entityId: "..." }
```

```yaml
action: dynatrace.ownership:get-ownership-from-entity
input: { entityId: "..." }
```

Useful for routing alerts to the team that owns the affected entity.

### HTTP

If neither built-in action nor a connector covers your destination (PagerDuty, internal API, etc.), use `run-javascript` with `fetch()` and read secrets from the Credential Vault:

```js
import { credentialVaultClient } from "@dynatrace-sdk/client-classic-environment-v2";
const apiKey = (await credentialVaultClient.getCredentialsDetails({ id: "CREDENTIALS_VAULT-XXXX" })).token;
await fetch("https://api.example.com/webhook", {
  method: "POST",
  headers: { Authorization: `Bearer ${apiKey}` },
  body: JSON.stringify({ ... }),
});
```

## Reading the failure message

| Failure text | Cause |
|---|---|
| `Function \`/api/<suffix>\` not found` | The `<namespace>:<suffix>` action is not installed — wrong name, or the Hub app isn't present |
| `Error: Validation failed: - <field>: Invalid input` | Action exists but a required input field is missing or malformed |
| Action runs but result is empty / wrong shape | Input was accepted but semantics were wrong — re-inspect a known-good workflow's input |

When in doubt, the safest move is to find a working workflow in the tenant that uses the same action and copy its `input:` block.
