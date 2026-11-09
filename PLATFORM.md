# mylocalon-platform (Next.js + TypeScript)

**Business logic and identity.** Everything the Gateway must not do: users,
authentication, API keys, projects, reserved subdomains, JWT signing, the
dashboard, and the REST API. Later: billing.

## 1. Responsibilities

- User accounts and authentication (session for dashboard, API keys for CLI).
- Issue **short-lived Ed25519 JWTs** that the Gateway verifies offline.
- Manage projects, API keys (create/revoke), reserved subdomains.
- Serve the dashboard (login, projects, tunnel status/history).
- Expose the versioned REST API ([API.md](./API.md)).
- Own PostgreSQL — the source of truth.

The Platform never terminates tunnel traffic and never talks the tunnel
protocol.

## 2. Auth model

- **Dashboard** → cookie session (httpOnly, SameSite=Lax, secure).
- **CLI/SDK** → user creates an **API key** (opaque, shown once, hashed at rest).
  The CLI exchanges the API key at `POST /api/v1/tokens` for a short-lived JWT
  used with the Gateway. API keys are long-lived + revocable; JWTs are
  short-lived so the Gateway can stay stateless. See [SECURITY.md](./SECURITY.md).
- **JWT signing:** Ed25519 private key held only by the Platform. Public key is
  published to the Gateway (file/endpoint). Key id (`kid`) in the JWT header
  enables rotation.

## 3. Data model (PostgreSQL — source of truth)

```
users(id, email, password_hash|oauth, created_at)
api_keys(id, user_id, name, key_hash, prefix, created_at, revoked_at, last_used_at)
projects(id, user_id, name, created_at)
reserved_subdomains(id, user_id, name UNIQUE, project_id?, created_at, released_at)
tunnel_audit(id, tunnel_id, user_id, subdomain, event, meta_jsonb, at)  -- history
signing_keys(kid, public_pem, private_pem_enc, active, created_at, retired_at)
plans/limits(user_id, max_tunnels, max_body_bytes, ...)   -- MVP: static defaults
```

- `reserved_subdomains.name` is globally unique — enforces subdomain uniqueness
  for reserved names. Generated subdomains are checked for collision at
  allocation time by the Gateway against Redis + this table.
- `tunnel_audit` is **history only**; live state lives in Redis (Gateway).

## 4. Request flow (CLI login → tunnel)

```
1. mylocalon login → opens browser device flow → user authenticates
2. Platform creates API key, returns it to CLI once
3. CLI stores API key in ~/.mylocalon/config.json (see CLI.md)
4. On `mylocalon http 3000`: CLI POST /api/v1/tokens (Bearer API key)
   → Platform verifies key, signs JWT{sub,exp,aud:gateway}, returns it
5. CLI opens WSS to Gateway with the JWT
```

## 5. Folder structure (Next.js App Router)

```
mylocalon-platform/
├── app/
│   ├── (dashboard)/            # authed UI: projects, subdomains, tunnels
│   ├── (auth)/login/           # login / device-flow approve
│   └── api/v1/                 # route handlers (REST API, see API.md)
│       ├── tokens/route.ts     # API key → JWT exchange
│       ├── keys/route.ts       # create/list/revoke API keys
│       ├── projects/route.ts
│       ├── subdomains/route.ts # reserve / release
│       └── jwks/route.ts       # publish public signing keys
├── src/
│   ├── domain/                 # entities + use-cases (framework-free)
│   ├── services/               # auth, tokens (JWT sign), subdomains, keys
│   ├── db/                     # Prisma/Drizzle schema + migrations + repos
│   ├── auth/                   # session, api-key hashing/verify
│   └── lib/                    # config, logger, errors, validation (zod)
├── prisma/ (or drizzle/)
└── package.json                # strict TS
```

Clean layering: `domain` (pure) ← `services` ← `api routes`/`app` (delivery).
DB access only through repositories.

## 6. Interfaces (service seams)

```ts
interface TokenService {
  issueGatewayJWT(userId: string, opts?: { ttlSec?: number }): Promise<string>;
}
interface ApiKeyService {
  create(userId: string, name: string): Promise<{ id: string; secret: string }>;
  verify(secret: string): Promise<{ userId: string; keyId: string } | null>;
  revoke(userId: string, keyId: string): Promise<void>;
}
interface SubdomainService {
  reserve(userId: string, name: string): Promise<Reserved>;   // enforces uniqueness
  release(userId: string, name: string): Promise<void>;
  isOwnedBy(userId: string, name: string): Promise<boolean>;  // used by Gateway path
}
```

## 7. Edge cases

| Case                                  | Handling                                    |
|---------------------------------------|---------------------------------------------|
| API key exchange with revoked key     | 401; CLI prompts re-login                    |
| Reserve an already-taken subdomain    | 409 conflict                                 |
| Reserve a reserved-word/blocklist name| 422 (blocklist: `www`, `api`, `admin`, …)    |
| JWT signing key rotation              | new `kid`; JWKS serves both during overlap   |
| Concurrent key creation               | unique constraints + transactions            |

## 8. Config (env)

```
DATABASE_URL=postgres://...
JWT_PRIVATE_KEY_PATH / JWT_PUBLIC_KEY_PATH
JWT_ISSUER=mylocalon-platform
JWT_AUDIENCE=mylocalon-gateway
JWT_TTL_SECONDS=3600
SESSION_SECRET=...
BASE_DOMAIN=mylocalon.com
```

## 9. Build order (Phase 5)

1. DB schema + migrations + config/logger.
2. Auth (session) + API keys (create/hash/verify/revoke).
3. `POST /tokens` JWT issuance + `GET /jwks`.
4. Projects + reserved subdomains (uniqueness).
5. Dashboard pages (Phase 6).
6. `isOwnedBy` read path for the Gateway's reserved-subdomain check.

## 10. Implementation notes (as built)

- DB access is plain SQL over `pg` with hand-written migrations
  (`src/db/migrations`, applied by `npm run db:migrate`), not an ORM.
- Sessions are DB rows keyed by `sha256(cookie)`; API keys and passwords use
  argon2id; key lookup is by the cleartext 17-char prefix.
- Signing keys come from files / systemd credentials (`JWT_PRIVATE_KEY_PATH`,
  `JWT_EXTRA_PUBLIC_KEYS` for rotation) instead of a `signing_keys` table, so
  the private key never enters the database.
- `plans/limits` are static env defaults (`MAX_TUNNELS`, `MAX_BODY_BYTES`,
  `MAX_RESERVED_SUBDOMAINS`) until billing exists.
- Rate limits are in-process (single Platform instance); move to Redis when scaling out.
- Requires Node 20–22.
