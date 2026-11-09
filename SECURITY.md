# mylocalon — Security

Production-hardened from day one, without over-engineering. This doc is the
single reference for trust boundaries, tokens, limits, and secret handling.

## 1. Trust boundaries

```
untrusted internet ─▶ Caddy (TLS) ─▶ [Gateway ingress]  ← public HTTP
untrusted clients  ─▶ Caddy (TLS) ─▶ [Gateway WSS]       ← must present valid JWT
Gateway ──verifies JWT with Platform PUBLIC key (offline)──▶ trusts identity
Platform ── owns identity, signs JWT with PRIVATE key ──▶ never exposed
internal localhost (Gateway↔Redis, Platform↔Postgres) ── isolated, may be plain HTTP
```

Rule: **internal services are never exposed to the internet** — only Caddy is.
Gateway/Platform bind to localhost/private interfaces.

## 2. Tokens

- **API key** (CLI): opaque `mlk_live_<random>`. Stored **hashed** (Argon2id/
  bcrypt) in Postgres; only a `prefix` is kept in cleartext for display. Shown to
  the user exactly once. Revocable (`DELETE /keys/{id}`).
- **Gateway JWT**: Ed25519, short TTL (≤1h), claims `sub/iss/aud/exp/iat[/jti]`.
  Signed by Platform's private key; Gateway verifies with the public key —
  **offline**, no network hop, and the Gateway cannot mint tokens.
- **Revocation:** primary control is the short TTL. Immediate revocation via a
  Redis `jwt:denylist:{jti}` the Gateway checks (optional in v1).
- **Rotation:** signing keys carry a `kid`; JWKS serves old+new during overlap so
  in-flight JWTs verify while new ones use the new key.

### Token hygiene
- Tokens are **never logged** at any level (structured logger redacts `token`,
  `authorization`, `apiKey`, `secret`).
- Never placed in URLs/query strings (WSS handshake carries JWT in the first
  message, not the URL).
- Config file `~/.mylocalon/config.json` is `chmod 0600`; CLI refuses looser perms.

## 3. Rate limiting & connection limits

| Limit                        | Scope        | Where     |
|------------------------------|--------------|-----------|
| New tunnel connections / min | per IP, per token | Gateway (Redis counters) |
| HTTP requests / sec          | per tunnel / per IP | Gateway |
| Concurrent tunnels           | per user     | Gateway (`user:{id}:tunnels`) |
| API calls / min              | per API key  | Platform  |

Exceed → `429`/`RATE_LIMITED` with backoff guidance. Counters use fixed/sliding
windows in Redis; distributed-safe for future multi-node.

## 4. Request-size & shape limits

From [PROTOCOL.md](./PROTOCOL.md) §13, enforced at the Gateway edge **before**
forwarding: max body (10 MiB, plan-overridable → 413), max header bytes (32 KiB →
431), max control message (64 KiB). Prevents memory-exhaustion abuse.

## 5. Input validation (all boundaries)

- Platform: every route validates input with zod schemas; reject unknown fields.
- Gateway: strict protocol decode; malformed message → `PROTOCOL_ERROR` + close.
- Subdomain names: DNS-label regex + blocklist (see [API.md](./API.md)).
- Host header on ingress: parse subdomain safely; reject malformed hosts.

## 6. Abuse prevention (start simple, design for plugins)

MVP: block obviously-malicious host/path patterns and enforce the limits above.
Design hook: an `AbuseFilter` interface in the Gateway ingress
(`Allow(req) (bool, reason)`) so content/domain scanning plugins can be added
later without touching core routing. No heavy scanning in MVP.

## 7. TLS

Caddy terminates TLS for `mylocalon.com` and `*.mylocalon.com` (wildcard cert via
ACME DNS challenge). HSTS on. Internal hops on isolated localhost may be plain
HTTP. WSS end-to-end from client to Caddy.

## 8. Secret handling

- No secrets in code or git. `.env` in dev (gitignored), a secrets manager/vault
  or systemd `LoadCredential`/`EnvironmentFile` (mode 0600) in prod.
- Secrets: Postgres creds, Redis creds, `SESSION_SECRET`, Ed25519 **private** key
  (Platform only), Caddy ACME creds.
- **Rotation strategy** (documented, manual at start):
  - JWT signing key: generate new keypair → publish public via JWKS with new
    `kid` → flip `active` → retire old after max-JWT-TTL. Zero downtime.
  - API keys: user-initiated revoke + recreate.
  - DB/Redis creds: rotate in vault → rolling restart.

## 9. Denial-of-service posture

Connection + rate + size limits are the first line. Caddy provides basic
connection throttling. Heavy DDoS is out of scope for MVP (would sit at the
network edge/CDN later) — documented as a known limitation.

## 10. Checklist (must hold before "production")

- [ ] TLS everywhere; internal services not internet-exposed.
- [ ] JWT verified offline; Gateway holds only the public key.
- [ ] API keys hashed at rest; shown once; revocable.
- [ ] Tokens never logged; config file 0600.
- [ ] Body/header/message size caps enforced at edge.
- [ ] Per-IP + per-token rate limits; per-user tunnel cap.
- [ ] Input validation on every boundary.
- [ ] Secrets in env/vault, rotation documented.
