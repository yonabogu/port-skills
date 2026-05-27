---
name: port-software-catalog
description: >
  Port.io software catalog expert. Use when creating, querying, or managing Port catalog entities:
  entity JSON structure (identifier, title, properties, relations, team), ingesting entities via
  REST API, upserting in bulk, querying with search rules, using integrations for auto-discovery,
  and understanding entity meta-properties ($identifier, $title, $createdAt, $updatedAt, $team).
  Essential when the user mentions Port entities, the software catalog, asset inventory, or
  bulk ingestion.
license: Apache-2.0
metadata:
  author: port-io
  version: "1.0"
compatibility: Designed for use with Port.io (port.io).
---

# Port Software Catalog

The software catalog stores all entities — instances of blueprints that represent your real infrastructure assets, services, and resources.

## Entity JSON Structure

```json
{
  "identifier": "payment-service",          // required, unique per blueprint, max 1000 chars
  "title": "Payment Service",               // required, human-readable
  "blueprint": "microservice",              // required, target blueprint
  "team": ["platform-team"],                // optional, array of team names
  "icon": "Microservice",                   // optional
  "properties": {
    "language": "go",
    "tier": "tier-1",
    "on_call": "alice@example.com",
    "readme_url": "https://github.com/my-org/payment-service#readme"
  },
  "relations": {
    "team": "platform-team",                // single relation: string identifier
    "dependencies": ["auth-service", "user-service"]  // many relation: array
  }
}
```

## Creating / Upserting Entities

### Single entity upsert

```bash
curl -X POST "https://api.port.io/v1/blueprints/microservice/entities?upsert=true&merge=true" \
  -H "Authorization: Bearer $PORT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "identifier": "payment-service",
    "title": "Payment Service",
    "properties": {
      "language": "go",
      "tier": "tier-1"
    },
    "relations": {
      "team": "platform-team"
    }
  }'
```

Key query params:
- `upsert=true` — create if not exists, update if exists
- `merge=true` — merge properties instead of replacing the full entity
- `create_missing_related_entities=true` — auto-create related entities if they don't exist

### Bulk upsert

```bash
curl -X POST "https://api.port.io/v1/blueprints/microservice/entities/bulk" \
  -H "Authorization: Bearer $PORT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "entities": [
      { "identifier": "svc-a", "title": "Service A", "properties": {} },
      { "identifier": "svc-b", "title": "Service B", "properties": {} }
    ]
  }'
```

## Reading Entities

```bash
# Get single entity
curl "https://api.port.io/v1/blueprints/microservice/entities/payment-service" \
  -H "Authorization: Bearer $PORT_TOKEN"

# List all entities of a blueprint
curl "https://api.port.io/v1/blueprints/microservice/entities" \
  -H "Authorization: Bearer $PORT_TOKEN"

# Search entities
curl -X POST "https://api.port.io/v1/entities/search" \
  -H "Authorization: Bearer $PORT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "rules": [
      { "property": "tier", "operator": "=", "value": "tier-1" },
      { "property": "on_call", "operator": "isEmpty" }
    ],
    "combinator": "and"
  }'
```

## Deleting Entities

```bash
# Delete single entity
curl -X DELETE "https://api.port.io/v1/blueprints/microservice/entities/payment-service" \
  -H "Authorization: Bearer $PORT_TOKEN"

# Delete with cascade (also deletes related entities)
curl -X DELETE "https://api.port.io/v1/blueprints/microservice/entities/payment-service?delete_dependents=true" \
  -H "Authorization: Bearer $PORT_TOKEN"
```

## Meta-Properties

Every entity has read-only meta-properties accessible in calculations, rules, and JQ:

| Meta-property | Type | Description |
|---|---|---|
| `$identifier` | string | Entity's unique identifier |
| `$title` | string | Entity's display title |
| `$blueprint` | string | Blueprint identifier |
| `$createdAt` | datetime | Creation timestamp (UTC) |
| `$updatedAt` | datetime | Last update timestamp (UTC) |
| `$team` | array | Owning teams |

Access in JQ: `.identifier`, `.title`, `.blueprint`, `.createdAt`, `.updatedAt`, `.team`

## Ingestion Methods

1. **Port REST API** — direct CRUD (shown above)
2. **CI/CD** — push entities from GitHub Actions / GitLab pipelines after deployments
3. **Ocean integrations** — continuous sync from external tools (GitHub, K8s, Jira, etc.)
4. **Webhooks** — real-time push from external systems via Port's webhook listener
5. **Port's UI** — manual entity creation through the catalog builder

## Search Rules Operators

Used in `/entities/search` and automation conditions:

```json
{ "property": "language",   "operator": "=",           "value": "go" },
{ "property": "tier",       "operator": "in",          "value": ["tier-1", "tier-2"] },
{ "property": "on_call",    "operator": "isEmpty" },
{ "property": "replicas",   "operator": ">",           "value": 3 },
{ "property": "$createdAt", "operator": "between",     "value": { "preset": "last7Days" } }
```

## Gotchas

- `identifier` is case-sensitive and immutable after creation — choose carefully.
- `merge=true` only updates provided fields; omitted properties keep their existing values.
- Without `upsert=true`, POSTing an entity that already exists returns a 409 conflict.
- Max request body size is **1 MiB** — use bulk or pagination for large datasets.
- `$team` (meta-property) and `team` (property defined on blueprint schema) are different things.
- The `team` field at the root of the entity JSON sets ownership; it must match existing Port team names.
