# mylocalon-cli (TypeScript)

Thin shell over `@mylocalon/sdk-node`: credentials, config and terminal output.

```
mylocalon login [--platform <url>] [--gateway <wss-url>] [--no-browser]
mylocalon login --api-key <key> | --token <jwt>
mylocalon http <port|host:port|url> [--subdomain <name>] [--json]
mylocalon status | logout | version
```

- **Config:** `~/.mylocalon/config.json`, dir 0700 / file 0600. The CLI refuses
  to read it if group/world bits are set. `MYLOCALON_CONFIG_DIR` overrides.
- **Login:** default is the browser device flow (API.md "Device login"): the
  CLI prints a code, opens the approval page, polls, and saves the issued API
  key. `--api-key` stores an existing key; `--token` a raw dev JWT.
- **Auth:** with an API key, every (re)connect exchanges it at
  `POST {platform}/api/v1/tokens` for a fresh short-lived JWT (via the SDK's
  `tokenProvider`), so new reservations apply on the next connect and a
  revoked key stops the CLI instead of retrying. `MYLOCALON_TOKEN` overrides.
- **Output:** human view (URL banner, one line per request), or `--json`
  (one JSON event per line) for scripting. Secrets are only ever shown redacted.
- **Exit:** Ctrl+C sends CLOSE and exits 0; a second Ctrl+C forces exit.
  Fatal GOAWAYs exit 1 with a hint (e.g. `TUNNEL_LIMIT`, `SUBDOMAIN_FORBIDDEN`).
