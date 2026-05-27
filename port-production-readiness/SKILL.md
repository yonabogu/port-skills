---
name: port-production-readiness
description: >
  Port.io production readiness scorecard recipe. Use when designing or implementing a production
  readiness framework in Port: choosing the right scorecard levels and rules, mapping quality
  dimensions (reliability, security, observability, ownership, documentation) to Bronze/Silver/Gold
  levels, writing the scorecard JSON, and setting up automations to alert on scorecard degradation.
  Essential when the user wants to build a production readiness scorecard, engineer maturity model,
  or service quality dashboard in Port.
license: Apache-2.0
metadata:
  author: port-io
  version: "1.0"
compatibility: Designed for use with Port.io (port.io).
---

# Port Production Readiness Scorecard

A production readiness scorecard grades services across ownership, observability, reliability, security, and documentation dimensions.

## Recommended Level Structure

| Level | Theme | What it means |
|---|---|---|
| Basic | Exists | The service is in the catalog |
| Bronze | Owned | Has an owner and basic documentation |
| Silver | Observable | Monitored, tested, has runbook |
| Gold | Reliable & Secure | Fully production-grade |

## Full Scorecard JSON

```json
{
  "identifier": "ProductionReadiness",
  "title": "Production Readiness",
  "blueprint": "microservice",
  "levels": [
    { "color": "paleBlue", "title": "Basic"  },
    { "color": "bronze",   "title": "Bronze" },
    { "color": "silver",   "title": "Silver" },
    { "color": "gold",     "title": "Gold"   }
  ],
  "rules": [
    {
      "identifier": "has_on_call",
      "title": "Has On-Call Owner",
      "level": "Bronze",
      "query": { "combinator": "and",
        "rules": [{ "property": "on_call", "operator": "isNotEmpty" }] }
    },
    {
      "identifier": "has_readme",
      "title": "Has README / Docs URL",
      "level": "Bronze",
      "query": { "combinator": "and",
        "rules": [{ "property": "readme_url", "operator": "isNotEmpty" }] }
    },
    {
      "identifier": "has_team",
      "title": "Has Owning Team",
      "level": "Bronze",
      "query": { "combinator": "and",
        "rules": [{ "property": "$team", "operator": "isNotEmpty" }] }
    },
    {
      "identifier": "has_runbook",
      "title": "Has Runbook",
      "level": "Silver",
      "query": { "combinator": "and",
        "rules": [{ "property": "runbook_url", "operator": "isNotEmpty" }] }
    },
    {
      "identifier": "has_pagerduty",
      "title": "PagerDuty Service Configured",
      "level": "Silver",
      "query": { "combinator": "and",
        "rules": [{ "property": "pagerduty_svc", "operator": "isNotEmpty" }] }
    },
    {
      "identifier": "coverage_80",
      "title": "Test Coverage ≥ 80%",
      "level": "Silver",
      "query": { "combinator": "and",
        "rules": [{ "property": "test_coverage", "operator": ">=", "value": 80 }] }
    },
    {
      "identifier": "no_critical_vulns",
      "title": "No Critical CVEs",
      "level": "Gold",
      "query": { "combinator": "and",
        "rules": [{ "property": "critical_vulns", "operator": "=", "value": 0 }] }
    },
    {
      "identifier": "has_slo",
      "title": "SLO Defined",
      "level": "Gold",
      "query": { "combinator": "and",
        "rules": [{ "property": "slo_url", "operator": "isNotEmpty" }] }
    },
    {
      "identifier": "deploys_regularly",
      "title": "Deploys Regularly (≥ 1/week)",
      "level": "Gold",
      "query": { "combinator": "and",
        "rules": [{ "property": "avg_deploys_per_week", "operator": ">=", "value": 1 }] }
    }
  ]
}
```

## Required Blueprint Properties

To use the full scorecard above, your `microservice` blueprint needs:

```json
{
  "on_call":             { "type": "string", "format": "user",  "title": "On-Call" },
  "readme_url":          { "type": "string", "format": "url",   "title": "README" },
  "runbook_url":         { "type": "string", "format": "url",   "title": "Runbook" },
  "pagerduty_svc":       { "type": "string", "format": "url",   "title": "PagerDuty" },
  "slo_url":             { "type": "string", "format": "url",   "title": "SLO" },
  "test_coverage":       { "type": "number", "title": "Test Coverage (%)" },
  "critical_vulns":      { "type": "number", "title": "Critical Vulnerabilities" },
  "avg_deploys_per_week":{ "type": "number", "title": "Avg Deploys/Week" }
}
```

`avg_deploys_per_week` is best modeled as an aggregation property linked to a `deployment` blueprint.

## Automation: Alert When Service Degrades from Gold

```json
{
  "identifier": "gold_degraded",
  "title": "Alert: Service Left Gold",
  "trigger": {
    "type": "automation",
    "event": {
      "type": "ENTITY_UPDATED",
      "blueprintIdentifier": "microservice"
    },
    "condition": {
      "type": "JQ",
      "expressions": [
        ".diff.before.scorecards.ProductionReadiness.level == \"Gold\"",
        ".diff.after.scorecards.ProductionReadiness.level != \"Gold\""
      ],
      "combinator": "and"
    }
  },
  "invocationMethod": {
    "type": "WEBHOOK",
    "url": "https://hooks.slack.com/services/YOUR/WEBHOOK/URL",
    "body": {
      "text": "⚠️ {{ .event.context.entityIdentifier }} dropped from Gold production readiness"
    }
  },
  "publish": true
}
```

## Populating Scorecard Data via CI

Add these steps to your CI pipeline to keep scorecard properties current:

```bash
# Update test coverage after test run
curl -X PATCH "https://api.port.io/v1/blueprints/microservice/entities/$SERVICE_NAME?merge=true" \
  -H "Authorization: Bearer $PORT_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"properties\": {\"test_coverage\": $COVERAGE_PERCENT}}"

# Update CVE count after security scan
curl -X PATCH "https://api.port.io/v1/blueprints/microservice/entities/$SERVICE_NAME?merge=true" \
  -H "Authorization: Bearer $PORT_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"properties\": {\"critical_vulns\": $CRITICAL_COUNT}}"
```

## Gotchas

- Scorecard rules evaluate **static values only** — no property-to-property comparisons.
- An entity must pass all lower-level rules to reach a higher level — it cannot skip levels.
- Use aggregation properties (e.g., `avg_deploys_per_week`) to derive metrics from related entities rather than maintaining them manually.
- For large blueprints (millions of entities), scorecard results may take up to 24 hours to sync after rule changes.
