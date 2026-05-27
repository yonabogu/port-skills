---
name: port-platform-rbac
description: >
  Port.io RBAC and permissions expert. Use when configuring access control in Port: understanding
  the three roles (Admin, Moderator, Member), setting blueprint-level permissions, restricting
  which teams can execute actions, configuring team ownership on entities, syncing teams from SSO/
  identity providers, and using dynamic jqQuery conditions on action permissions. Essential when
  the user asks about Port permissions, role-based access, team ownership, restricting action
  visibility, or managing users and teams.
license: Apache-2.0
metadata:
  author: port-io
  version: "1.0"
compatibility: Designed for use with Port.io (port.io).
---

# Port Platform RBAC

Port implements role-based access control at the user, team, and blueprint levels.

## Core Roles

| Role | Scope | Permissions |
|---|---|---|
| **Admin** | Global | Full access to everything |
| **Moderator** | Per blueprint | Manage a specific blueprint and its entities |
| **Member** | Global | Read-only + execute self-service actions (if permitted) |

Moderators are blueprint-specific — a user can be Moderator of `microservice` but Member of `k8sCluster`.

## Teams

Teams are first-class entities in Port. Every Port org has a `team` blueprint auto-created.

```json
// Team entity structure
{
  "identifier": "platform-team",
  "title": "Platform Team",
  "blueprint": "team",
  "properties": {
    "slack_channel": "#platform",
    "email": "platform@example.com"
  }
}
```

Teams are synced from SSO (Okta, Azure AD, Google Workspace) or managed manually via Settings → Teams.

**When SSO is enabled**: Teams from your IdP are read-only in Port — don't try to edit/delete them manually.

## Entity Ownership

Assign a team to an entity via the `team` root field:

```json
{
  "identifier": "payment-service",
  "blueprint": "microservice",
  "team": ["platform-team"],
  "properties": { ... }
}
```

Or set ownership through a `team` relation (prefer this for traceability):

```json
{
  "identifier": "payment-service",
  "relations": { "team": "platform-team" }
}
```

## Blueprint Permissions

Configure at blueprint level to control entity CRUD per role:

```json
{
  "entities": {
    "create": { "roles": ["Admin", "Moderator"], "users": [], "teams": [] },
    "update": { "roles": ["Admin", "Moderator"], "users": [], "teams": ["platform-team"] },
    "delete": { "roles": ["Admin"],              "users": [], "teams": [] },
    "register": { "roles": ["Admin", "Moderator", "Member"] }
  }
}
```

Set via API:

```bash
curl -X PUT "https://api.port.io/v1/blueprints/microservice/permissions" \
  -H "Authorization: Bearer $PORT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "entities": {
      "register": { "roles": ["Admin", "Moderator", "Member"] },
      "create":   { "roles": ["Admin", "Moderator"] },
      "update": {
        "roles": ["Admin", "Moderator"],
        "ownedEntitiesOnly": true
      },
      "delete": { "roles": ["Admin"] }
    }
  }'
```

`ownedEntitiesOnly: true` — Members can only update entities where they are in the `$team` meta-property.

## Action Permissions

Restrict who can execute a specific action:

### Simple role-based

```json
"trigger": {
  "type": "self-service",
  "operation": "DAY-2",
  ...
  "requiredApproval": false,
  "userInputs": { ... }
}
```

Set in action permissions (separate from trigger):

```bash
curl -X PATCH "https://api.port.io/v1/blueprints/microservice/actions/scale_service/permissions" \
  -H "Authorization: Bearer $PORT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "execute": {
      "roles": ["Admin", "Moderator"],
      "users": [],
      "teams": ["platform-team"],
      "ownedEntitiesOnly": false
    },
    "approve": {
      "roles": ["Admin"],
      "users": ["alice@example.com"]
    }
  }'
```

### Dynamic permissions using jqQuery

Restrict action execution to the owning team of the entity:

```json
"execute": {
  "jqQuery": ".user.teams | map(.name) | contains([.entity.team[0]])"
}
```

Only users in the same team as the entity can execute:

```json
"execute": {
  "jqQuery": "(.user.teams | map(.name)) | any(. == .entity.relations.team)"
}
```

## Service Accounts (Bots)

For CI/CD and integrations that call the Port API:

1. Create via: Settings → Users → Service Accounts
2. Email format: `<name>.serviceaccounts.getport.io`
3. Assign roles and blueprint permissions like any user
4. Credentials: client ID + secret (same auth flow as regular API access)

## SSO Team Sync

When SSO is configured, Port syncs teams on each user login. The `$team` meta-property on entities is automatically populated based on which teams the user belongs to (if ownership is inherited).

SSO providers supported: Okta, Azure AD, Google Workspace, OneLogin, PingOne, Jumpcloud.

## Gotchas

- SSO-synced teams **cannot be edited or deleted** from Port's UI — make changes in your IdP.
- `ownedEntitiesOnly: true` checks the `$team` meta-property, not the `team` relation. These are different.
- Moderator role is blueprint-specific — users don't automatically get Moderator access across all blueprints.
- Service accounts consume a user seat in your Port license.
- Dynamic `jqQuery` permissions are evaluated at action-trigger time using the current user context and entity state.
