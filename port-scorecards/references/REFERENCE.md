# Port Scorecards — Reference

## Level Colors

Standard palette values for `color` in levels:

`red`, `orange`, `yellow`, `green`, `blue`, `purple`, `paleBlue`, `bronze`, `silver`, `gold`, `grey`

## Checking Scorecard Results via API

```bash
# Entities at a specific level
curl "https://api.port.io/v1/blueprints/microservice/entities?scorecards.ProductionReadiness.level=Gold" \
  -H "Authorization: Bearer $PORT_TOKEN"
```
