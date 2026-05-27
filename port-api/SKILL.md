---
name: port-api
description: >
  Port.io REST API expert. Use when authenticating to Port's API, forming correct API requests,
  handling pagination, performing bulk operations, understanding base URLs (EU vs US regions),
  generating bearer tokens, and using the most common endpoints: entities CRUD, blueprints,
  actions, action runs, scorecards, automations. Essential when the user asks about the Port API,
  HTTP calls to Port, token generation, or API authentication.
license: Apache-2.0
metadata:
  author: port-io
  version: "1.0"
compatibility: Designed for use with Port.io (port.io). Requires Port client credentials.
---

# Port REST API

## Base URLs

| Region | URL |
|---|---|
| Europe (default) | `https://api.port.io/v1` |
| United States | `https://api.us.port.io/v1` |

All API calls use the same regional base URL throughout a project. Don't mix regions.

## Authentication

Port uses OAuth 2.0 client credentials to generate a short-lived bearer token (valid 3 hours).

### Step 1: Get a token

```bash
TOKEN=$(curl -s -X POST "https://api.port.io/v1/auth/access_token" \
  -H "Content-Type: application/json" \
  -d '{
    "clientId": "'$PORT_CLIENT_ID'",
    "clientSecret": "'$PORT_CLIENT_SECRET'"
  }' | jq -r '.accessToken')
```

Store `PORT_CLIENT_ID` and `PORT_CLIENT_SECRET` as secrets. Find them in Port → Settings → Credentials.

### Step 2: Use the token

```bash
curl "https://api.port.io/v1/blueprints" \
  -H "Authorization: Bearer $TOKEN"
```

## Common Endpoints

### Blueprints

```
GET    /blueprints                      # List all blueprints
GET    /blueprints/{id}                 # Get one blueprint
POST   /blueprints                      # Create blueprint
PATCH  /blueprints/{id}                 # Update blueprint (partial)
DELETE /blueprints/{id}                 # Delete blueprint
```

### Entities

```
GET    /blueprints/{bp}/entities               # List entities
GET    /blueprints/{bp}/entities/{id}          # Get entity
POST   /blueprints/{bp}/entities               # Create entity
POST   /blueprints/{bp}/entities?upsert=true   # Upsert entity
PATCH  /blueprints/{bp}/entities/{id}          # Update entity (partial)
DELETE /blueprints/{bp}/entities/{id}          # Delete entity
POST   /entities/search                        # Search across blueprints
```

### Actions

```
GET    /blueprints/{bp}/actions         # List actions
POST   /blueprints/{bp}/actions         # Create action
PATCH  /blueprints/{bp}/actions/{id}    # Update action
DELETE /blueprints/{bp}/actions/{id}    # Delete action
```

### Action Runs

```
GET    /actions/runs                    # List runs
GET    /actions/runs/{runId}            # Get run status
PATCH  /actions/runs/{runId}            # Update run status (SUCCESS/FAILURE)
POST   /actions/runs/{runId}/logs       # Add log line to run
```

### Scorecards

```
GET    /blueprints/{bp}/scorecards      # List scorecards
POST   /blueprints/{bp}/scorecards      # Create scorecard
PATCH  /blueprints/{bp}/scorecards/{id} # Update scorecard
DELETE /blueprints/{bp}/scorecards/{id} # Delete scorecard
```

### Automations

```
GET    /automations                     # List automations
POST   /automations                     # Create automation
PATCH  /automations/{id}                # Update automation
DELETE /automations/{id}                # Delete automation
```

## Pagination

```bash
curl "https://api.port.io/v1/blueprints/microservice/entities?limit=50&page=2" \
  -H "Authorization: Bearer $TOKEN"
```

Response includes pagination metadata. Default limit varies by endpoint; max is typically 50–100.

## Request Body Limit

Max request body: **1 MiB**. For large bulk operations, split into batches.

## Useful Headers

| Header | Purpose |
|---|---|
| `Authorization: Bearer <token>` | Required on every request |
| `Content-Type: application/json` | Required for POST/PATCH/PUT |
| `X-Trace-Id` | Returned in responses for debugging with Port support |

## Error Handling

| Status | Meaning |
|---|---|
| `200` / `201` | Success |
| `400` | Bad request — check JSON schema |
| `401` | Token missing or expired — re-authenticate |
| `403` | Insufficient permissions |
| `404` | Resource not found |
| `409` | Conflict — entity already exists (use `upsert=true`) |
| `429` | Rate limited — back off and retry |

## Python SDK Pattern

```python
import requests, os

def get_port_token():
    resp = requests.post(
        "https://api.port.io/v1/auth/access_token",
        json={"clientId": os.environ["PORT_CLIENT_ID"],
              "clientSecret": os.environ["PORT_CLIENT_SECRET"]}
    )
    resp.raise_for_status()
    return resp.json()["accessToken"]

token = get_port_token()
headers = {"Authorization": f"Bearer {token}"}

# Upsert entity
requests.post(
    "https://api.port.io/v1/blueprints/microservice/entities",
    headers=headers,
    params={"upsert": "true", "merge": "true"},
    json={
        "identifier": "my-service",
        "title": "My Service",
        "properties": {"language": "python"}
    }
).raise_for_status()
```

## Gotchas

- Tokens expire after **3 hours** — build token refresh into long-running scripts.
- EU and US APIs are completely separate tenants; credentials for one don't work on the other.
- `PATCH` on entities is a **partial update** — only fields included in the body are changed.
- `DELETE /blueprints/{id}` also deletes all entities of that blueprint. This is irreversible.
- The `X-Trace-Id` response header is useful when filing support tickets with Port.
