# Port Self-Service Actions — Reference

## Reporting Action Status from Backend

Your backend should call Port's API to update the action run status:

```bash
# Mark as success
curl -X PATCH https://api.port.io/v1/actions/runs/$PORT_RUN_ID \
  -H "Authorization: Bearer $PORT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"status": "SUCCESS", "message": {"run_status": "Scaffold completed"}}'

# Mark as failure
curl -X PATCH https://api.port.io/v1/actions/runs/$PORT_RUN_ID \
  -H "Authorization: Bearer $PORT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"status": "FAILURE", "message": {"run_status": "Scaffold failed", "error": "..."}}'

# Add log line
curl -X POST https://api.port.io/v1/actions/runs/$PORT_RUN_ID/logs \
  -H "Authorization: Bearer $PORT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"message": "Repo created: https://github.com/my-org/my-service"}'
```

## GitHub Actions Workflow Template

Minimal workflow for a Port action:

```yaml
name: Scaffold Service
on:
  workflow_dispatch:
    inputs:
      service_name:
        required: true
        type: string
      port_run_id:
        required: true
        type: string

jobs:
  scaffold:
    runs-on: ubuntu-latest
    steps:
      - name: Create repo
        run: |
          gh repo create my-org/${{ inputs.service_name }} --private
        env:
          GH_TOKEN: ${{ secrets.GH_TOKEN }}

      - name: Report success to Port
        if: success()
        run: |
          curl -X PATCH https://api.port.io/v1/actions/runs/${{ inputs.port_run_id }} \
            -H "Authorization: Bearer ${{ secrets.PORT_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d '{"status":"SUCCESS","message":{"run_status":"Done"}}'

      - name: Report failure to Port
        if: failure()
        run: |
          curl -X PATCH https://api.port.io/v1/actions/runs/${{ inputs.port_run_id }} \
            -H "Authorization: Bearer ${{ secrets.PORT_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d '{"status":"FAILURE","message":{"run_status":"Failed"}}'
```

## Multi-Step Form Example

```json
"userInputs": {
  "steps": [
    {
      "title": "Basic Details",
      "order": ["service_name", "language"]
    },
    {
      "title": "Infrastructure",
      "order": ["team", "environment", "replicas"]
    }
  ],
  "properties": {
    "service_name": { "type": "string", "title": "Name" },
    "language": { "type": "string", "enum": ["python", "go"] },
    "team": { "type": "string", "format": "team", "title": "Team" },
    "environment": { "type": "string", "enum": ["dev", "staging", "prod"] },
    "replicas": { "type": "number", "title": "Replicas", "default": 2 }
  },
  "required": ["service_name", "language"]
}
```
