# Port Production Readiness — Reference

## Aggregation Property: Average Deploys Per Week

Add to the `microservice` blueprint to automatically calculate deployment frequency:

```json
"aggregationProperties": {
  "avg_deploys_per_week": {
    "title": "Avg Deploys / Week",
    "type": "number",
    "target": "deployment",
    "calculationSpec": {
      "func": "average",
      "calculationBy": "entities",
      "averageOf": "week",
      "measureTimeBy": "$createdAt"
    }
  }
}
```

Requires a `deployment` blueprint with a relation back to `microservice`.

## Production Readiness Dashboard Filters

To show only Tier-1 services in the scorecard view, use the scorecard `filter`:

```json
"filter": {
  "combinator": "and",
  "rules": [
    { "property": "tier", "operator": "=", "value": "tier-1" }
  ]
}
```

## Scoring Summary (for reporting)

Use the Port search API to get a count by level:

```bash
# Count services at each level
curl -X POST "https://api.port.io/v1/entities/search" \
  -H "Authorization: Bearer $PORT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "rules": [
      { "property": "$blueprint", "operator": "=", "value": "microservice" }
    ],
    "combinator": "and",
    "include": ["scorecards"]
  }'
```
