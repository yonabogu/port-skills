# Port API — Reference

## Full Authentication Flow (shell)

```bash
#!/usr/bin/env bash
set -e

get_token() {
  curl -s -X POST "https://api.port.io/v1/auth/access_token" \
    -H "Content-Type: application/json" \
    -d "{\"clientId\":\"$PORT_CLIENT_ID\",\"clientSecret\":\"$PORT_CLIENT_SECRET\"}" \
    | jq -r '.accessToken'
}

TOKEN=$(get_token)
export PORT_TOKEN=$TOKEN
```

## Bulk Entity Delete

```bash
# Delete all entities of a blueprint
curl "https://api.port.io/v1/blueprints/microservice/entities" \
  -H "Authorization: Bearer $PORT_TOKEN" \
  | jq -r '.entities[].identifier' \
  | while read id; do
      curl -s -X DELETE "https://api.port.io/v1/blueprints/microservice/entities/$id" \
        -H "Authorization: Bearer $PORT_TOKEN"
    done
```

## Reporting Action Run Status

```bash
# Success with entity upsert
curl -X PATCH "https://api.port.io/v1/actions/runs/$PORT_RUN_ID" \
  -H "Authorization: Bearer $PORT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "status": "SUCCESS",
    "message": { "run_status": "Service scaffolded" },
    "link": ["https://github.com/my-org/new-service"]
  }'
```
