# mylocalon — Documentation

mylocalon is an open-source developer tunneling platform. It exposes a local app
to the internet over a stable public URL:

```
localhost:3000  →  https://abc123.mylocalon.com
```

The MVP ships a Node.js **CLI** and **SDK**, but the core Gateway is
**language-independent** — any language can add an SDK by implementing the
[Tunnel Protocol](./PROTOCOL.md) with **zero Gateway changes**.

## Repositories (no monorepo)

Each component is its own repo, laid out as sibling folders under `mylocalon/`.

| Folder                     | Language          | Doc                              |
|----------------------------|-------------------|----------------------------------|
| `mylocalon-gateway`         | Go                | [GATEWAY.md](./GATEWAY.md)       |
| `mylocalon-platform`        | Next.js + TS      | [PLATFORM.md](./PLATFORM.md)     |
| `mylocalon-cli`             | TypeScript        | [CLI.md](./CLI.md)               |
| `mylocalon-sdk-node`        | TypeScript        | [SDK-NODE.md](./SDK-NODE.md)     |
| *future SDKs*              | Java, Python, …   | implement `PROTOCOL.md`          |

Start here: [QUICKSTART.md](./QUICKSTART.md) — clone, run locally, test.

## Documentation map

| Doc                                | Scope                                             |
|------------------------------------|---------------------------------------------------|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | System design, request/data flow, dependencies    |
| [PROTOCOL.md](./PROTOCOL.md)       | The WSS Tunnel Protocol v1 (the language seam)     |
| [GATEWAY.md](./GATEWAY.md)         | Go tunnel gateway — networking only                |
| [PLATFORM.md](./PLATFORM.md)       | Next.js platform — auth, projects, subdomains      |
| [CLI.md](./CLI.md)                 | `mylocalon` command-line client                     |
| [SDK-NODE.md](./SDK-NODE.md)       | Programmatic Node SDK                              |
| [API.md](./API.md)                 | Platform REST API reference + versioning           |
| [SECURITY.md](./SECURITY.md)       | Trust boundaries, tokens, limits, hardening        |
| [OBSERVABILITY.md](./OBSERVABILITY.md) | Logging, metrics, tracing, correlation ids     |
| [TESTING.md](./TESTING.md)         | Test strategy per component + local runbooks       |
| [DEPLOYMENT.md](./DEPLOYMENT.md)   | Caddy, systemd, Ubuntu, backups, secrets           |

## Build phases

1. **Protocol & architecture design** ← docs (this set)
2. Gateway core
3. CLI
4. Node SDK
5. Platform API
6. Dashboard
7. Full local integration testing
8. Production deployment (Ubuntu + systemd)
9. Scaling (only after MVP is stable)

Each feature follows: explain → why → architecture → request flow → data flow →
folder structure → interfaces → edge cases → implement → test locally → next.

## Core principles

- Clean architecture, SOLID, strong typing, testable, production quality.
- Gateway does **networking only** — no user management, no business logic.
- Redis = runtime state only. PostgreSQL = source of truth.
- Design for horizontal scale; do **not** add microservices/k8s/queues in MVP.
