---
name: port-scorecards
description: >
  Port.io scorecards expert. Use when creating, modifying, or debugging Port scorecards:
  defining levels (Basic/Bronze/Silver/Gold), writing rules with operators (=, !=, contains,
  isEmpty, isNotEmpty, <, >), applying filters, configuring scorecard JSON, understanding
  level progression logic, and triggering automations from scorecard changes. Essential when
  the user mentions Port scorecards, production readiness, quality levels, compliance checks,
  or entity scoring.
license: Apache-2.0
metadata:
  author: port-io
  version: "1.0"
compatibility: Designed for use with Port.io (port.io).
---

# Port Scorecards

Scorecards evaluate catalog entities against quality/compliance standards and assign them a level.

## Core Concepts

- **Levels** progress from lowest to highest: `Basic → Bronze → Silver → Gold` (customizable).
- An entity reaches a level only after passing **all rules at that level and all lower levels**.
- Rules use the same query operators as Port's search syntax.
- Scorecard results for large blueprints can take up to **24 hours** to sync.

## Scorecard JSON Schema

The identifiers, blueprint, and property names below are illustrative — replace them with your own.

```json
{
  "identifier": "<scorecard-identifier>",
  "title": "<Scorecard Title>",
  "blueprint": "<your-blueprint>",
  "filter": {
    "combinator": "and",
    "rules": [
      { "property": "environment", "operator": "=", "value": "production" }
    ]
  },
  "levels": [
    { "color": "paleBlue", "title": "Basic" },
    { "color": "bronze",   "title": "Bronze" },
    { "color": "silver",   "title": "Silver" },
    { "color": "gold",     "title": "Gold" }
  ],
  "rules": [
    {
      "identifier": "has_on_call",
      "title": "Has On-Call",
      "level": "Bronze",
      "query": {
        "combinator": "and",
        "rules": [
          { "property": "on_call", "operator": "isNotEmpty" }
        ]
      }
    },
    {
      "identifier": "has_readme",
      "title": "Has README",
      "level": "Bronze",
      "query": {
        "combinator": "and",
        "rules": [
          { "property": "readme_url", "operator": "isNotEmpty" }
        ]
      }
    },
    {
      "identifier": "high_test_coverage",
      "title": "Test Coverage ≥ 80%",
      "level": "Silver",
      "query": {
        "combinator": "and",
        "rules": [
          { "property": "test_coverage", "operator": ">=", "value": 80 }
        ]
      }
    },
    {
      "identifier": "no_critical_vulns",
      "title": "No Critical Vulnerabilities",
      "level": "Gold",
      "query": {
        "combinator": "and",
        "rules": [
          { "property": "critical_vuln_count", "operator": "=", "value": 0 }
        ]
      }
    }
  ]
}
```

## Available Rule Operators

| Operator | Works on | Description |
|---|---|---|
| `=` | string, number, boolean | Exact match |
| `!=` | string, number, boolean | Not equal |
| `<` | number | Less than |
| `>` | number | Greater than |
| `<=` | number | Less than or equal |
| `>=` | number | Greater than or equal |
| `contains` | string | Substring match |
| `doesNotContain` | string | Substring absence |
| `beginsWith` | string | Prefix match |
| `endsWith` | string | Suffix match |
| `isEmpty` | any | Property is null/empty |
| `isNotEmpty` | any | Property has a value |
| `in` | string, array | Value is in list |
| `notIn` | string, array | Value is not in list |
| `between` | number, date | Value within range |

## Filters

The optional `filter` block restricts which entities are evaluated by the scorecard:

```json
"filter": {
  "combinator": "or",
  "rules": [
    { "property": "tier", "operator": "=", "value": "tier-1" },
    { "property": "tier", "operator": "=", "value": "tier-2" }
  ]
}
```

You can also filter by meta-properties:

```json
{ "property": "$team", "operator": "isNotEmpty" }
```

## Multi-condition Rules

Rules can combine conditions with `and`/`or`:

```json
{
  "identifier": "sre_ready",
  "title": "SRE Ready",
  "level": "Gold",
  "query": {
    "combinator": "and",
    "rules": [
      { "property": "on_call", "operator": "isNotEmpty" },
      { "property": "pagerduty_service", "operator": "isNotEmpty" },
      { "property": "runbook_url", "operator": "isNotEmpty" }
    ]
  }
}
```

## Scorecard API

```bash
# Create scorecard
curl -X POST https://api.port.io/v1/blueprints/{blueprint_id}/scorecards \
  -H "Authorization: Bearer $PORT_TOKEN" \
  -H "Content-Type: application/json" \
  -d @scorecard.json

# List scorecards for a blueprint
curl https://api.port.io/v1/blueprints/{blueprint_id}/scorecards \
  -H "Authorization: Bearer $PORT_TOKEN"

# Get scorecard results for an entity
curl "https://api.port.io/v1/entities/{entity_id}/scorecards" \
  -H "Authorization: Bearer $PORT_TOKEN"
```

## Triggering Automations from Scorecard Changes

Use the `ENTITY_UPDATED` automation trigger with a JQ condition on the scorecard level.
Replace `<your-blueprint>`, `<scorecard-id>`, and `<level-title>` with your own values.

```json
{
  "trigger": {
    "type": "automation",
    "event": {
      "type": "ENTITY_UPDATED",
      "blueprintIdentifier": "<your-blueprint>"
    },
    "condition": {
      "type": "JQ",
      "expressions": [
        ".diff.after.scorecards.<scorecard-id>.level == \"<level-title>\""
      ],
      "combinator": "and"
    }
  }
}
```

## Gotchas

- Rules only compare properties to **static values** — no property-to-property comparisons.
- All rules at a level are equally weighted; there is no weighting mechanism.
- An entity cannot skip levels — it must pass all lower-level rules to advance.
- The `filter` applies before evaluation; filtered-out entities show no scorecard result.
- Max **5 million rule result entities** supported per scorecard.
