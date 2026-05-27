---
name: port-blueprints
description: >
  Port.io blueprint schema expert. Use when creating, modifying, or debugging Port blueprints:
  defining properties (string/number/boolean/array/object/URL/email/user/team/datetime/timer/
  markdown/YAML/enum/calculation/aggregation/mirror), configuring relations (single vs many),
  setting up mirror properties, aggregation properties, calculation properties, schema validation,
  and blueprint API calls (POST /blueprints, PATCH /blueprints/{id}). Essential when the user
  mentions Port blueprints, data model, catalog schema, or blueprint JSON.
license: Apache-2.0
metadata:
  author: port-io
  version: "1.0"
compatibility: Designed for use with Port.io (port.io). Requires access to Port's REST API or UI.
allowed-tools: WebFetch
---

# Port Blueprints

Blueprints are the data model layer of Port's software catalog. Every entity in Port is an instance of a blueprint.

## Blueprint JSON Schema

```json
{
  "identifier": "microservice",        // required, max 100 chars, immutable
  "title": "Microservice",             // required, human-readable
  "description": "A backend service",  // optional
  "icon": "Microservice",              // optional
  "schema": {
    "properties": { ... },             // required (can be empty object)
    "required": ["language"]           // optional: list of required property keys
  },
  "calculationProperties": {},         // required (can be empty object)
  "mirrorProperties": {},              // optional
  "relations": {},                     // optional
  "aggregationProperties": {}          // optional
}
```

API endpoints:
- `POST /blueprints` — create
- `PATCH /blueprints/{identifier}` — update
- `GET /blueprints/{identifier}` — read

## Properties

Properties live inside `schema.properties`. Every property requires `type`.

### Primitive types

```json
"language": { "type": "string", "title": "Language" },
"replicas": { "type": "number", "title": "Replicas" },
"is_active": { "type": "boolean", "title": "Active" }
```

### String formats

```json
"docs_url":   { "type": "string", "format": "url",       "title": "Docs URL" },
"owner_email":{ "type": "string", "format": "email",     "title": "Owner Email" },
"readme":     { "type": "string", "format": "markdown",  "title": "README" },
"config":     { "type": "string", "format": "yaml",      "title": "Config" },
"owner":      { "type": "string", "format": "user",      "title": "Owner" },
"team":       { "type": "string", "format": "team",      "title": "Team" },
"created_at": { "type": "string", "format": "date-time", "title": "Created At" },
"ttl":        { "type": "string", "format": "timer",     "title": "TTL" }
```

### Enum (dropdown)

```json
"environment": {
  "type": "string",
  "title": "Environment",
  "enum": ["production", "staging", "development"],
  "enumColors": {
    "production": "red",
    "staging": "yellow",
    "development": "green"
  }
}
```

### Array

```json
"tags": { "type": "array", "title": "Tags", "items": { "type": "string" } },
"ports": { "type": "array", "title": "Ports", "items": { "type": "number" } }
```

### Object / YAML

```json
"labels": { "type": "object", "title": "Labels" },
"helm_values": { "type": "string", "format": "yaml", "title": "Helm Values" }
```

## Relations

Relations connect blueprints. Defined outside `schema`, at the top level.

```json
"relations": {
  "team": {
    "title": "Owning Team",
    "target": "team",         // target blueprint identifier
    "required": false,        // default false
    "many": false             // default false
  },
  "dependencies": {
    "title": "Dependencies",
    "target": "microservice",
    "required": false,
    "many": true              // many: true allows multiple targets
  }
}
```

**Constraint**: `many: true` and `required: true` cannot both be set.

## Mirror Properties

Pull a property value from a related entity through a relation. Read-only.

```json
"mirrorProperties": {
  "team_slack_channel": {
    "title": "Team Slack Channel",
    "path": "team.slack_channel"   // relation_name.property_name
  },
  "team_name": {
    "title": "Team Name",
    "path": "team.$title"          // $title and $identifier are meta-properties
  }
}
```

Supports chaining through multiple relations: `"relation1.relation2.property"`.

## Calculation Properties

Compute new values from existing properties using JQ. Supports all JQ operators.

```json
"calculationProperties": {
  "full_name": {
    "title": "Full Name",
    "type": "string",
    "calculation": ".properties.first_name + \" \" + .properties.last_name"
  },
  "external_url": {
    "title": "External Link",
    "type": "string",
    "format": "url",
    "calculation": "\"https://your-tool.example.com/service/\" + .identifier"
  },
  "is_high_replica": {
    "title": "High Replica Count",
    "type": "boolean",
    "calculation": ".properties.replicas > 5"
  }
}
```

Access properties via `.properties.<name>`. Use `.identifier` for the entity identifier.
For property names with dashes: `.properties.'prop-with-dash'`.

## Aggregation Properties

Calculate metrics from related entities (counts, sums, averages, min, max, median).

Replace `<related-blueprint>` with the blueprint you want to aggregate from, and `<property>` with a numeric property on that blueprint.

```json
"aggregationProperties": {
  "open_items_count": {
    "title": "Open Items",
    "type": "number",
    "target": "<related-blueprint>",  // blueprint to aggregate from
    "calculationSpec": {
      "func": "count",
      "calculationBy": "entities"
    },
    "query": {
      "combinator": "and",
      "rules": [
        { "property": "<status-property>", "operator": "=", "value": "<open-value>" }
      ]
    }
  },
  "avg_frequency_per_week": {
    "title": "Avg Per Week",
    "type": "number",
    "target": "<related-blueprint>",
    "calculationSpec": {
      "func": "average",
      "calculationBy": "entities",
      "averageOf": "week",
      "measureTimeBy": "$createdAt"
    }
  },
  "total_points": {
    "title": "Total Points",
    "type": "number",
    "target": "<related-blueprint>",
    "calculationSpec": {
      "func": "sum",
      "calculationBy": "property",
      "property": "<numeric-property>"
    }
  }
}
```

`calculationBy: "entities"` → operate on counts. `calculationBy: "property"` → operate on a numeric property value.

## Gotchas

- `type` on a property is **immutable** — changing it requires creating a new property and migrating data.
- `calculationProperties` must be present in the blueprint JSON even if empty (`{}`).
- Mirror properties reference the **relation key name**, not the relation title.
- Aggregation results for large blueprints can take up to 24 hours to sync.
- `many: true` + `required: true` on a relation is invalid and will be rejected.
- Property names cannot start with `$` — that prefix is reserved for Port meta-properties (`$identifier`, `$title`, `$createdAt`, `$updatedAt`, `$team`).

## Further Reference

See [references/REFERENCE.md](references/REFERENCE.md) for property type quick-reference and API examples.
