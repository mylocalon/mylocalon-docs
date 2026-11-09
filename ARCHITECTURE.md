# mylocalon — System Architecture

## 1. Goal & shape

mylocalon forwards public HTTP traffic to a developer's local app through a
persistent outbound WebSocket the client opens to the Gateway. Two independent
services, two data stores, one edge proxy:

```
                         Internet
                            │  (TLS)
                            ▼
                    ┌───────────────┐
                    │     Caddy     │  TLS termination + host routing
                    └──────┬────────┘
              mylocalon.com  │  *.mylocalon.com
            ┌──────────────┴───────────────┐
            ▼                              ▼
   ┌─────────────────┐            ┌──────────────────┐
   │  Platform       │            │  Gateway         │
   │  (Next.js/TS)   │            │  (Go)            │
   │  auth, projects │            │  tunnels, proxy  │
   │  subdomains,API │            │  WSS, routing    │
   └───────┬─────────┘            └────────┬─────────┘
           │                               │
           ▼                               ▼
     ┌───────────┐                    ┌─────────┐
     │ PostgreSQL│  source of truth   │  Redis  │  runtime state only
     └───────────┘                    └─────────┘
                     ▲                      │
                     └── client SDK/CLI ────┘  (outbound WSS, see PROTOCOL.md)
```

The developer's machine holds one WSS connection out to the Gateway. No inbound
ports are opened on the dev machine.

## 2. Components (responsibility split)

| Component  | Owns                                                            | Never does                        |
|------------|----------------------------------------------------------------|-----------------------------------|
| Caddy      | TLS, SNI/host routing, HTTP/2                                   | business logic                    |
| Gateway    | WSS server, tunnel lifecycle, HTTP↔WS forwarding, heartbeat, rate limits | user mgmt, auth issuance, billing |
| Platform   | users, auth, API keys, projects, reserved subdomains, JWT signing, dashboard, REST API | tunnel networking |
| PostgreSQL | permanent data (users, keys, subdomains, audit)                | runtime tunnel state              |
| Redis      | runtime state (active tunnels, mappings, heartbeats, rate counters) | source of truth              |

**Hard rule:** the Gateway contains no user management or business logic. It
verifies a JWT (Platform's public key) and enforces limits. Platform and Gateway
share stores physically in MVP but never read each other's tables/keys — they
communicate only via well-defined contracts (JWT + optional read APIs).

## 3. Request flow — public HTTP hitting a tunnel

```
1. GET https://abc123.mylocalon.com/users            (browser)
2. Caddy terminates TLS, routes *.mylocalon.com → Gateway :HTTP
3. Gateway extracts subdomain "abc123"
4. Gateway looks up subdomain → tunnelId → live WS conn (Redis + in-proc map)
   - miss → 404 "tunnel not found" page
5. Gateway builds HTTP_REQUEST{reqId,...}, writes it to that tunnel's WS
6. Client (CLI/SDK) receives it, calls http://localhost:3000/users
7. Client returns HTTP_RESPONSE{reqId,status,headers,bodyB64}
8. Gateway matches reqId → writes the HTTP response back to the browser
```

Per-request state (the pending response writer keyed by `reqId`) lives **in the
Gateway process** that owns the socket, not in Redis. Redis holds only the
subdomain→tunnel mapping needed to *find* the owning process.

## 4. Auth / control flow

```
1. mylocalon login  → browser/device flow → Platform issues API key (Postgres)
2. CLI exchanges API key → short-lived Ed25519 JWT (Platform signs)
3. CLI opens WSS to Gateway, HELLO{ token=JWT }
4. Gateway verifies JWT offline with Platform's PUBLIC key (no network hop)
5. Gateway assigns subdomain, WELCOME, tunnel ACTIVE
```

See [PROTOCOL.md](./PROTOCOL.md) §4/§6 and [SECURITY.md](./SECURITY.md).

## 5. Data ownership

**PostgreSQL (Platform, source of truth)** — users, api_keys, projects,
reserved_subdomains, tunnel_audit (history), plans/limits.

**Redis (Gateway, runtime, all with TTLs)**
| Key pattern                     | Value                                  |
|---------------------------------|----------------------------------------|
| `tunnel:{tunnelId}`             | hash: subdomain, userId, state, gwNode, lastSeen |
| `subdomain:{name}`              | tunnelId (active mapping)              |
| `resume:{tunnelId}`             | hashed resumeToken (grace TTL)         |
| `user:{userId}:tunnels`         | set of tunnelIds (limit enforcement)   |
| `rl:ip:{ip}` / `rl:tok:{jti}`   | rate-limit counters                    |
| `jwt:denylist:{jti}`            | revocation (optional v1)               |

Redis is disposable: losing it drops live tunnels (clients reconnect), but never
loses users/subdomains (those are in Postgres).

## 6. Why these boundaries

- **Stateless Gateway** (offline JWT verify, state in Redis) → run N Gateway
  instances behind Caddy/LB later with no code change. A tunnel is pinned to the
  process holding its socket; the subdomain→node hint in Redis lets a future
  multi-node setup route/forward correctly. MVP runs a single Gateway node.
- **Platform owns identity/business logic** → the Gateway never needs a DB
  migration when billing/plans change.
- **Two stores by purpose, not by fashion** → Postgres for durability &
  relations, Redis for ephemeral, TTL'd, high-churn state.

## 7. Scalability posture (design-only for MVP)

Allowed now: nothing beyond a single Gateway + single Platform + shared
Postgres/Redis. Explicitly **not** in MVP: microservices, Kubernetes, message
queues, distributed consensus.

Future growth, unlocked by the above boundaries (no rewrite):
- Multiple stateless Gateway nodes; Redis `tunnel:{id}.gwNode` enables
  cross-node request handoff or sticky LB routing.
- Read replicas for Postgres; Redis cluster for counters.
- Event stream (connect/disconnect) Gateway→Platform for live dashboards — a
  thin publish, added only when the dashboard needs realtime.

## 8. Failure behavior (system level)

| Failure                | Effect                                              |
|------------------------|-----------------------------------------------------|
| Local app down         | Public 502/504 via `UPSTREAM_*` (PROTOCOL §8.3)     |
| Client socket drops    | Tunnel DISCONNECTED, subdomain held 30s, auto-resume|
| Redis down             | Live tunnels fail; clients reconnect when it returns |
| Platform down          | New logins/JWT issuance fail; **existing** tunnels keep working (offline JWT verify) |
| Gateway restart        | GOAWAY{reconnect:true}; clients reconnect            |
| Postgres down          | Login/subdomain changes fail; running tunnels unaffected |

The offline-JWT design means Platform downtime does not kill live traffic — a
deliberate resilience property of the trust boundary.
