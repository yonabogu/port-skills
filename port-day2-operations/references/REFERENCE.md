# Port Day-2 Operations — Reference

## GitHub Actions Workflow: Scale Deployment

```yaml
name: Scale Deployment
on:
  workflow_dispatch:
    inputs:
      workload_id:  { required: true, type: string }
      namespace:    { required: true, type: string }
      cluster:      { required: true, type: string }
      replicas:     { required: true, type: string }
      port_run_id:  { required: true, type: string }

jobs:
  scale:
    runs-on: ubuntu-latest
    steps:
      - name: Configure kubectl
        uses: azure/k8s-set-context@v3
        with:
          method: kubeconfig
          kubeconfig: ${{ secrets.KUBECONFIG }}

      - name: Scale deployment
        run: |
          kubectl scale deployment/${{ inputs.workload_id }} \
            --replicas=${{ inputs.replicas }} \
            -n ${{ inputs.namespace }}

      - name: Update Port entity
        run: |
          curl -s -X PATCH \
            "https://api.port.io/v1/blueprints/workload/entities/${{ inputs.workload_id }}?merge=true" \
            -H "Authorization: Bearer ${{ secrets.PORT_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d "{\"properties\": {\"replicas\": ${{ inputs.replicas }}}}"

      - name: Report success
        if: success()
        run: |
          curl -s -X PATCH "https://api.port.io/v1/actions/runs/${{ inputs.port_run_id }}" \
            -H "Authorization: Bearer ${{ secrets.PORT_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d "{\"status\":\"SUCCESS\",\"message\":{\"run_status\":\"Scaled to ${{ inputs.replicas }} replicas\"}}"

      - name: Report failure
        if: failure()
        run: |
          curl -s -X PATCH "https://api.port.io/v1/actions/runs/${{ inputs.port_run_id }}" \
            -H "Authorization: Bearer ${{ secrets.PORT_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d '{"status":"FAILURE","message":{"run_status":"Scale failed"}}'
```

## Availability Condition: Limit Action to Production

```json
"condition": {
  "type": "SEARCH",
  "rules": [
    { "property": "environment", "operator": "=", "value": "production" }
  ],
  "combinator": "and"
}
```

## Common Day-2 Action Templates

| Action | Backend | Notes |
|---|---|---|
| Scale | GitHub/kubectl | Pass namespace + cluster from entity |
| Restart | GitHub/kubectl | `kubectl rollout restart` |
| Rollback | GitHub | Require approval, use `target_version` input |
| Update config | Webhook | Use secrets for auth token |
| Trigger deploy | GitHub | Dispatch a deploy workflow |
| Toggle feature flag | Webhook | POST to feature flag service |
