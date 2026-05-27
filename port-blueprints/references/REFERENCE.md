# Port Blueprints — Reference

## All Property Types Quick Reference

| Type | Format | Notes |
|---|---|---|
| `string` | — | Plain text |
| `string` | `url` | URL with clickable link in UI |
| `string` | `email` | Email address |
| `string` | `user` | Port user (renders as avatar) |
| `string` | `team` | Port team |
| `string` | `date-time` | ISO 8601 datetime |
| `string` | `timer` | Countdown timer; triggers automation on expiry |
| `string` | `markdown` | Rendered markdown |
| `string` | `yaml` | YAML editor |
| `number` | — | Integer or float |
| `boolean` | — | True/false toggle |
| `object` | — | Free-form JSON object |
| `array` | — | Array; specify `items.type` |
| `string` | `proto` | Protobuf schema |

## Calculation Property Output Types

Supported `type` + `format` combinations for calculation properties:

| type | format |
|---|---|
| `string` | — |
| `string` | `url`, `email`, `user`, `team`, `yaml`, `markdown` |
| `number` | — |
| `boolean` | — |
| `object` | — |
| `array` | — |

## Blueprint API Examples

### Create a blueprint

```bash
curl -X POST https://api.port.io/v1/blueprints \
  -H "Authorization: Bearer $PORT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "identifier": "microservice",
    "title": "Microservice",
    "icon": "Microservice",
    "schema": {
      "properties": {
        "language": { "type": "string", "title": "Language" },
        "tier": {
          "type": "string",
          "title": "Tier",
          "enum": ["tier-1", "tier-2", "tier-3"],
          "enumColors": { "tier-1": "red", "tier-2": "yellow", "tier-3": "green" }
        }
      },
      "required": ["language"]
    },
    "calculationProperties": {},
    "relations": {}
  }'
```

### Add a property to an existing blueprint

```bash
curl -X PATCH https://api.port.io/v1/blueprints/microservice \
  -H "Authorization: Bearer $PORT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "schema": {
      "properties": {
        "on_call": { "type": "string", "format": "user", "title": "On-Call" }
      }
    }
  }'
```

### Add a relation

```bash
curl -X PATCH https://api.port.io/v1/blueprints/microservice \
  -H "Authorization: Bearer $PORT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "relations": {
      "cluster": {
        "title": "Deployed On",
        "target": "k8sCluster",
        "required": false,
        "many": false
      }
    }
  }'
```

## Common Blueprint Patterns

### Service with team ownership and scorecard fields

```json
{
  "identifier": "service",
  "title": "Service",
  "schema": {
    "properties": {
      "language":   { "type": "string", "title": "Language" },
      "on_call":    { "type": "string", "format": "user",   "title": "On-Call" },
      "readme":     { "type": "string", "format": "url",    "title": "README" },
      "has_tests":  { "type": "boolean", "title": "Has Tests" },
      "pagerduty_service": { "type": "string", "format": "url", "title": "PagerDuty" }
    }
  },
  "calculationProperties": {},
  "relations": {
    "team": { "title": "Team", "target": "team", "required": false, "many": false }
  }
}
```
