# port-io/skills

Agent Skills for [Port.io](https://port.io) — the Internal Developer Portal platform.

These skills give AI agents structured expertise on Port's core concepts so they stop hallucinating field names, JSON schemas, and JQ expressions.

## Install

```bash
npx skills install github.com/port-io/skills
```

Or install individual skills:

```bash
npx skills install github.com/port-io/skills/port-blueprints
```

## Skills

### Tier 1 — Core

| Skill | Description |
|---|---|
| [port-blueprints](./port-blueprints/) | Blueprint schema, properties, relations, mirror/aggregation/calculation properties |
| [port-self-service-actions](./port-self-service-actions/) | Action backends, input forms, RBAC, approval flows |
| [port-scorecards](./port-scorecards/) | Rules, levels (Basic→Gold), automation triggers |
| [port-automations](./port-automations/) | Triggers, JQ conditions, workflow patterns |
| [port-software-catalog](./port-software-catalog/) | Entity ingestion, auto-discovery, API CRUD |
| [port-api](./port-api/) | REST API auth, pagination, bulk operations |

### Tier 2 — Integrations

| Skill | Description |
|---|---|
| [port-github-integration](./port-github-integration/) | Exporter config, mapping files, PR/workflow ingestion |
| [port-kubernetes-integration](./port-kubernetes-integration/) | K8s exporter, CRDs, workload mapping |
| [port-terraform-provider](./port-terraform-provider/) | IaC management of blueprints and entities |
| [port-jq-patterns](./port-jq-patterns/) | JQ across mapping, automations, and computed properties |

### Tier 3 — Recipes

| Skill | Description |
|---|---|
| [port-service-onboarding](./port-service-onboarding/) | End-to-end: blueprint → ingest → scorecard → action |
| [port-production-readiness](./port-production-readiness/) | Building a production readiness scorecard |
| [port-day2-operations](./port-day2-operations/) | Scale, restart, rollback, env config patterns |
| [port-platform-rbac](./port-platform-rbac/) | Teams, roles, blueprint and action permissions |

## Relationship to Port's MCP Server

Port has an MCP server at `mcp.port.io/v1` that provides live portal data access. These skills complement it: MCP gives the agent live entity data; skills give it condensed expertise on *how* Port works. Both together is the strongest setup.

## License

Apache-2.0
