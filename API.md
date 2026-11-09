# mylocalon Platform REST API — v1

Base: `https://mylocalon.com/api/v1`. JSON in, JSON out. Breaking changes ship as
`/api/v2`; v1 only gains optional fields.

## Auth

| Method | How | Used by |
|---|---|---|
| API key | `Authorization: Bearer mlk_live_…` | CLI, SDK, scripts |
| Session | `ml_session` cookie (httpOnly, SameSite=Lax) | dashboard |

Cookie-authenticated `POST/DELETE` must send a same-origin `Origin` header
(CSRF). API calls are limited to 120/min per key; login/signup to 10/min per IP.

## Errors

```json
{ "error": { "code": "CONFLICT", "message": "\"myapp\" is already taken", "details": {} } }
```

| Status | `code` | When |
|---|---|---|
| 401 | `UNAUTHORIZED` | missing/invalid/revoked credentials |
| 403 | `FORBIDDEN` / `ACCESS_DENIED` | cross-origin cookie request / device login denied |
| 404 | `NOT_FOUND` | unknown resource or not yours |
| 409 | `CONFLICT` | duplicate email, subdomain taken |
| 410 | `GONE` | device code expired |
| 422 | `INVALID` | validation failed (unknown fields are rejected) |
| 429 | `RATE_LIMITED` | slow down |

Every response carries `x-request-id`.

## Endpoints

### Accounts
| | | |
|---|---|---|
| `POST /auth/signup` | `{email, password(≥10)}` → 201 `{user}` + cookie | |
| `POST /auth/login`  | `{email, password}` → `{user}` + cookie | |
| `POST /auth/logout` | → `{ok}`, clears cookie | |
| `GET /me` | → `{user, via}` | key or session |

### API keys
| | |
|---|---|
| `GET /keys` | → `{keys:[{id,name,prefix,createdAt,revokedAt,lastUsedAt}]}` |
| `POST /keys` `{name}` | → 201 `{key, secret}` — **secret shown only here** |
| `DELETE /keys/{id}` | revoke → `{ok}` |

### Gateway tokens
`POST /tokens` (**API key only**), optional `{ttlSec: 60..3600}` →
`{token, expiresAt}`. EdDSA JWT: `sub, iss=mylocalon-platform,
aud=mylocalon-gateway, iat, exp, jti, plan, limits{maxTunnels, maxBodyBytes,
subdomains[]}` — `subdomains` lists the user's reserved names, which is how
the Gateway authorizes them offline.

`GET /jwks` (public) → `{keys:[{kty:"OKP",crv:"Ed25519",x,kid,alg:"EdDSA",use:"sig"}]}`.

### Projects
`GET /projects` · `POST /projects {name}` → 201 · `DELETE /projects/{id}`

### Reserved subdomains
`GET /subdomains` · `POST /subdomains {name, projectId?}` → 201 ·
`DELETE /subdomains/{name}` (release).
Names: 3–63 chars `[a-z0-9-]`, no leading/trailing/double hyphen; blocklist
(`www, api, admin, gateway, app, …`) → 422; taken → 409; plan limit → 422.

### Tunnels
`GET /tunnels` → `{tunnels:[{tunnelId, subdomain, status: active|reconnecting|closed, since, lastEventAt}], history:[…]}` (last 24h).

### Device login (`mylocalon login`)
1. `POST /device/start {clientName}` → `{deviceCode, userCode, verificationUri, verificationUriComplete, interval, expiresIn}`
2. User opens `verificationUriComplete`, signs in, approves → `POST /device/decide {userCode, approve}` (session only)
3. CLI polls `POST /device/poll {deviceCode}` every `interval`s →
   `{status:"pending"}` · `{status:"approved", apiKey}` (once) · 403 denied · 410 expired.

Only `sha256(deviceCode)` is stored; the API key is minted at poll time.

### Internal (Gateway → Platform, private network, `Bearer INTERNAL_API_TOKEN`)
`GET /internal/reserved-subdomains` → `{names}` ·
`POST /internal/tunnel-events {events:[{tunnelId,userId,subdomain,event,meta?,at}]}` → 202.
Caddy blocks `/api/v1/internal/*` publicly.

### Health
`GET /health` → `{status, version, uptimeSec}` · `GET /ready` (Postgres) · `GET /metrics` (private).
