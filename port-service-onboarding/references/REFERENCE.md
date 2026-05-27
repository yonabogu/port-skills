# Port Service Onboarding — Reference

## GitHub Scaffold Workflow Template

```yaml
name: Scaffold Service
on:
  workflow_dispatch:
    inputs:
      service_name: { required: true, type: string }
      language:     { required: true, type: string }
      team:         { required: false, type: string }
      port_run_id:  { required: true, type: string }

env:
  PORT_TOKEN: ${{ secrets.PORT_TOKEN }}

jobs:
  scaffold:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Log start
        run: |
          curl -s -X POST "https://api.port.io/v1/actions/runs/${{ inputs.port_run_id }}/logs" \
            -H "Authorization: Bearer $PORT_TOKEN" \
            -H "Content-Type: application/json" \
            -d "{\"message\": \"Scaffolding ${{ inputs.service_name }}...\"}"

      - name: Create GitHub repo
        run: |
          gh repo create my-org/${{ inputs.service_name }} \
            --private --description "${{ inputs.service_name }} service"
        env:
          GH_TOKEN: ${{ secrets.GH_TOKEN }}

      - name: Register entity in Port
        run: |
          curl -s -X POST "https://api.port.io/v1/blueprints/microservice/entities?upsert=true" \
            -H "Authorization: Bearer $PORT_TOKEN" \
            -H "Content-Type: application/json" \
            -d "{
              \"identifier\": \"${{ inputs.service_name }}\",
              \"title\": \"${{ inputs.service_name }}\",
              \"properties\": {
                \"language\": \"${{ inputs.language }}\",
                \"repo_url\": \"https://github.com/my-org/${{ inputs.service_name }}\"
              },
              \"relations\": { \"team\": \"${{ inputs.team }}\" }
            }"

      - name: Report success
        if: success()
        run: |
          curl -s -X PATCH "https://api.port.io/v1/actions/runs/${{ inputs.port_run_id }}" \
            -H "Authorization: Bearer $PORT_TOKEN" \
            -H "Content-Type: application/json" \
            -d "{\"status\":\"SUCCESS\",\"message\":{\"run_status\":\"Done\"},
                 \"link\":[\"https://github.com/my-org/${{ inputs.service_name }}\"]}"

      - name: Report failure
        if: failure()
        run: |
          curl -s -X PATCH "https://api.port.io/v1/actions/runs/${{ inputs.port_run_id }}" \
            -H "Authorization: Bearer $PORT_TOKEN" \
            -H "Content-Type: application/json" \
            -d '{"status":"FAILURE","message":{"run_status":"Workflow failed"}}'
```
