# mylocalon-gateway (Go)

**Networking only.** Terminates tunnel WSS connections, verifies Platform JWTs
offline, routes public HTTP on `*.{base}` to the owning tunnel. No users, no
business logic, no database.

## Layout

```
cmd/gateway/        main: config, wiring, graceful shutdown (GOAWAY reconnect:true)
cmd/devtoken/       DEV ONLY: keygen + JWT minting for running without the Platform
internal/
  protocol/         wire format: typed messages, strict codec (PROTOCOL.md)
  auth/             Ed25519 JWT verify (alg pinned, kid rotation, iss/aud/exp)
  tunnel/           Hub (handshake, resume, limits) · Session (read/write pumps,
                    heartbeat, pending reqId map) · registry (in-proc routing)
  ingress/          public HTTP → HTTP_REQUEST → response; 404/413/429/502/504 pages;
                    AbuseFilter plugin hook
  platform/         optional Platform link: reserved-name sync, event publisher
  store/            runtime state: Store interface, Redis (ARCHITECTURE §5 keys) + Memory
  ratelimit/        fixed-window counters on the Store (multi-node safe, fails open)
  subdomain/        generator, DNS-label validation, blocklist, Host parsing
  server/           public router (Host-based) + admin (/health /ready /metrics)
  logging/ metrics/ clock/ httpx/ config/
```

## Interfaces (test seams)

| Interface            | Real                     | Test                    |
|----------------------|--------------------------|-------------------------|
| `store.Store`        | `store.Redis`            | `store.Memory` (fake clock) |
| `auth.Verifier`      | `auth.Ed25519Verifier`   | same, with generated keys |
| `ingress.Tunnels`    | `server.TunnelsAdapter`  | fake client over httptest |
| `ingress.AbuseFilter`| allow-all                | —                       |
| `clock.Clock`        | `clock.Real`             | `clock.Fake`            |

## Key behaviours

- **Routing:** `<label>.{base}` → ingress; `gateway.{base}`/apex/localhost →
  `/tunnel` (WSS) and `/health`. Admin binds `127.0.0.1:7003`.
- **Handshake:** subprotocol `mylocalon.v1` required (else 426) → HELLO within
  10s → version → JWT → denylist (`jti`) → per-IP/per-token conn rate →
  resume-or-fresh → per-user tunnel cap → WELCOME.
- **Reserved subdomains:** allowed iff listed in the JWT `limits.subdomains`
  claim (Platform puts the user's reservations there) — keeps the Gateway
  offline. Random subdomains are 8 chars, start with a letter.
- **Heartbeat:** Gateway PINGs every 15s; any client message resets a 45s read
  deadline. Each tick refreshes Redis TTLs (`timeout + grace`).
- **Drop:** state → `DISCONNECTED`, subdomain held 30s; public requests get
  502 "reconnecting" instead of 404. Resume rotates the resume token (stored
  as sha256 only). Grace expiry → `CLOSED`, keys removed.
- **CLOSE / protocol error:** immediate `CLOSED`, subdomain released.
- **Limits:** body 10 MiB (413 before forwarding), headers 32 KiB (431), control
  frames 64 KiB, response 10 MiB, request wait 35s (504).

## Config

See `configs/gateway.example.env` (all `ML_*`). `ML_REDIS_URL` empty → in-memory
store (single node, dev only).

## Platform link (optional, private network)

- `ML_JWKS_URL` — keys fetched from the Platform, refreshed every 5 min and on
  any unknown `kid` (rotation without restart). The last good set is kept
  during outages, so verification stays offline. PEM files are a fallback.
- `ML_PLATFORM_URL` + `ML_INTERNAL_TOKEN` — reserved-name sync every 30s
  (random subdomains skip them) and batched lifecycle events
  (`connected/resumed/disconnected/closed`) for dashboard history. Never on
  the request path; events are dropped, never blocking, if the Platform is down.

## Abuse & limits

`BasicFilter` (the `AbuseFilter` hook): operator takedown list
(`ML_BLOCKED_SUBDOMAINS`), method allowlist (no TRACE/CONNECT), malformed or
>8 KiB request targets. Per-IP public request limit (`ML_IP_REQ_RATE_PER_SEC`)
on top of per-tunnel limits.

## Not in MVP

Multi-node request handoff (Redis already records `gwNode`), binary body
streaming and WebSocket forwarding (PROTOCOL §12).
