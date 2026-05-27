---
name: port-jq-patterns
description: >
  Port.io JQ transformation expert. Use when writing or debugging JQ expressions for Port:
  mapping configurations (itemsToParse, query filters, property extraction), automation
  conditions (.diff.before/.diff.after), calculation properties (.properties.<name>, .identifier),
  aggregation property queries, and self-service action input transforms. Covers Port-specific JQ
  gotchas: UTC runtime, .properties accessor, special-character property names, and the
  itemsToParseTopLevelTransform pattern. Essential when the user is stuck on a JQ expression in any
  Port context.
license: Apache-2.0
metadata:
  author: port-io
  version: "1.0"
compatibility: Designed for use with Port.io (port.io).
---

# Port JQ Patterns

Port uses JQ as its expression language across four contexts:
1. **Integration mapping** — transform API responses into Port entities
2. **Calculation properties** — compute derived properties from existing ones
3. **Automation conditions** — filter which events trigger an automation
4. **Aggregation property queries** — filter related entities before aggregating

## Context 1: Integration Mapping

Mapping files use JQ to extract properties from API payloads.

### Basic property extraction

```yaml
resources:
  - kind: repository
    selector:
      query: "true"              # JQ: include all items
    port:
      entity:
        mappings:
          identifier: .name
          title: .name
          blueprint: '"microservice"'
          properties:
            url: .html_url
            language: .language
            is_private: .private
            created_at: .created_at
          relations:
            team: .owner.login
```

### itemsToParse — expand array into multiple entities

```yaml
- kind: repository
  selector:
    query: '.topics | length > 0'   # only repos with topics
  port:
    itemsToParse: .topics
    entity:
      mappings:
        identifier: .item            # .item = current element from the array
        title: .item
        blueprint: '"topic"'
        properties:
          repo: .name               # parent-level fields still accessible
```

`itemsToParseTopLevelTransform: false` keeps the original array in the payload (default is `true` which removes it).

### Conditional filtering

```yaml
selector:
  query: '.archived == false and .language != null'
```

### Nested field access

```yaml
language: .primaryLanguage.name      # null-safe: returns null if primaryLanguage is null
owner: .owner.login
```

### String concatenation

```yaml
identifier: '.owner.login + "/" + .name'
```

### Transforming enums

```yaml
status: 'if .state == "open" then "active" else "inactive" end'
```

### Date normalization

```yaml
# Convert epoch to ISO 8601
created_at: '.created_at | todate'

# Port JQ runs in UTC — todate, now, strftime all use UTC
```

## Context 2: Calculation Properties

Access entity data through `.properties.<name>` and `.identifier`.

```json
// Concatenate string properties
"calculation": ".properties.first_name + \" \" + .properties.last_name"

// Build a URL
"calculation": "\"https://grafana.example.com/d/\" + .identifier"

// Conditional
"calculation": "if .properties.replicas > 5 then \"high\" else \"normal\" end"

// Arithmetic
"calculation": ".properties.passed_tests / .properties.total_tests * 100"

// Array length
"calculation": ".properties.tags | length"

// Check if array contains value
"calculation": ".properties.environments | contains([\"production\"])"
```

Property names with special characters (dashes, dots):

```json
"calculation": ".properties.'my-prop-with-dash'"
```

Available meta-properties in calculations:

```json
".identifier"         // entity identifier
".title"              // entity title
".createdAt"          // ISO datetime string
".updatedAt"          // ISO datetime string
".team"               // array of team names
```

## Context 3: Automation Conditions

JQ expressions evaluate the diff payload:

```json
// Property equals specific value
".diff.after.properties.status == \"failed\""

// Property changed
".diff.before.properties.tier != .diff.after.properties.tier"

// Relation newly set
".diff.before.relations.team == null and .diff.after.relations.team != null"

// Scorecard level reached
".diff.after.scorecards.ProductionReadiness.level == \"Gold\""

// Scorecard level degraded from Gold
".diff.before.scorecards.ProductionReadiness.level == \"Gold\" and
 .diff.after.scorecards.ProductionReadiness.level != \"Gold\""

// Array contains value
".diff.after.properties.environments | contains([\"production\"])"

// Numeric threshold crossed
".diff.after.properties.critical_vulns > 0 and .diff.before.properties.critical_vulns == 0"
```

## Context 4: Aggregation Property Queries

Filter related entities before aggregating. Same operators as search rules but written in the `query` object:

```json
"query": {
  "combinator": "and",
  "rules": [
    { "property": "status", "operator": "=",          "value": "open" },
    { "property": "priority", "operator": "in",       "value": ["high", "critical"] },
    { "property": "$createdAt", "operator": "between", "value": { "preset": "last30Days" } }
  ]
}
```

## Common JQ Patterns

```jq
# Null coalescing
.language // "unknown"

# Safely access nested field
(.owner // {}).login // null

# Convert boolean to string
if .is_archived then "archived" else "active" end

# Join array to string
.tags | join(", ")

# Filter array
.items | map(select(.status == "active"))

# Count matching elements
[.items[] | select(.critical == true)] | length

# Extract specific keys from object
.metadata | {name, namespace, labels}

# Map array to new array
.items | map({id: .id, name: .name})

# First element
.items[0]

# Last element
.items[-1]

# Slice array
.items[0:5]
```

## Gotchas

- Port's JQ runtime runs in **UTC**. `now | todate` returns UTC time.
- Property names with hyphens or dots in calculation properties need single quotes: `.properties.'my-prop'`.
- In **mapping** context, use `.` to refer to the current API response item. In **calculation** context, use `.properties.<name>` to access blueprint properties.
- `itemsToParse` makes `.item` the loop variable; the parent object fields remain accessible at the top level.
- `selector.query` must evaluate to a **boolean** — `"true"` (string) or a boolean JQ expression. A non-boolean truthy value may cause unexpected behavior.
- String literals in JQ require double quotes inside the expression: `'"microservice"'` (outer single quotes are YAML, inner double quotes are JQ).
- In automation conditions, `null` comparisons need `== null` not `isEmpty` (that's for search rules, not JQ).
