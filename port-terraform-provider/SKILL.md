---
name: port-terraform-provider
description: >
  Port.io Terraform provider expert. Use when managing Port resources with Terraform/OpenTofu:
  configuring the port-labs/port provider, writing resources for blueprints (port_blueprint),
  entities (port_entity), actions (port_action), scorecards (port_scorecard), automations
  (port_automation), webhooks (port_webhook), and reading data with data sources. Essential when
  the user mentions Port Terraform, IaC for Port, port-labs/port provider, or managing Port
  configuration as code.
license: Apache-2.0
metadata:
  author: port-io
  version: "1.0"
compatibility: Designed for use with Port.io (port.io) and Terraform >= 1.0 / OpenTofu >= 1.6.
---

# Port Terraform Provider

The `port-labs/port` Terraform provider manages Port resources as code.

## Provider Configuration

```hcl
terraform {
  required_providers {
    port = {
      source  = "port-labs/port"
      version = "~> 2.0"
    }
  }
}

provider "port" {
  client_id     = var.port_client_id       # or PORT_CLIENT_ID env var
  secret        = var.port_client_secret   # or PORT_CLIENT_SECRET env var
  base_url      = "https://api.port.io"    # use https://api.us.port.io for US region
}
```

## Blueprint Resource

```hcl
resource "port_blueprint" "microservice" {
  identifier = "microservice"
  title      = "Microservice"
  icon       = "Microservice"
  description = "A backend microservice"

  properties = {
    string_props = {
      "language" = { title = "Language", required = true }
      "tier" = {
        title  = "Tier"
        enum   = ["tier-1", "tier-2", "tier-3"]
        enum_colors = {
          "tier-1" = "red"
          "tier-2" = "yellow"
          "tier-3" = "green"
        }
      }
      "on_call" = { title = "On-Call", format = "user" }
      "readme_url" = { title = "README", format = "url" }
    }
    number_props = {
      "replicas" = { title = "Replicas", default = 1 }
    }
    boolean_props = {
      "is_active" = { title = "Active", default = true }
    }
    array_props = {
      "tags" = { title = "Tags", string_items = {} }
    }
  }

  relations = {
    "team" = {
      title    = "Team"
      target   = "team"
      required = false
      many     = false
    }
  }

  calculation_properties = {
    "datadog_url" = {
      title       = "Datadog"
      type        = "string"
      format      = "url"
      calculation = "\"https://app.datadoghq.com/logs?service=\" + .identifier"
    }
  }
}
```

## Entity Resource

```hcl
resource "port_entity" "payment_service" {
  identifier = "payment-service"
  title      = "Payment Service"
  blueprint  = port_blueprint.microservice.identifier

  properties = {
    string_props = {
      "language" = "go"
      "tier"     = "tier-1"
      "on_call"  = "alice@example.com"
    }
    number_props = {
      "replicas" = 3
    }
    boolean_props = {
      "is_active" = true
    }
  }

  relations = {
    "team" = {
      value = "platform-team"
    }
  }
}
```

## Scorecard Resource

```hcl
resource "port_scorecard" "production_readiness" {
  blueprint  = port_blueprint.microservice.identifier
  identifier = "ProductionReadiness"
  title      = "Production Readiness"

  levels = [
    { color = "paleBlue", title = "Basic"  },
    { color = "bronze",   title = "Bronze" },
    { color = "silver",   title = "Silver" },
    { color = "gold",     title = "Gold"   }
  ]

  rules = [
    {
      identifier = "has_on_call"
      title      = "Has On-Call"
      level      = "Bronze"
      query = {
        combinator = "and"
        conditions = [
          jsonencode({ property = "on_call", operator = "isNotEmpty" })
        ]
      }
    },
    {
      identifier = "has_readme"
      title      = "Has README"
      level      = "Bronze"
      query = {
        combinator = "and"
        conditions = [
          jsonencode({ property = "readme_url", operator = "isNotEmpty" })
        ]
      }
    }
  ]
}
```

## Action Resource

```hcl
resource "port_action" "scaffold_service" {
  blueprint  = port_blueprint.microservice.identifier
  identifier = "scaffold_service"
  title      = "Scaffold New Service"
  icon       = "Git"
  operation  = "CREATE"

  user_properties = {
    string_props = {
      "service_name" = { title = "Service Name", required = true }
    }
  }

  github_method = {
    org      = "my-org"
    repo     = "port-actions"
    workflow = "scaffold-service.yml"
  }
}
```

## Data Sources

```hcl
# Read an existing blueprint
data "port_blueprint" "team" {
  identifier = "team"
}

# Read an existing entity
data "port_entity" "platform_team" {
  identifier = "platform-team"
  blueprint  = "team"
}

output "team_slack" {
  value = data.port_entity.platform_team.properties.string_props["slack_channel"]
}
```

## Gotchas

- Provider version `~> 2.0` is the current major version. Version 1.x had a different schema — don't mix docs.
- `enum_colors` only applies to `string_props` with `enum` defined.
- Blueprint `identifier` changes require destroying and recreating the resource — import existing ones with `terraform import`.
- Use `jsonencode()` for scorecard rule conditions — they expect a JSON string.
- The `relations` block in `port_entity` uses `value` for single relations and `values` (list) for many relations.
- Set `PORT_CLIENT_ID` and `PORT_CLIENT_SECRET` as environment variables instead of hard-coding in provider config.
