# Port JQ Patterns — Reference

## Mapping YAML Cheat Sheet

```yaml
port:
  entity:
    mappings:
      identifier: .id | tostring        # convert number to string identifier
      title: .name
      blueprint: '"my-blueprint"'        # string literal needs double quotes inside single quotes
      properties:
        # Primitives
        count:    .count
        name:     .name
        active:   .active
        # Nested
        owner:    .owner.login
        # Default
        lang:     '.language // "unknown"'
        # Conditional
        status:   'if .archived then "archived" else "active" end'
        # Concatenation
        url:      '(.html_url // "")'
        # Date
        created:  .created_at           # already ISO 8601? pass through
        epoch_ts: '.timestamp | todate' # epoch to ISO 8601
      relations:
        team:    .owner.login
        parent:  .parent.full_name
```

## Automation Condition Quick Reference

```json
// Entity property check
".diff.after.properties.ENV_VAR == \"production\""

// Meta-property
".diff.after.title | startswith(\"prod-\")"

// Team ownership set
".diff.after.team | length > 0"

// Scorecard
".diff.after.scorecards.ScoreId.level"

// Multiple conditions (use combinator: "and")
// Expression 1: ".diff.after.properties.tier == \"tier-1\""
// Expression 2: ".diff.after.properties.on_call == null"
```

## Calculation Property Examples

```json
// URL builder
"\"https://app.datadoghq.com/logs?query=service:\" + .identifier"

// Slack deep link
"\"https://slack.com/app_redirect?channel=\" + .properties.slack_channel"

// Percentage
"(.properties.passing_checks / .properties.total_checks * 100 | floor | tostring) + \"%\""

// Environment label
"if (.properties.environments // [] | contains([\"production\"])) then \"prod\" else \"non-prod\" end"
```
