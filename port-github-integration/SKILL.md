---
name: port-github-integration
description: >
  Port.io GitHub integration expert. Use when configuring Port's GitHub exporter/integration:
  setting up the Ocean GitHub integration, writing mapping files for repositories, pull requests,
  issues, workflows, teams, and environments; using workflow_dispatch to trigger Port actions from
  GitHub Actions; and ingesting GitHub data into the Port catalog. Essential when the user mentions
  Port GitHub integration, GitHub exporter, syncing repos to Port, or triggering Port actions from
  GitHub workflows.
license: Apache-2.0
metadata:
  author: port-io
  version: "1.0"
compatibility: Designed for use with Port.io (port.io) and GitHub.
---

# Port GitHub Integration

Port's GitHub integration (Ocean-based) continuously syncs GitHub data into your catalog.

## Supported Resource Kinds

| Kind | What it maps |
|---|---|
| `repository` | GitHub repos → entities |
| `pull-request` | PRs → entities |
| `issue` | Issues → entities |
| `workflow` | GitHub Actions workflows |
| `workflow-run` | Workflow execution runs |
| `team` | GitHub org teams |
| `environment` | Repo environments |
| `deployment` | GitHub deployments |
| `tag` | Git tags |
| `release` | GitHub releases |

## Integration Config (`integration.yml` / `config.yml`)

```yaml
resources:
  - kind: repository
    selector:
      query: '.archived == false'
    port:
      entity:
        mappings:
          identifier: .name
          title: .name
          blueprint: '"microservice"'
          properties:
            url:         .html_url
            description: .description
            language:    .language
            default_branch: .default_branch
            open_issues: .open_issues_count
            stars:       .stargazers_count
            is_private:  .private
            topics:      .topics
          relations:
            team: .owner.login

  - kind: pull-request
    selector:
      query: '.state == "open"'
    port:
      entity:
        mappings:
          identifier: '.base.repo.name + "-" + (.number | tostring)'
          title: .title
          blueprint: '"pullRequest"'
          properties:
            url:        .html_url
            status:     .state
            author:     .user.login
            created_at: .created_at
            updated_at: .updated_at
            merged_at:  .merged_at
            base_branch:.base.ref
            head_branch:.head.ref
          relations:
            service: .base.repo.name
```

## Triggering Port Actions from GitHub

When a Port action runs, the GitHub Actions workflow receives:
- All user inputs as `workflow_dispatch` inputs
- `port_run_id` — use this to report status back to Port

```yaml
# .github/workflows/scaffold-service.yml
name: Scaffold Service
on:
  workflow_dispatch:
    inputs:
      service_name:
        required: true
        type: string
      language:
        required: true
        type: string
      port_run_id:
        required: true
        type: string

jobs:
  scaffold:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Log start to Port
        run: |
          curl -s -X POST "https://api.port.io/v1/actions/runs/${{ inputs.port_run_id }}/logs" \
            -H "Authorization: Bearer ${{ secrets.PORT_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d '{"message": "Starting scaffold for ${{ inputs.service_name }}"}'

      - name: Do work
        run: |
          # your scaffolding logic here
          echo "Creating ${{ inputs.service_name }}"

      - name: Report success
        if: success()
        run: |
          curl -s -X PATCH "https://api.port.io/v1/actions/runs/${{ inputs.port_run_id }}" \
            -H "Authorization: Bearer ${{ secrets.PORT_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d '{"status":"SUCCESS","message":{"run_status":"Done"}}'

      - name: Report failure
        if: failure()
        run: |
          curl -s -X PATCH "https://api.port.io/v1/actions/runs/${{ inputs.port_run_id }}" \
            -H "Authorization: Bearer ${{ secrets.PORT_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d '{"status":"FAILURE","message":{"run_status":"Workflow failed"}}'
```

## Port Action Backend Config (GitHub)

```json
"invocationMethod": {
  "type": "GITHUB",
  "org": "my-org",
  "repo": "port-actions",
  "workflow": "scaffold-service.yml",
  "workflowInputs": {
    "service_name": "{{ .inputs.service_name }}",
    "language":     "{{ .inputs.language }}",
    "port_run_id":  "{{ .run.id }}"
  },
  "reportWorkflowStatus": true
}
```

## Ingesting Deployment Status from CI

```yaml
- name: Upsert deployment to Port
  run: |
    curl -s -X POST "https://api.port.io/v1/blueprints/deployment/entities?upsert=true" \
      -H "Authorization: Bearer ${{ secrets.PORT_TOKEN }}" \
      -H "Content-Type: application/json" \
      -d '{
        "identifier": "${{ github.sha }}",
        "title": "${{ github.ref_name }} @ ${{ github.sha }}",
        "properties": {
          "sha":        "${{ github.sha }}",
          "branch":     "${{ github.ref_name }}",
          "status":     "SUCCESS",
          "run_url":    "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
        },
        "relations": {
          "service": "${{ github.event.repository.name }}"
        }
      }'
```

## Gotchas

- The GitHub workflow file must exist on the **default branch** even when triggering via a `ref` parameter.
- GitHub Actions `workflow_dispatch` supports **max 10 inputs** — Port truncates extra ones.
- `reportWorkflowStatus: true` (default) means Port marks the action run SUCCESS/FAILURE based on the workflow conclusion. Set to `false` if your workflow reports status manually.
- The Ocean GitHub integration runs as a polling integration — it doesn't webhook every push in real time. For near-real-time deployments, push directly via the API from CI.
