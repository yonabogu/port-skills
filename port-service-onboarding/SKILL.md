---
name: port-service-onboarding
description: >
  Port.io service onboarding recipe. Use when building an end-to-end service onboarding workflow
  in Port: creating the service blueprint, ingesting existing services, setting up a production
  readiness scorecard, and adding a self-service action to scaffold new services. This recipe
  walks through the complete: blueprint → ingest → scorecard → action pattern. Essential when the
  user wants to implement developer self-service for service creation or onboard a services catalog
  from scratch.
license: Apache-2.0
metadata:
  author: port-io
  version: "1.0"
compatibility: Designed for use with Port.io (port.io). Requires Port API access and GitHub for action backend.
---

# Port Service Onboarding Recipe

End-to-end guide: build a services catalog with production readiness scoring and a scaffold action.

## Step 1: Create the Service Blueprint

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
        "language":         { "type": "string", "title": "Language", "enum": ["go","python","typescript","java","rust"] },
        "tier":             { "type": "string", "title": "Tier", "enum": ["tier-1","tier-2","tier-3"],
                              "enumColors": { "tier-1": "red", "tier-2": "yellow", "tier-3": "green" } },
        "on_call":          { "type": "string", "format": "user",  "title": "On-Call" },
        "readme_url":       { "type": "string", "format": "url",   "title": "README" },
        "runbook_url":      { "type": "string", "format": "url",   "title": "Runbook" },
        "pagerduty_svc":    { "type": "string", "format": "url",   "title": "PagerDuty" },
        "test_coverage":    { "type": "number",  "title": "Test Coverage (%)" },
        "critical_vulns":   { "type": "number",  "title": "Critical Vulnerabilities" },
        "repo_url":         { "type": "string", "format": "url",   "title": "Repository" }
      },
      "required": ["language"]
    },
    "calculationProperties": {
      "grafana_url": {
        "type": "string", "format": "url", "title": "Grafana",
        "calculation": "\"https://grafana.example.com/d/services?var-service=\" + .identifier"
      }
    },
    "relations": {
      "team": { "title": "Owning Team", "target": "team", "required": false, "many": false }
    }
  }'
```

## Step 2: Ingest Existing Services

Option A — from GitHub (via mapping config):

```yaml
resources:
  - kind: repository
    selector:
      query: '.archived == false and (.topics | contains(["service"]))'
    port:
      entity:
        mappings:
          identifier: .name
          title: .name
          blueprint: '"microservice"'
          properties:
            language:   .language
            repo_url:   .html_url
          relations:
            team: .owner.login
```

Option B — one-time bulk import via API:

```bash
curl -X POST "https://api.port.io/v1/blueprints/microservice/entities/bulk" \
  -H "Authorization: Bearer $PORT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "entities": [
      { "identifier": "payment-service", "title": "Payment Service",
        "properties": { "language": "go", "tier": "tier-1" } },
      { "identifier": "auth-service", "title": "Auth Service",
        "properties": { "language": "python", "tier": "tier-1" } }
    ]
  }'
```

## Step 3: Add a Production Readiness Scorecard

```bash
curl -X POST "https://api.port.io/v1/blueprints/microservice/scorecards" \
  -H "Authorization: Bearer $PORT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "identifier": "ProductionReadiness",
    "title": "Production Readiness",
    "levels": [
      { "color": "paleBlue", "title": "Basic"  },
      { "color": "bronze",   "title": "Bronze" },
      { "color": "silver",   "title": "Silver" },
      { "color": "gold",     "title": "Gold"   }
    ],
    "rules": [
      { "identifier": "has_on_call",   "title": "Has On-Call",   "level": "Bronze",
        "query": { "combinator": "and", "rules": [{ "property": "on_call",    "operator": "isNotEmpty" }] } },
      { "identifier": "has_readme",    "title": "Has README",    "level": "Bronze",
        "query": { "combinator": "and", "rules": [{ "property": "readme_url", "operator": "isNotEmpty" }] } },
      { "identifier": "has_runbook",   "title": "Has Runbook",   "level": "Silver",
        "query": { "combinator": "and", "rules": [{ "property": "runbook_url","operator": "isNotEmpty" }] } },
      { "identifier": "coverage_80",   "title": "Test Coverage ≥ 80%", "level": "Silver",
        "query": { "combinator": "and", "rules": [{ "property": "test_coverage","operator": ">=","value": 80 }] } },
      { "identifier": "no_criticals",  "title": "No Critical CVEs", "level": "Gold",
        "query": { "combinator": "and", "rules": [{ "property": "critical_vulns","operator": "=","value": 0 }] } },
      { "identifier": "has_pagerduty", "title": "PagerDuty Linked","level": "Gold",
        "query": { "combinator": "and", "rules": [{ "property": "pagerduty_svc","operator": "isNotEmpty" }] } }
    ]
  }'
```

## Step 4: Create a Scaffold New Service Action

```bash
curl -X POST "https://api.port.io/v1/blueprints/microservice/actions" \
  -H "Authorization: Bearer $PORT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "identifier": "scaffold_service",
    "title": "Scaffold New Service",
    "icon": "Git",
    "description": "Creates a new service from a Cookiecutter template",
    "trigger": {
      "type": "self-service",
      "operation": "CREATE",
      "blueprintIdentifier": "microservice",
      "userInputs": {
        "properties": {
          "service_name": { "type": "string", "title": "Service Name" },
          "language": { "type": "string", "title": "Language",
                        "enum": ["go","python","typescript"] },
          "team": { "type": "string", "format": "team", "title": "Owning Team" }
        },
        "required": ["service_name", "language"]
      }
    },
    "invocationMethod": {
      "type": "GITHUB",
      "org": "my-org",
      "repo": "port-actions",
      "workflow": "scaffold-service.yml",
      "workflowInputs": {
        "service_name": "{{ .inputs.service_name }}",
        "language":     "{{ .inputs.language }}",
        "team":         "{{ .inputs.team }}",
        "port_run_id":  "{{ .run.id }}"
      },
      "reportWorkflowStatus": true
    },
    "requiredApproval": false
  }'
```

## Step 5: Auto-create the Catalog Entity after Scaffold

In your GitHub Actions scaffold workflow, after the repo is created, call Port to register the new entity:

```bash
curl -X POST "https://api.port.io/v1/blueprints/microservice/entities?upsert=true" \
  -H "Authorization: Bearer $PORT_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"identifier\": \"$SERVICE_NAME\",
    \"title\": \"$SERVICE_NAME\",
    \"properties\": {
      \"language\": \"$LANGUAGE\",
      \"repo_url\": \"https://github.com/my-org/$SERVICE_NAME\"
    },
    \"relations\": {
      \"team\": \"$TEAM\"
    }
  }"
```

## Checklist

- [ ] Blueprint created with key properties
- [ ] Existing services ingested (API or integration)
- [ ] Scorecard defined with Bronze/Silver/Gold rules
- [ ] Scaffold action created with GitHub backend
- [ ] GitHub workflow reports success/failure to Port via run ID
- [ ] New entity auto-created in Port after scaffold
