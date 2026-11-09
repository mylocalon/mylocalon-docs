# mylocalon-sdk-node (TypeScript)

Programmatic tunnel creation for Node.js. This SDK is the **reference
implementation** of the [Tunnel Protocol](./PROTOCOL.md); every future SDK
(Python, Go, Rust, Java) mirrors its behavior. The CLI is built on top of it.

## 1. Responsibilities

- Open a WSS connection to the Gateway and run the protocol handshake.
- Forward `HTTP_REQUEST` → local target → `HTTP_RESPONSE`.
- Heartbeat (PONG replies), detect dead sockets.
- Auto-reconnect with resume (stable URL); expose lifecycle events.
- Enforce client-side guards (body size hint, target validation).
- Never manage users/auth — it consumes a JWT the caller supplies.

## 2. Public API (surface)

```ts
import { Tunnel } from "@mylocalon/sdk-node";

const tunnel = new Tunnel({
  gatewayUrl: "wss://gateway.mylocalon.com/tunnel",
  token: jwt,                          // or tokenProvider() for refresh
  target: "http://localhost:3000",
  subdomain: "myapp",                  // optional reserved subdomain
  reconnect: true,                     // default true
});

tunnel.on("active",       (info) => console.log(info.url));   // WELCOME
tunnel.on("request",      (r)    => {});                       // observability
tunnel.on("disconnected", ()     => {});
tunnel.on("reconnecting", (n)    => {});
tunnel.on("closed",       (why)  => {});
tunnel.on("error",        (e)    => {});

await tunnel.connect();                // resolves on first ACTIVE
console.log(tunnel.url);               // https://myapp.mylocalon.com
await tunnel.close();                  // graceful CLOSE
```

- `token` may be a string or `tokenProvider: () => Promise<string>` so
  long-running processes can refresh an expiring JWT transparently.
- The SDK is transport-only: give it a JWT; it does not know about API keys.

## 3. Internal structure

```
mylocalon-sdk-node/
├── src/
│   ├── tunnel.ts            # public Tunnel class (EventEmitter), lifecycle
│   ├── protocol/
│   │   ├── messages.ts      # typed message unions (HELLO, WELCOME, ...)
│   │   ├── codec.ts         # encode/decode + validation (matches PROTOCOL.md)
│   │   └── version.ts       # PROTOCOL_VERSION = 1
│   ├── connection.ts        # ws wrapper: connect, read/write pump, ping
│   ├── forwarder.ts         # HTTP_REQUEST → local fetch → HTTP_RESPONSE
│   ├── reconnect.ts         # backoff + resume-token handling
│   ├── state.ts             # client-side state machine (PROTOCOL §11)
│   └── lib/                 # logger, errors, base64, config
├── package.json             # strict TS, ESM + CJS build
└── tsconfig.json
```

## 4. Forwarding (the core loop)

```
onMessage(HTTP_REQUEST):
  decode headers[], base64 body
  fetch(target + path, { method, headers, body })
    ok  → HTTP_RESPONSE{ reqId, status, headers[], bodyB64, durationMs }
    err → ERROR{ reqId, code: UPSTREAM_UNREACHABLE | UPSTREAM_TIMEOUT }
  write result to WS (via write pump — single writer)
```

- Uses Node `fetch`/`undici`. Per-request timeout (default 30s) → `UPSTREAM_TIMEOUT`.
- Strips hop-by-hop headers; preserves duplicates via the pairs array.
- Response body is buffered then base64'd (v1); large-body streaming is the
  future negotiated capability (PROTOCOL §12).

## 5. State machine (client side)

Mirrors PROTOCOL §11: `CREATED → CONNECTING → ACTIVE → (DISCONNECTED →
RECONNECTING → ACTIVE)* → CLOSED`. Events are emitted on each transition so the
CLI/embedders can render status. Resume token + tunnelId are held in memory.

## 6. Interfaces (testable seams)

```ts
interface Connection {                 // fake in tests, real = ws
  send(msg: ClientMessage): void;
  onMessage(cb: (m: ServerMessage) => void): void;
  onClose(cb: (code: number) => void): void;
  close(): Promise<void>;
}
interface LocalForwarder {             // fake target in tests
  forward(req: HttpRequestMsg): Promise<HttpResponseMsg | ErrorMsg>;
}
```

## 7. Edge cases

| Case                              | Handling                                     |
|-----------------------------------|----------------------------------------------|
| Target unreachable                | `UPSTREAM_UNREACHABLE` per request; tunnel stays up |
| Target slow                       | per-request timeout → `UPSTREAM_TIMEOUT` (504)|
| Socket drop                       | resume within grace → same URL; else new URL  |
| JWT expired on reconnect          | call `tokenProvider()` for a fresh JWT        |
| Backpressure (many concurrent)    | bounded concurrency; write pump serializes    |
| `close()` while requests in flight| stop accepting, drain briefly, send CLOSE     |

## 8. Guarantees for future SDKs (parity checklist)

Any language SDK MUST reproduce: handshake + version const, header **pairs**
preservation, base64 bodies, PONG on PING, resume with `resumeToken`, jittered
backoff, and the same `ERROR` codes for upstream failures. This doc + PROTOCOL.md
is the conformance spec; a shared protocol test vector set (JSON fixtures) will
back it — see [TESTING.md](./TESTING.md).

## 9. Testing

Unit: codec round-trips against PROTOCOL fixtures, forwarder against a local
stub server, reconnect/backoff with a fake clock. Integration: SDK against the
real Gateway on localhost.
