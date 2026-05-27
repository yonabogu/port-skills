---
name: port-automations
description: >
  Port.io automations expert. Use when creating, modifying, or debugging Port automations:
  defining trigger types (ENTITY_CREATED, ENTITY_UPDATED, ENTITY_DELETED, TIMER_PROPERTY_EXPIRED,
  ANY_ENTITY_CHANGE), writing JQ conditions, configuring invocation methods (webhook, GitHub Actions,
  GitLab), chaining automation workflows, and managing automation JSON schema. Essential when the
  user mentions Port automations, event-driven workflows, automated responses, or catalog event triggers.
license: Apache-2.0
metadata:
  author: port-io
  version: "1.0"
compatibility: Designed for use with Port.io (port.io).
---

# Port Automations

Automations execute backend workflows automatically in response to catalog events.

## Automation JSON Schema

```json
{
  "identifier": "<automation-identifier>",
  "title": "<Human Readable Title>",
  "trigger": {
    "type": "automation",
    "event": {
      "type": "ENTITY_CREATED",          // see Trigger Types below
      "blueprintIdentifier": "<your-blueprint>"
    },
    "condition": {                        // optional: omit to fire on every event
      "type": "JQ",
      "expressions": [
        "<jq-expression-returning-boolean>"
      ],
      "combinator": "and"
    }
  },
  "invocationMethod": { ... },           // same options as self-service actions
  "publish": true                        // false = draft, won't fire
}
```

## Trigger Types

| Event type | Fires when |
|---|---|
| `ENTITY_CREATED` | A new entity is created in the blueprint |
| `ENTITY_UPDATED` | Any property/relation on an entity changes |
| `ENTITY_DELETED` | An entity is deleted |
| `TIMER_PROPERTY_EXPIRED` | A `timer`-format property reaches its deadline |
| `ANY_ENTITY_CHANGE` | Created, updated, or deleted (catches all) |

All triggers require `blueprintIdentifier`.

For `TIMER_PROPERTY_EXPIRED`, also specify which property:

```json
"event": {
  "type": "TIMER_PROPERTY_EXPIRED",
  "blueprintIdentifier": "environment",
  "propertyIdentifier": "ttl"
}
```

## JQ Conditions

Conditions narrow which events fire the automation. Evaluated against the event diff payload.

### Accessing changed values

```json
"condition": {
  "type": "JQ",
  "expressions": [
    ".diff.after.properties.status == \"failed\""
  ],
  "combinator": "and"
}
```

### Detecting a specific property changed

```json
"expressions": [
  ".diff.before.properties.on_call != .diff.after.properties.on_call"
]
```

### Checking scorecard level transition

```json
"expressions": [
  ".diff.after.scorecards.ProductionReadiness.level == \"Gold\"",
  ".diff.before.scorecards.ProductionReadiness.level != \"Gold\""
]
```

### Checking relation set

```json
"expressions": [
  ".diff.after.relations.team != null"
]
```

## Invocation Methods

Automations share the same invocation methods as self-service actions:

```json
// Webhook
{
  "type": "WEBHOOK",
  "url": "<your-webhook-url>",
  "method": "POST",
  "headers": { "Authorization": "Bearer {{ .secrets.<secret-name> }}" },
  "body": {
    "entity_id":  "{{ .event.context.entityIdentifier }}",
    "blueprint":  "{{ .event.context.blueprintIdentifier }}",
    "event_type": "{{ .event.action }}"
  }
}

// GitHub Actions
{
  "type": "GITHUB",
  "org": "<your-github-org>",
  "repo": "<your-repo>",
  "workflow": "<workflow-file>.yml",
  "workflowInputs": {
    "entity_id": "{{ .event.context.entityIdentifier }}",
    "event_type": "{{ .event.action }}"
  }
}
```

## Event Payload Variables

| Variable | Value |
|---|---|
| `{{ .event.action }}` | `CREATE`, `UPDATE`, `DELETE`, `TIMER_EXPIRED` |
| `{{ .event.context.entityIdentifier }}` | Affected entity identifier |
| `{{ .event.context.blueprintIdentifier }}` | Blueprint identifier |
| `{{ .diff.before }}` | Full entity state before the change |
| `{{ .diff.after }}` | Full entity state after the change |

## Common Patterns

### Auto-cleanup on timer expiry

Trigger: `TIMER_PROPERTY_EXPIRED` on `<your-blueprint>`, property `<your-timer-property>`.
Use `{{ .event.context.entityIdentifier }}` to pass the expired entity to the backend.

```json
{
  "identifier": "<automation-id>",
  "trigger": {
    "type": "automation",
    "event": {
      "type": "TIMER_PROPERTY_EXPIRED",
      "blueprintIdentifier": "<your-blueprint>",
      "propertyIdentifier": "<your-timer-property>"
    }
  },
  "invocationMethod": {
    "type": "GITHUB",
    "org": "<your-github-org>",
    "repo": "<your-repo>",
    "workflow": "<workflow-file>.yml",
    "workflowInputs": {
      "entity_id": "{{ .event.context.entityIdentifier }}"
    }
  },
  "publish": true
}
```

### React to scorecard level change

Trigger: `ENTITY_UPDATED` on `<your-blueprint>`. Use JQ to detect the specific transition.
Replace `<scorecard-id>` and `<level-name>` with your scorecard identifier and level title.

```json
{
  "identifier": "<automation-id>",
  "trigger": {
    "type": "automation",
    "event": {
      "type": "ENTITY_UPDATED",
      "blueprintIdentifier": "<your-blueprint>"
    },
    "condition": {
      "type": "JQ",
      "expressions": [
        ".diff.before.scorecards.<scorecard-id>.level == \"<level-name>\"",
        ".diff.after.scorecards.<scorecard-id>.level != \"<level-name>\""
      ],
      "combinator": "and"
    }
  },
  "invocationMethod": {
    "type": "WEBHOOK",
    "url": "<your-webhook-url>"
  }
}
```

## Gotchas

- `publish: false` saves the automation as a draft — it won't fire until set to `true`.
- Automations cannot chain directly into other automations via `workflow_dispatch` (GitHub limitation — use separate triggers or an orchestration tool).
- JQ runtime runs in **UTC** — be careful with time-based expressions.
- The `TIMER_PROPERTY_EXPIRED` event fires at the timer's exact expiry time ± a small delay; don't assume sub-minute precision.
- Condition expressions are AND'd with `combinator: "and"`, OR'd with `combinator: "or"`. Omitting `condition` fires on every matching event.
