# mylocalon — Observability

MVP goal: you can answer "what happened to this request/tunnel?" from logs alone,
with correlation ids, plus basic health/metrics endpoints. Prometheus and
OpenTelemetry are **designed for** but not wired up now (stubs only).

## 1. Structured logging

- **Format:** JSON, one object per line. Go uses `log/slog`; TS uses `pino`.
- **Required fields:** `ts`, `level`, `msg`, `service`, `env`, plus context
  (`tunnelId`, `reqId`, `userId`, `subdomain`, `remoteIp`, `durationMs`) where
  applicable.
- **Redaction:** `token`, `authorization`, `apiKey`, `secret`, cookie values are
  never emitted (redaction middleware in both services). Bodies are never logged.
- **Levels:** `error` (needs attention), `warn` (degraded), `info` (lifecycle),
  `debug` (dev only).

Example:
```json
{"ts":"2026-07-24T10:00:00Z","level":"info","service":"gateway",
 "msg":"http_forwarded","tunnelId":"tnl_abc","reqId":"rq_01H","subdomain":"abc123",
 "method":"GET","path":"/health","status":200,"durationMs":12,"remoteIp":"203.0.113.9"}
```

## 2. Correlation ids

- `reqId` (Gateway-generated) ties a public request → forwarded message →
  response across log lines, and is echoed to the client SDK for its own logs.
- A `traceId` (per public request) is generated at ingress and propagated in
  logs; it becomes the OpenTelemetry trace id when tracing is enabled later.
- Tunnel lifecycle logs carry `tunnelId` from CONNECTING → CLOSED, so a tunnel's
  whole life is greppable.

## 3. Error tracking

- Every `error` log includes a stack trace (Go: `%+v`/wrapped errors; TS: pino
  `err` serializer) and the active correlation ids.
- Panics/uncaught rejections are recovered at the top of each request/goroutine,
  logged with context, and converted to a 500/502 — never crash the process.
- Future: a Sentry/OTel exporter is a drop-in; logs already carry the needed ids.

## 4. Request & connection logs

| Event                | Fields                                                   |
|----------------------|----------------------------------------------------------|
| `http_ingress`       | reqId, subdomain, method, path, remoteIp, bytesIn         |
| `http_forwarded`     | reqId, tunnelId, status, durationMs, bytesOut             |
| `tunnel_connect`     | tunnelId, userId, subdomain, clientVersion, protocol      |
| `tunnel_state`       | tunnelId, from, to                                        |
| `tunnel_disconnect`  | tunnelId, reason, durationSec                             |
| `upstream_error`     | reqId, tunnelId, code (UPSTREAM_*), message               |

## 5. Health & metrics endpoints

- **`GET /health`** (both services) → `200 {"status":"ok","version":"...","uptimeSec":N}`.
  Also a `/ready` that checks Redis/Postgres reachability for load balancers.
- **`GET /metrics`** — MVP: a stub returning basic counters in Prometheus text
  format, designed to become a real Prometheus endpoint. Bind on the private
  admin address only.

## 6. Basic metrics (MVP counters/gauges)

| Metric                          | Type    | Source  |
|---------------------------------|---------|---------|
| `mylocalon_active_tunnels`       | gauge   | Gateway |
| `mylocalon_requests_total`       | counter | Gateway |
| `mylocalon_request_duration_ms`  | histogram (stub) | Gateway |
| `mylocalon_tunnel_connects_total`| counter | Gateway |
| `mylocalon_upstream_errors_total`| counter | Gateway |
| `mylocalon_api_requests_total`   | counter | Platform|

## 7. Future (designed, not built)

- **Prometheus:** flesh out `/metrics` with a real registry + histograms.
- **OpenTelemetry:** wrap ingress + forwarding in spans using the existing
  `traceId`; export via OTLP. No code restructure needed — the ids exist today.
- **Dashboards/alerts:** on active tunnels, error rate, p95 latency, connect
  failures.

Do not build these in MVP; the logging + stubs above are the deliverable.
