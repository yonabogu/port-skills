# Port Scorecards — Reference

## Level Colors

Standard palette values for `color` in levels:

`red`, `orange`, `yellow`, `green`, `blue`, `purple`, `paleBlue`, `bronze`, `silver`, `gold`, `grey`

## Checking Scorecard Results via API

```bash
# Entities at a specific level
curl "https://api.port.io/v1/blueprints/microservice/entities?scorecards.ProductionReadiness.level=Gold" \
  -H "Authorization: Bearer $PORT_TOKEN"
```

## Production Readiness Scorecard Template

Common rules for a service production readiness scorecard:

| Level | Rule | Property |
|---|---|---|
| Bronze | Has on-call owner | `on_call` isNotEmpty |
| Bronze | Has README/docs | `readme_url` isNotEmpty |
| Bronze | Has runbook | `runbook_url` isNotEmpty |
| Silver | PagerDuty service linked | `pagerduty_service` isNotEmpty |
| Silver | Test coverage ≥ 80% | `test_coverage` >= 80 |
| Silver | Has CI pipeline | `ci_pipeline_url` isNotEmpty |
| Gold | No critical CVEs | `critical_vuln_count` = 0 |
| Gold | SLO defined | `slo_url` isNotEmpty |
| Gold | Deployment frequency tracked | `avg_deploys_per_week` > 0 |
