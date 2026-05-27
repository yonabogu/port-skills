# Port Automations — Reference

## Automation API

```bash
# Create automation
curl -X POST https://api.port.io/v1/automations \
  -H "Authorization: Bearer $PORT_TOKEN" \
  -H "Content-Type: application/json" \
  -d @automation.json

# List automations
curl https://api.port.io/v1/automations \
  -H "Authorization: Bearer $PORT_TOKEN"

# Enable / disable
curl -X PATCH https://api.port.io/v1/automations/{identifier} \
  -H "Authorization: Bearer $PORT_TOKEN" \
  -d '{"publish": false}'
```

## Full Event Payload Structure

```json
{
  "event": {
    "action": "UPDATE",
    "context": {
      "entityIdentifier": "my-service",
      "blueprintIdentifier": "microservice",
      "runId": null
    }
  },
  "diff": {
    "before": {
      "identifier": "my-service",
      "title": "My Service",
      "properties": { "status": "running" },
      "relations": {},
      "scorecards": {}
    },
    "after": {
      "identifier": "my-service",
      "title": "My Service",
      "properties": { "status": "failed" },
      "relations": {},
      "scorecards": {}
    }
  }
}
```

## Useful JQ Condition Patterns

```json
// Only fire if a specific property changed
".diff.before.properties.owner != .diff.after.properties.owner"

// Only fire for high-priority entities
".diff.after.properties.tier == \"tier-1\""

// Fire when entity first gets a team relation
".diff.before.relations.team == null and .diff.after.relations.team != null"

// Fire when CVE count increases
".diff.after.properties.critical_vulns > .diff.before.properties.critical_vulns"
```
