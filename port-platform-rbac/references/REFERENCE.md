# Port Platform RBAC — Reference

## Permissions API Endpoints

```bash
# Get blueprint permissions
GET  /blueprints/{id}/permissions

# Update blueprint permissions
PUT  /blueprints/{id}/permissions

# Get action permissions
GET  /blueprints/{bp}/actions/{action}/permissions

# Update action permissions
PATCH /blueprints/{bp}/actions/{action}/permissions
```

## Full Blueprint Permissions Schema

```json
{
  "entities": {
    "register": {
      "roles": ["Admin", "Moderator", "Member"],
      "users": [],
      "teams": []
    },
    "create": {
      "roles": ["Admin", "Moderator"],
      "users": [],
      "teams": []
    },
    "update": {
      "roles": ["Admin", "Moderator"],
      "users": [],
      "teams": [],
      "ownedEntitiesOnly": false
    },
    "delete": {
      "roles": ["Admin"],
      "users": [],
      "teams": []
    }
  }
}
```

## jqQuery Examples for Dynamic Action Permissions

```json
// Allow if user is in 'platform-team'
".user.teams | map(.name) | contains([\"platform-team\"])"

// Allow if user is the on-call owner of the entity
".user.email == .entity.properties.on_call"

// Allow if entity tier is not tier-1 (tier-1 requires Admin)
".entity.properties.tier != \"tier-1\""

// Allow only during business hours (UTC)
"(now | strftime(\"%H\") | tonumber) >= 9 and (now | strftime(\"%H\") | tonumber) <= 17"
```

## Assigning Users to Blueprints as Moderators

Via Settings → Users → click user → Assign as moderator for specific blueprints.

Or via API:

```bash
curl -X PATCH "https://api.port.io/v1/users/{user_id}" \
  -H "Authorization: Bearer $PORT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "roles": ["Member"],
    "blueprintRoles": {
      "microservice": "Moderator",
      "deployment": "Member"
    }
  }'
```
