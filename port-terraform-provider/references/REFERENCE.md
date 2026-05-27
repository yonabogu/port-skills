# Port Terraform Provider — Reference

## Importing Existing Resources

```bash
# Import an existing blueprint
terraform import port_blueprint.microservice microservice

# Import an existing entity
terraform import port_entity.my_service "blueprint_id:entity_id"
# e.g.:
terraform import port_entity.payment port_entity.payment-service microservice:payment-service
```

## Aggregation Property in Terraform

```hcl
resource "port_blueprint" "service" {
  # ...
  aggregation_properties = {
    "open_issues" = {
      title  = "Open Issues"
      type   = "number"
      target = "jiraIssue"
      calculation_spec = jsonencode({
        func          = "count"
        calculationBy = "entities"
      })
      query = jsonencode({
        combinator = "and"
        rules = [
          { property = "status", operator = "=", value = "open" }
        ]
      })
    }
  }
}
```

## Mirror Property in Terraform

```hcl
resource "port_blueprint" "service" {
  # ...
  mirror_properties = {
    "team_slack" = {
      title = "Team Slack Channel"
      path  = "team.slack_channel"
    }
  }
}
```

## Variable Pattern for Credentials

```hcl
variable "port_client_id" {
  description = "Port Client ID"
  sensitive   = true
}

variable "port_client_secret" {
  description = "Port Client Secret"
  sensitive   = true
}

provider "port" {
  client_id = var.port_client_id
  secret    = var.port_client_secret
}
```
