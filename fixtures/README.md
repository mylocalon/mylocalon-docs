# Protocol conformance fixtures

Canonical v1 messages from PROTOCOL.md. Every implementation (Gateway in Go,
each SDK) MUST decode every file here, and re-encoding MUST preserve all known
fields. `invalid/` holds messages that MUST be rejected with `PROTOCOL_ERROR`.
