---
name: port-self-service-actions
description: >
  Port.io self-service actions expert. Use when creating, modifying, or debugging Port self-service
  actions: defining action types (Create/Delete/Day-2), configuring user input forms (13 input types),
  setting up backends (GitHub Actions, GitLab, Webhook, Azure DevOps), configuring RBAC, adding
  approval flows, and writing the action JSON schema. Essential when the user mentions Port actions,
  self-service, scaffolding, runbooks, or action backends.
license: Apache-2.0
metadata:
  author: port-io
  version: "1.0"
compatibility: Designed for use with Port.io (port.io).
---

# Port Self-Service Actions

Self-service actions let you expose infrastructure operations (scaffold, scale, restart, provision) as wizard-like experiences tied to catalog blueprints.

## Action Types

| Type | Use case |
|---|---|
| `CREATE` | Provision a new resource (no existing entity required) |
| `DELETE` | Tear down an existing entity |
| `DAY-2` | Modify or operate on an existing entity |

## Full Action JSON Schema

```json
{
  "identifier": "scaffold_service",
  "title": "Scaffold New Service",
  "icon": "Git",
  "description": "Creates a new microservice from template",
  "trigger": {
    "type": "self-service",
    "operation": "CREATE",
    "blueprintIdentifier": "microservice",
    "userInputs": {
      "properties": {
        "service_name": {
          "type": "string",
          "title": "Service Name",
          "description": "Lowercase, hyphens only"
        },
        "language": {
          "type": "string",
          "title": "Language",
          "enum": ["python", "go", "typescript", "java"]
        },
        "team": {
          "type": "string",
          "format": "team",
          "title": "Owning Team"
        }
      },
      "required": ["service_name", "language"]
    }
  },
  "invocationMethod": { ... },  // see backends below
  "requiredApproval": false
}
```

## User Input Types

All 13 input types share the same base fields: `type`, `title`, `description`, `default`, `icon`.

```json
"name":        { "type": "string",  "title": "Name" },
"replicas":    { "type": "number",  "title": "Replicas" },
"is_dry_run":  { "type": "boolean", "title": "Dry Run" },
"config":      { "type": "object",  "title": "Config" },
"env_vars":    { "type": "array",   "title": "Env Vars", "items": { "type": "string" } },
"manifest":    { "type": "string",  "format": "yaml",    "title": "K8s Manifest" },
"repo_url":    { "type": "string",  "format": "url",     "title": "Repo URL" },
"notify_email":{ "type": "string",  "format": "email",   "title": "Notify Email" },
"owner":       { "type": "string",  "format": "user",    "title": "Owner" },
"team":        { "type": "string",  "format": "team",    "title": "Team" },
"deploy_time": { "type": "string",  "format": "date-time", "title": "Deploy Time" },
"api_key":     { "type": "string",  "format": "secret",  "title": "API Key" }
```

### Entity selector input

Let users pick from existing catalog entities:

```json
"target_cluster": {
  "type": "string",
  "format": "entity",
  "title": "Target Cluster",
  "blueprint": "k8sCluster",
  "dataset": {
    "combinator": "and",
    "rules": [
      { "property": "environment", "operator": "=", "value": "production" }
    ]
  }
}
```

## Backends

### GitHub Actions

```json
"invocationMethod": {
  "type": "GITHUB",
  "org": "my-org",
  "repo": "port-actions",
  "workflow": "scaffold-service.yml",
  "workflowInputs": {
    "service_name": "{{ .inputs.service_name }}",
    "blueprint_id": "{{ .entity.identifier }}",
    "port_run_id":  "{{ .run.id }}"
  },
  "reportWorkflowStatus": true
}
```

The workflow must have `workflow_dispatch` trigger. Max 10 input parameters.
Port auto-reports success/failure unless `reportWorkflowStatus: false`.

### Webhook

```json
"invocationMethod": {
  "type": "WEBHOOK",
  "url": "https://hooks.example.com/port",
  "agent": false,
  "synchronized": false,
  "method": "POST",
  "headers": {
    "Authorization": "Bearer {{ .secrets.WEBHOOK_SECRET }}"
  },
  "body": {
    "runId": "{{ .run.id }}",
    "entityId": "{{ .entity.identifier }}",
    "inputs": "{{ .inputs }}"
  }
}
```

### GitLab Pipeline

```json
"invocationMethod": {
  "type": "GITLAB",
  "groupName": "my-group",
  "projectName": "port-actions",
  "defaultRef": "main",
  "pipelineVariables": {
    "SERVICE_NAME": "{{ .inputs.service_name }}",
    "PORT_RUN_ID": "{{ .run.id }}"
  }
}
```

### Azure DevOps

```json
"invocationMethod": {
  "type": "AZURE-DEVOPS",
  "org": "my-org",
  "webhook": "port-webhook",
  "payload": {
    "service": "{{ .inputs.service_name }}",
    "run_id": "{{ .run.id }}"
  }
}
```

## Approval Flows

```json
"requiredApproval": true
```

Or with conditions:

```json
"requiredApproval": {
  "type": "ANY_TEAM_MEMBER"
}
```

## RBAC

Control who can execute an action:

```json
"trigger": {
  ...
  "condition": {
    "type": "SEARCH",
    "rules": [
      {
        "operator": "relatedTo",
        "blueprint": "team",
        "value": "{{ .user.teams[0] }}"
      }
    ],
    "combinator": "and"
  }
}
```

Dynamic permission via jqQuery on user properties:

```json
"requiredApproval": {
  "type": "TEAM_MEMBERSHIP",
  "jqQuery": ".user.teams | map(.name) | contains([\"platform-team\"])"
}
```

## Port Action Context Variables

Available in `workflowInputs`, `body`, and payload templates:

| Variable | Value |
|---|---|
| `{{ .run.id }}` | Action run ID (use to report status back) |
| `{{ .inputs.<key> }}` | User-provided input value |
| `{{ .entity.identifier }}` | Target entity identifier (Day-2/Delete) |
| `{{ .entity.title }}` | Target entity title |
| `{{ .entity.properties.<key> }}` | Target entity property |
| `{{ .user.identifier }}` | User who triggered the action |
| `{{ .user.email }}` | Triggering user's email |
| `{{ .secrets.<name> }}` | Port secret value |

## Gotchas

- GitHub Actions backend requires the workflow file to exist on the **default branch** even when triggering from a non-default branch via `ref`.
- GitHub Actions supports **max 10 input parameters**.
- `workflow_dispatch` trigger must be present in the workflow YAML.
- Use `{{ .run.id }}` in the payload so your backend can report action status back to Port.
- `operation: "DAY-2"` and `operation: "DELETE"` require `blueprintIdentifier` to be set.
- Secret inputs (`format: "secret"`) are encrypted at rest; never log them in workflows.

## Further Reference

See [references/REFERENCE.md](references/REFERENCE.md) for complete invocation method schemas and reporting patterns.
