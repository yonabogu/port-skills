# Port Software Catalog — Reference

## Search Presets for Date Ranges

```json
{ "property": "$createdAt", "operator": "between", "value": { "preset": "today" } }
{ "property": "$createdAt", "operator": "between", "value": { "preset": "yesterday" } }
{ "property": "$updatedAt", "operator": "between", "value": { "preset": "last7Days" } }
{ "property": "$updatedAt", "operator": "between", "value": { "preset": "last14Days" } }
{ "property": "$updatedAt", "operator": "between", "value": { "preset": "last30Days" } }
{ "property": "$updatedAt", "operator": "between", "value": { "preset": "last3Months" } }
{ "property": "$updatedAt", "operator": "between", "value": { "preset": "last6Months" } }
{ "property": "$updatedAt", "operator": "between", "value": { "preset": "last12Months" } }
```

## Ingesting from CI/CD (GitHub Actions example)

```yaml
- name: Report deployment to Port
  run: |
    curl -X POST "https://api.port.io/v1/blueprints/deployment/entities?upsert=true" \
      -H "Authorization: Bearer ${{ secrets.PORT_TOKEN }}" \
      -H "Content-Type: application/json" \
      -d '{
        "identifier": "${{ github.sha }}",
        "title": "Deploy ${{ github.ref_name }}",
        "properties": {
          "sha": "${{ github.sha }}",
          "branch": "${{ github.ref_name }}",
          "status": "running"
        },
        "relations": {
          "service": "${{ inputs.service_name }}"
        }
      }'
```

## Pagination for Large Blueprints

```bash
# Page through entities (max 50 per page by default)
curl "https://api.port.io/v1/blueprints/microservice/entities?limit=50&page=1" \
  -H "Authorization: Bearer $PORT_TOKEN"
```
