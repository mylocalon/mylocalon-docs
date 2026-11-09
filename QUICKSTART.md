# mylocalon

Open-source developer tunnels: `localhost:3000 → https://abc123.mylocalon.com`.
Design docs: [docs/](README.md) · API: [docs/API.md](API.md) ·
Deploy: [docs/DEPLOYMENT.md](DEPLOYMENT.md)

| Folder               | What                                         |
|----------------------|----------------------------------------------|
| `mylocalon-gateway/`  | Go tunnel gateway (networking only)          |
| `mylocalon-platform/` | Next.js: accounts, API keys, subdomains, JWT issuing, dashboard |
| `mylocalon-cli/`      | `mylocalon` CLI                               |
| `mylocalon-sdk-node/` | Node SDK — reference protocol implementation |
| `docs/`              | design docs + cross-language protocol fixtures |
| `deploy/`            | Caddy, systemd units, install/backup scripts; `deploy/dev/` = local Postgres + Redis |

## Get the code

Each part is its own repo under [github.com/mylocalon](https://github.com/mylocalon).
Clone them side by side — tests expect the docs repo in a folder named `docs`:

```bash
mkdir mylocalon && cd mylocalon
for r in gateway platform cli sdk-node; do git clone https://github.com/mylocalon/mylocalon-$r.git; done
git clone https://github.com/mylocalon/mylocalon-docs.git docs
git clone https://github.com/mylocalon/mylocalon-deploy.git deploy
```

## Run everything locally

Needs Go, Docker, and **Node 22** for the Platform (`nvm use 22`).

```bash
docker compose -f deploy/dev/docker-compose.yml up -d

# 1. platform (dashboard + API) on :3001
cd mylocalon-platform
cp .env.example .env            # set INTERNAL_API_TOKEN: openssl rand -hex 32
npm i && npm run keys:gen && npm run db:migrate && npm run dev

# 2. gateway on :7002 — trusts the platform's JWKS
cd mylocalon-gateway
cp configs/gateway.example.env .env
#   set ML_JWKS_URL=http://127.0.0.1:3001/api/v1/jwks, ML_JWT_PUBLIC_KEYS=,
#   ML_PLATFORM_URL=http://127.0.0.1:3001, ML_INTERNAL_TOKEN=<same token>
make run

# 3. sdk + cli
cd mylocalon-sdk-node && npm i && npm run build
cd mylocalon-cli && npm i && npm run build
node dist/index.js login --platform http://localhost:3001 --gateway ws://localhost:7002/tunnel
#   → approve in the browser (sign up first)
python3 -m http.server 3000 &
node dist/index.js http 3000
curl http://<sub>.localtest.me:7002/        # *.localtest.me → 127.0.0.1
```

Without the Platform: leave the gateway on `ML_JWT_PUBLIC_KEYS=keys/ed25519.pub`,
run `make gen-keys`, and log in with `--token "$(make -s token)"`.

## Tests

```bash
cd mylocalon-gateway  && make test-redis   # unit + integration (+ Redis store contract)
cd mylocalon-platform && npm test          # API + services against Postgres (mylocalon_test)
cd mylocalon-sdk-node && npm test          # incl. SDK ↔ real Gateway e2e
cd mylocalon-cli      && npm test
```
