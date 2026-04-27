---
name: grafana
description: Use when investigating Grafana issues
argument-hint: [optional message hint]
---

You're an expert Grafana user.

Connect to the Grafana API to investigate the user's request.

## Instance Selection
Pick the Grafana instance based on the current project directory:
| Project dir pattern | Env var | Instance |
|---|---|---|
| `ecommerce*` | `GRAFANA_RAPIDAND_API_KEY` | rapidand.grafana.net |
| `customs*` | `GRAFANA_CUSTOMS_API_KEY` | rapidcustoms.grafana.net |
| anything else | `GRAFANA_SLNC_API_KEY` | slnc.grafana.net |

## API Key Handling (CRITICAL - no leaks)
1. **Load the key into a shell variable in every Bash call.** Keys live in `~/.env`. Always start commands with:
   ```bash
   export $(grep GRAFANA_RAPIDAND_API_KEY ~/.env | xargs) && GK="$GRAFANA_RAPIDAND_API_KEY" && curl ...
   ```
   Do NOT use `source ~/.env` (variables don't persist across Bash tool calls). Use `export $(grep VAR_NAME ~/.env | xargs)` instead.
   Use `$GK` in the curl command, never paste the literal key value.
2. **Never echo, print, or cat the key.** Do not run `echo $GK`, `env | grep GRAFANA`, `cat ~/.env`, or anything that would output the key value.
3. **Never write the key into files, variables in code, or conversation text.** If you need to show an example command, use the env var name (`$GK`), not the resolved value.
4. **If a command accidentally outputs the key**, do not repeat it. Tell the user the key may have been exposed and they should rotate it.

# Grafana Cloud API Reference

## Authentication
All requests use a Bearer token:
```
Authorization: Bearer <API_KEY>
```

Test auth with: `GET /api/search?limit=1` (note: `/api/org` returns 401 on Grafana Cloud service accounts)

## Discovering Datasources
The `/api/datasources` endpoint requires higher permissions and may return 401.
Instead, use `/api/frontend/settings` which always works:
```bash
curl -s -H "Authorization: Bearer $GK" "https://<instance>.grafana.net/api/frontend/settings" \
  | jq '.datasources | keys'
```
To get a datasource's UID (needed for proxy queries):
```bash
curl -s -H "Authorization: Bearer $GK" "https://<instance>.grafana.net/api/frontend/settings" \
  | jq '.datasources["<datasource-name>"].uid'
```

### Rapidand datasources (rapidand.grafana.net)
| Name | UID | Type |
|------|-----|------|
| grafanacloud-rapidand-prom | `grafanacloud-prom` | Prometheus/Mimir |
| grafanacloud-rapidand-logs | `grafanacloud-logs` | Loki |
| grafanacloud-rapidand-traces | `grafanacloud-traces` | Tempo |

## Querying Prometheus/Mimir Metrics
Proxy endpoint: `/api/datasources/proxy/uid/<datasource-uid>/api/v1/query`

### Instant query
```bash
curl -s -G -H "Authorization: Bearer $GK" \
  "https://rapidand.grafana.net/api/datasources/proxy/uid/grafanacloud-prom/api/v1/query" \
  --data-urlencode 'query=sum by (server_address) (rate(http_client_request_duration_seconds_count{deployment_environment="prod"}[1h]))' \
  | jq '.data.result[] | {addr: .metric.server_address, qps: .value[1]}'
```

### Range query
```bash
curl -s -G -H "Authorization: Bearer $GK" \
  "https://rapidand.grafana.net/api/datasources/proxy/uid/grafanacloud-prom/api/v1/query_range" \
  --data-urlencode 'query=<promql>' \
  --data-urlencode "start=$(date -d '1 hour ago' +%s)" \
  --data-urlencode "end=$(date +%s)" \
  --data-urlencode 'step=60'
```

### List series/labels for a metric
```bash
curl -s -G -H "Authorization: Bearer $GK" \
  "https://rapidand.grafana.net/api/datasources/proxy/uid/grafanacloud-prom/api/v1/series" \
  --data-urlencode 'match[]=http_client_request_duration_seconds_count' \
  --data-urlencode "start=$(date -d '1 hour ago' +%s)" \
  --data-urlencode "end=$(date +%s)" \
  | jq '.'
```

### Key labels on http_client_* metrics (ecommerce-backend)
- `server_address` (NOT `peer_name`) - the remote host, e.g. `us17.api.mailchimp.com`
- `http_request_method` - GET, POST, PUT, DELETE, etc.
- `http_response_status_code` - 200, 404, etc.
- `deployment_environment` - prod, staging, dev-juan
- `service_namespace` - rapidand, rapidsur
- `job` - e.g. `rapidand/ecommerce-backend`

## Querying Loki Logs
Proxy endpoint: `/api/datasources/proxy/uid/<datasource-uid>/loki/api/v1/query_range`

### Search logs by text
```bash
END=$(date +%s)000000000
START=$(date -d '24 hours ago' +%s)000000000
curl -s -G -H "Authorization: Bearer $GK" \
  "https://rapidand.grafana.net/api/datasources/proxy/uid/grafanacloud-logs/loki/api/v1/query_range" \
  --data-urlencode 'query={service_name="ecommerce-backend"} |= `SEARCH_TERM`' \
  --data-urlencode "start=$START" \
  --data-urlencode "end=$END" \
  --data-urlencode "limit=50" \
  | jq '.data.result | length'
```

### Count log lines matching a pattern
```bash
curl -s -G -H "Authorization: Bearer $GK" \
  "https://rapidand.grafana.net/api/datasources/proxy/uid/grafanacloud-logs/loki/api/v1/query" \
  --data-urlencode 'query=count_over_time({service_name="ecommerce-backend"} |= `SEARCH_TERM` [24h])' \
  | jq '.data.result'
```

### Key log labels (ecommerce-backend)
- `service_name` - ecommerce-backend
- `deployment_environment` - prod, staging
- `service_namespace` - rapidand, rapidsur
- `detected_level` - info, error, warn

Note: Loki timestamps use nanoseconds (append `000000000` to unix seconds).

## Grafana Cloud Infinity Datasource
- The managed `grafanacloud-infinity` datasource **cannot fetch external URLs**. Outbound HTTP is blocked at the platform level regardless of `allowedHosts` or base URL config.
- Use `source: "inline"` with embedded JSON data instead. This works on both managed and custom Infinity datasources.
- To feed data into dashboards: build a sync script that reads data, embeds it as inline JSON in panel targets, and pushes the dashboard via `POST /api/dashboards/db`.
- Avoid `$` in JSONPath `root_selector` fields (e.g. `$[-1:]`). Grafana interprets `$` as variable interpolation and the query will 500.
- Always set `datasource` on **both** the panel object and inside each target object. Some plugin versions silently ignore the panel-level setting.
- Dashboard variables (`${var}`) in Infinity URL fields only interpolate in dashboard UI context. API queries (`/api/ds/query`) do not interpolate them.

## Python API Access
- Python `urllib` default User-Agent (`Python-urllib/3.x`) gets 403'd by Grafana Cloud. Always set `User-Agent: <your-tool>/1.0` explicitly.
- For scripts that push dashboards, fetch the current version first (`GET /api/dashboards/uid/<uid>` -> `meta.version`) and include it in the payload to avoid version conflicts.

## Tips
- The service account `rapidand-backups` has Admin role but some endpoints (like `/api/datasources`) still return 401 on Grafana Cloud. Use `/api/frontend/settings` as a workaround.
- Use `jq` to parse results. Prometheus results are in `.data.result[]`, Loki results in `.data.result[]`.
- For high-cardinality label exploration, use `/api/v1/series` with a `match[]` filter rather than `/api/v1/label/<name>/values`.
