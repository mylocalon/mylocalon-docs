# mylocalon — Testing Strategy

Rule: **do not advance to the next feature until the current one works locally
and its tests pass.** Every feature ships with unit tests, integration tests,
a local runbook, success criteria, and named failure scenarios.

## 1. Test pyramid per component

| Component | Unit (fast, no network)                          | Integration                          |
|-----------|--------------------------------------------------|--------------------------------------|
| Gateway   | protocol codec, state machine, subdomain gen, JWT verify, rate limiter | WSS handshake + forward with a fake client; ingress→mock tunnel |
| SDK-node  | codec round-trips, forwarder vs stub server, backoff w/ fake clock | SDK ↔ real Gateway on localhost |
| CLI       | arg parse, config store perms, token redaction, backoff | CLI ↔ mock Platform + mock Gateway |
| Platform  | services (api-key hash/verify, JWT sign, subdomain rules), validation | route handlers + test DB (Postgres) |

Go: standard `testing` + `httptest` + table tests. TS: `vitest`, strict types in
tests too.

## 2. Shared protocol conformance fixtures

A language-neutral fixture set (`docs/fixtures/*.json`) holds canonical
message examples from [PROTOCOL.md](./PROTOCOL.md). Both the Gateway (Go) and
every SDK decode/encode against these, guaranteeing wire compatibility. Adding a
future SDK = pass the same fixtures. This is the conformance contract.

## 3. Local runbook (integration, end-to-end MVP)

Prereqs: Docker (Postgres+Redis) or local installs, Go, Node.

```
# 1. infra
docker compose up -d postgres redis        # (compose lives at repo root of each service or a dev/ dir)

# 2. keys (dev)
make gen-keys                              # writes ed25519.key / ed25519.pub

# 3. platform
cd mylocalon-platform && cp .env.example .env && npm i && npm run db:migrate && npm run dev

# 4. gateway
cd mylocalon-gateway && cp configs/gateway.example.env .env && make run

# 5. a local app to expose
python3 -m http.server 3000

# 6. cli
cd mylocalon-cli && npm i && npm run build
./dist/index.js login
./dist/index.js http 3000
# → prints https://<sub>.localtest.me:PORT  (dev domain, see below)

# 7. verify
curl -H "Host: <sub>.mylocalon.com" http://localhost:7002/    # simulate Caddy routing
```

Dev domain: use `localtest.me`/`*.localtest.me` (resolves to 127.0.0.1) or set a
`ML_BASE_DOMAIN=localtest.me` so wildcard subdomains work without DNS.

## 4. Success criteria (per phase gate)

- **Gateway slice 1:** `GET /health` → 200; unit tests green; JSON logs emit.
- **Handshake:** valid JWT → WELCOME with url; invalid JWT → GOAWAY AUTH_FAILED.
- **Forwarding:** `curl` through the tunnel returns the local app's exact bytes,
  headers, and status; `reqId` correlates in logs.
- **Heartbeat/resume:** kill the socket → DISCONNECTED → reconnect within 30s →
  **same** subdomain; after 30s → new subdomain.
- **Limits:** oversized body → 413 without reaching the client.

## 5. Failure scenarios (must be tested explicitly)

| Scenario                     | Expected                                  |
|------------------------------|-------------------------------------------|
| Invalid/expired JWT          | GOAWAY `AUTH_FAILED`, WS closed            |
| Unknown subdomain request    | public 404 page                            |
| Local app down               | public 502 (`UPSTREAM_UNREACHABLE`)        |
| Local app slow               | public 504 (`UPSTREAM_TIMEOUT`)            |
| Malformed protocol message   | `ERROR PROTOCOL_ERROR` + close             |
| Reserved subdomain not owned | GOAWAY `SUBDOMAIN_FORBIDDEN`               |
| Resume after grace window    | `RESUME_EXPIRED` → fresh tunnel/new URL    |
| Rate limit exceeded          | `429`/`RATE_LIMITED` + backoff             |
| Oversized body/headers       | 413 / 431 at edge                          |
| Redis unavailable            | graceful degrade + error logs, no crash    |

## 6. CI (per repo)

Each repo runs on push: lint + typecheck/vet + unit tests + build. Integration
tests run with ephemeral Postgres/Redis service containers. A cross-repo
"conformance" job replays the protocol fixtures against Gateway + SDK.

## 7. Conventions

- Tests are deterministic: inject a `Clock` and RNG seed (no real `time.Now()`/
  `Math.random()` in logic under test).
- No real network in unit tests — everything behind the interfaces in
  [GATEWAY.md](./GATEWAY.md) §6 and [SDK-NODE.md](./SDK-NODE.md) §6.
- Every bug fix adds a regression test first.
