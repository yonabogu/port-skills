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
| Target entity | Required (user selects or pre-filled) | Not applicable |
| `blueprintIdentifier` | Required | Optional |
| Context available | `{{ .entity.* }}` | Not available |

## Scale Deployment Action

```json
{
  "identifier": "scale_deployment",
  "title": "Scale Deployment",
  "icon": "Kubernetes",
  "trigger": {
    "type": "self-service",
    "operation": "DAY-2",
    "blueprintIdentifier": "workload",
    "userInputs": {
      "properties": {
        "replicas": {
          "type": "number",
          "title": "Desired Replicas",
          "default": 2,
          "minimum": 1,
          "maximum": 20
        }
      },
      "required": ["replicas"]
    }
  },
  "invocationMethod": {
    "type": "GITHUB",
    "org": "my-org",
    "repo": "platform-actions",
    "workflow": "scale-deployment.yml",
    "workflowInputs": {
      "workload_id":  "{{ .entity.identifier }}",
      "namespace":    "{{ .entity.properties.namespace }}",
      "cluster":      "{{ .entity.properties.cluster }}",
      "replicas":     "{{ .inputs.replicas }}",
      "port_run_id":  "{{ .run.id }}"
    }
  }
}
```

## Restart Service Action

```json
{
  "identifier": "restart_service",
  "title": "Restart Service",
  "icon": "RefreshCw",
  "trigger": {
    "type": "self-service",
    "operation": "DAY-2",
    "blueprintIdentifier": "workload",
    "userInputs": {
      "properties": {
        "reason": { "type": "string", "title": "Reason for restart" }
      },
      "required": []
    }
  },
  "invocationMethod": {
    "type": "GITHUB",
    "org": "my-org",
    "repo": "platform-actions",
    "workflow": "restart-deployment.yml",
    "workflowInputs": {
      "workload_id": "{{ .entity.identifier }}",
      "namespace":   "{{ .entity.properties.namespace }}",
      "cluster":     "{{ .entity.properties.cluster }}",
      "reason":      "{{ .inputs.reason }}",
      "port_run_id": "{{ .run.id }}"
    }
  }
}
```

## Rollback Action

```json
{
  "identifier": "rollback_deployment",
  "title": "Rollback Deployment",
  "icon": "Undo",
  "trigger": {
    "type": "self-service",
    "operation": "DAY-2",
    "blueprintIdentifier": "workload",
    "userInputs": {
      "properties": {
        "target_version": {
          "type": "string",
          "title": "Target Image Tag / SHA",
          "description": "e.g. v1.2.3 or abc1234"
        },
        "confirm": {
          "type": "boolean",
          "title": "I confirm this will cause a service restart",
          "default": false
        }
      },
      "required": ["target_version", "confirm"]
    },
    "condition": {
      "type": "SEARCH",
      "rules": [
        { "property": "environment", "operator": "=", "value": "production" }
      ],
      "combinator": "and"
    }
  },
  "invocationMethod": {
    "type": "GITHUB",
    "org": "my-org",
    "repo": "platform-actions",
    "workflow": "rollback-deployment.yml",
    "workflowInputs": {
      "workload_id":     "{{ .entity.identifier }}",
      "target_version":  "{{ .inputs.target_version }}",
      "port_run_id":     "{{ .run.id }}"
    }
  },
  "requiredApproval": true
}
```

## Update Environment Config Action

```json
{
  "identifier": "update_env_config",
  "title": "Update Environment Config",
  "icon": "Settings",
  "trigger": {
    "type": "self-service",
    "operation": "DAY-2",
    "blueprintIdentifier": "microservice",
    "userInputs": {
      "properties": {
        "env_key":   { "type": "string", "title": "Config Key" },
        "env_value": { "type": "string", "title": "Config Value" },
        "environment": {
          "type": "string",
          "title": "Environment",
          "enum": ["development", "staging", "production"]
        }
      },
      "required": ["env_key", "env_value", "environment"]
    }
  },
  "invocationMethod": {
    "type": "WEBHOOK",
    "url": "https://api.example.com/config-change",
    "method": "POST",
    "headers": { "Authorization": "Bearer {{ .secrets.CONFIG_API_TOKEN }}" },
    "body": {
      "service":     "{{ .entity.identifier }}",
      "environment": "{{ .inputs.environment }}",
      "key":         "{{ .inputs.env_key }}",
      "value":       "{{ .inputs.env_value }}",
      "run_id":      "{{ .run.id }}"
    }
  }
}
```

## Updating Entity State After Day-2 Action

After your backend completes the operation, update the entity to reflect the new state:

```bash
# Update entity after scale
curl -X PATCH "https://api.port.io/v1/blueprints/workload/entities/$WORKLOAD_ID?merge=true" \
  -H "Authorization: Bearer $PORT_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"properties\": {\"replicas\": $NEW_REPLICAS}}"

# Then mark the action run as successful
curl -X PATCH "https://api.port.io/v1/actions/runs/$PORT_RUN_ID" \
  -H "Authorization: Bearer $PORT_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"status\": \"SUCCESS\", \"message\": {\"run_status\": \"Scaled to $NEW_REPLICAS replicas\"}}"
```

## Gotchas

- DAY-2 actions without a `condition` appear on **every** entity of the blueprint — add conditions to limit visibility (e.g., only for production environments).
- `requiredApproval: true` on destructive Day-2 actions (rollback, scale to 0) is strongly recommended.
- Use `{{ .entity.properties.<key> }}` in `workflowInputs` to pass the current entity state to the backend without requiring the user to re-enter known values.
- Always pass `port_run_id` to your backend so it can report success/failure; without it the action run stays "In Progress" indefinitely.
