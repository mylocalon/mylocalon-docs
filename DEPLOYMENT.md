# mylocalon — Production deployment (Ubuntu 24.04 + systemd)

Single host MVP: Caddy (public) → Platform `127.0.0.1:3001` + Gateway
`127.0.0.1:7002`, with local PostgreSQL and Redis. Files live in [`deploy/`](../deploy).

```
Internet ─TLS─▶ Caddy :443 ─┬─ mylocalon.com        → 127.0.0.1:3001  (platform)
                            └─ *.mylocalon.com      → 127.0.0.1:7002  (gateway)
gateway ─JWKS / internal API─▶ 127.0.0.1:3001     (never via Caddy)
platform ─▶ PostgreSQL 127.0.0.1:5432 · gateway ─▶ Redis 127.0.0.1:6379
```

## 0. Server

One VPS runs everything for the MVP.

| | Minimum | Recommended |
|---|---|---|
| OS | Ubuntu 24.04 LTS (x86_64) | same |
| CPU / RAM | 1 vCPU / 2 GB | **2 vCPU / 4 GB** |
| Disk | 25 GB SSD | 50 GB SSD |
| Network | public static IPv4, ports 22/80/443 open | + IPv6, ≥2 TB/month transfer |

All tunnel traffic flows through this box, so **bandwidth** is the cost to
watch. Pick a region near your users (India: DigitalOcean BLR1, AWS Lightsail
ap-south-1). Firewall: allow only 22, 80, 443 — Postgres, Redis, the Platform
and the Gateway all bind to 127.0.0.1.

## 1. DNS (Cloudflare)

Move `mylocalon.com` to Cloudflare (free plan) and add:

| Record            | Type | Value     | Proxy |
|-------------------|------|-----------|-------|
| `mylocalon.com`   | A    | server IP | DNS only (grey) |
| `*.mylocalon.com` | A    | server IP | DNS only (grey) |

Create an API token with **Zone → DNS → Edit** for `mylocalon.com`. Caddy
uses it for the wildcard certificate (ACME DNS-01).

## 2. Host setup (once)

```bash
rsync -a deploy/ root@SERVER:/root/mylocalon-deploy/
ssh root@SERVER /root/mylocalon-deploy/scripts/install.sh
```

Installs Node 22, PostgreSQL, Redis and Caddy (+ Cloudflare DNS module),
creates the `mylocalon` system user, binds Redis to localhost with a password,
creates the DB role, **generates every secret** into
`/etc/mylocalon/{gateway,platform}.env` (0640) and the Ed25519 signing key into
`/etc/mylocalon/credentials/jwt.key` (0600, root only — handed to the Platform
via systemd `LoadCredential`), installs the Caddyfile and units, and enables
the daily backup timer. Re-running it is safe.

Then put the Cloudflare token and your ACME email in `/etc/caddy/caddy.env`.

## 3. Build & deploy

```bash
# gateway: cross-compile on your machine, copy the binary
cd mylocalon-gateway && make build-linux VERSION=$(git describe --tags --always)
scp bin/gateway-linux root@SERVER:/opt/mylocalon/gateway/bin/gateway

# platform: copy source, build on the server (argon2 is a native module)
rsync -a --delete --exclude /node_modules --exclude /.next --exclude /.env --exclude /keys \
  mylocalon-platform/ root@SERVER:/opt/mylocalon/platform/
ssh root@SERVER 'chown -R mylocalon:mylocalon /opt/mylocalon && \
  cd /opt/mylocalon/platform && sudo -u mylocalon -H npm ci && sudo -u mylocalon -H npm run build && \
  sudo -u mylocalon -H npm prune --omit=dev'   # drop build/test-only tools

ssh root@SERVER 'systemctl enable --now mylocalon-platform mylocalon-gateway && systemctl restart caddy'
```

Migrations run automatically (`ExecStartPre`) before the Platform starts.

## 3b. Ship the CLI to users

Users install with `npm i -g mylocalon` and run `mylocalon login`. Defaults
already target `https://mylocalon.com` and `wss://gateway.mylocalon.com/tunnel`.

```bash
cd mylocalon-sdk-node && npm publish --access public   # @mylocalon/sdk-node
cd mylocalon-cli
npm pkg set dependencies.@mylocalon/sdk-node="^0.1.0"  # replace the file: link
npm install && npm run build && npm publish
```

## 4. Verify

```bash
curl -fsS https://mylocalon.com/health
curl -fsS http://127.0.0.1:3001/ready      # postgres
curl -fsS http://127.0.0.1:7003/ready      # redis
curl -fsS https://mylocalon.com/api/v1/jwks
journalctl -u mylocalon-gateway -o cat | grep jwks_refreshed
```

## 5. Operations

**Deploying a new gateway** — `systemctl restart mylocalon-gateway`. SIGTERM
sends `GOAWAY{reconnect:true}`; runtime state stays in Redis, so clients
resume with the **same URL** within the 30s grace window.

**Logs** — JSON on stdout → journald: `journalctl -u mylocalon-gateway -o cat | jq`.
Follow one request with its `reqId`, one tunnel with its `tunnelId`.

**Metrics** — `127.0.0.1:7003/metrics` (gateway) and `127.0.0.1:3001/metrics`
(platform); Caddy returns 404 for `/metrics` and `/api/v1/internal/*`.

**Abuse takedown** — add the subdomain to `ML_BLOCKED_SUBDOMAINS`, restart the gateway.

## 6. Backups

`mylocalon-backup.timer` runs `deploy/scripts/backup.sh` daily: `pg_dump`
custom format, integrity-checked with `pg_restore --list`, 14-day retention in
`/var/backups/mylocalon`. Copy that directory off-host (e.g. restic/rclone).
Redis is deliberately not backed up — it only holds disposable runtime state.

Restore drill (into a side database, never over the live one):

```bash
sudo -u postgres deploy/scripts/restore.sh /var/backups/mylocalon/<file>.dump mylocalon_restore
```

## 7. Secret rotation (SECURITY.md §8)

**JWT signing key — zero downtime:**
1. `openssl genpkey -algorithm ed25519 -out /etc/mylocalon/credentials/jwt-k2.key`
2. Export the old public key: `openssl pkey -in /etc/mylocalon/credentials/jwt.key -pubout -out /etc/mylocalon/jwt-k1.pub`
3. In `platform.env`: `JWT_KID=k2`, `JWT_EXTRA_PUBLIC_KEYS=k1=/etc/mylocalon/jwt-k1.pub`;
   point `LoadCredential=jwt.key:` at `jwt-k2.key`; restart the platform.
   JWKS now serves k1+k2; new tokens use k2. The gateway refreshes its JWKS
   every 5 min and immediately on an unknown `kid` — no gateway restart.
4. After `JWT_TTL_SECONDS` (1h), remove `JWT_EXTRA_PUBLIC_KEYS` and restart.

**Internal token** — set the new value in both env files, restart platform then gateway.
**DB / Redis passwords** — change, update the env file, restart the service.
**API keys** — user-driven: revoke + create in the dashboard.

## 8. Checklist (SECURITY.md §10)

- [x] TLS everywhere; only Caddy listens publicly (services bind 127.0.0.1).
- [x] JWT verified offline; gateway holds only public keys (JWKS).
- [x] API keys argon2id-hashed, shown once, revocable.
- [x] Tokens never logged; CLI config 0600.
- [x] Body/header/message caps at the edge.
- [x] Per-IP + per-token rate limits; per-user tunnel cap.
- [x] Input validation at every boundary (zod / strict protocol decode).
- [x] Secrets in root-owned env files / systemd credentials; rotation above.
