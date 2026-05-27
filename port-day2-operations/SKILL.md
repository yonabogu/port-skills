---
name: port-day2-operations
description: >
  Port.io Day-2 operations recipe. Use when building self-service actions for operational tasks
  in Port: scaling a deployment, restarting a service, rolling back to a previous version,
  changing environment variables, toggling feature flags, or any action that modifies an existing
  catalog entity. Covers action type DAY-2, backend configuration, user input design for operational
  actions, and updating entity state after action completion. Essential when the user wants to
  expose infrastructure operations (scale, restart, rollback, config change) as Port self-service.
license: Apache-2.0
metadata:
  author: port-io
  version: "1.0"
compatibility: Designed for use with Port.io (port.io). Requires GitHub Actions or webhook backend.
---

# Port Day-2 Operations

Day-2 actions operate on existing catalog entities — scale, restart, rollback, reconfigure.

## Key Differences from CREATE Actions

| Aspect | Day-2 | Create |
|---|---|---|
| `operation` field | `"DAY-2"` | `"CREATE"` |
| Target entity | Existing entity (user selects in UI) | Not applicable |
| `blueprintIdentifier` | Required | Optional |
| Entity context available | Yes — `{{ .entity.* }}` | No |

## Action Skeleton

Every Day-2 action follows this structure. Replace `<placeholders>` with values from the user's setup:

```json
{
  "identifier": "<action-identifier>",
  "title": "<Human Readable Title>",
  "trigger": {
    "type": "self-service",
    "operation": "DAY-2",
    "blueprintIdentifier": "<the-blueprint-this-action-targets>",
    "userInputs": {
      "properties": { ... },
      "required": [...]
    }
  },
  "invocationMethod": { ... },
  "requiredApproval": false
}
```

## Pattern 1: Forward Entity Context to the Backend

The most important Day-2 pattern: use `{{ .entity.* }}` to pass what Port already knows to the backend, so the user only inputs what's actually new.

```json
"workflowInputs": {
  "port_run_id": "{{ .run.id }}",

  // From the entity — user doesn't have to type these
  "entity_id":  "{{ .entity.identifier }}",
  "env":        "{{ .entity.properties.<environment-property-name> }}",
  "cluster":    "{{ .entity.properties.<cluster-property-name> }}",

  // From the user's form input
  "new_value":  "{{ .inputs.<input-name> }}"
}
```

Always include `port_run_id`. Without it the backend cannot report status back to Port and the action run stays "In Progress" forever.

## Pattern 2: Restrict Action Visibility with `condition`

Without a `condition`, a Day-2 action appears on **every** entity of the blueprint. Use `condition` to limit it:

```json
"trigger": {
  "operation": "DAY-2",
  "blueprintIdentifier": "<your-blueprint>",
  "condition": {
    "type": "SEARCH",
    "rules": [
      { "property": "<property-name>", "operator": "=", "value": "<restricting-value>" }
    ],
    "combinator": "and"
  }
}
```

Example use cases:
- Show a "promote to production" action only on entities where `environment = staging`
- Show a "scale down" action only where `replicas > 1`
- Hide a destructive action on entities owned by a different team

## Pattern 3: Require Approval for Destructive Actions

For anything irreversible (rollback, delete, scale to zero):

```json
"requiredApproval": true
```

Pair with a confirmation boolean input to make the user explicitly acknowledge the consequence:

```json
"userInputs": {
  "properties": {
    "target_version": { "type": "string", "title": "Target version to roll back to" },
    "confirm": {
      "type": "boolean",
      "title": "I understand this will restart the service",
      "default": false
    }
  },
  "required": ["target_version", "confirm"]
}
```

## Pattern 4: Update Entity State After the Action

Your backend should write the new state back to Port after completing the operation, so the catalog stays in sync:

```bash
# 1. Update the entity property that changed
curl -X PATCH "https://api.port.io/v1/blueprints/<blueprint>/entities/<entity-id>?merge=true" \
  -H "Authorization: Bearer $PORT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"properties": {"<changed-property>": <new-value>}}'

# 2. Report the action run result
curl -X PATCH "https://api.port.io/v1/actions/runs/$PORT_RUN_ID" \
  -H "Authorization: Bearer $PORT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"status": "SUCCESS", "message": {"run_status": "<what happened>"}}'
```

`merge=true` ensures only the updated property changes; all other entity fields are left untouched.

## Gotchas

- `operation: "DAY-2"` requires `blueprintIdentifier` — omitting it causes a validation error.
- Without `condition`, the action appears on every entity of the blueprint. Always consider whether it should be restricted.
- `{{ .entity.properties.<key> }}` only works if that property exists on the blueprint. If it doesn't, the value will be empty string — not null.
- `requiredApproval: true` blocks execution until an Admin or Moderator approves in the Port UI.
- The action run stays "In Progress" indefinitely if the backend never calls `PATCH /actions/runs/{runId}`. Always handle both success and failure paths.
