# Why a relay, not a server-side CRDT

Yjs is a CRDT: every client holds the full document and merges peer updates
locally. A collaboration *server* has two options — reimplement the CRDT and hold
authoritative state, or **relay** the opaque updates between clients and let them
converge on their own. `yrelay` takes the second path.

## What a relay does

- Track **rooms**, one per collaborative document.
- When a client joins, broadcast its incoming binary frames to every **other**
  client in the same room.
- When a client emits a `sync-step-1` handshake, the existing clients reply with
  their state — the relay does not intercept it, it just fans it out.
- When the last client leaves, GC the room (the relay holds no durable state).

## Why relay-only wins

- **Forward-compatible for free.** The y-protocols wire format is binary and
  versioned. Because the relay never parses an update, a new protocol version
  works the day clients ship it — no server change.
- **Small.** Server-side authoritative state needs a full Go Yjs implementation,
  a several-thousand-line project on its own. The relay is a few hundred lines of
  stdlib Go.
- **The upstream default.** Relay-only is exactly how the reference y-websocket
  server is deployed; persistence is an opt-in layer on top.

> **The server shuttles frames; the clients own the CRDT.**

## Where persistence goes

The relay is deliberately stateless: when a room empties, its frames are gone.
Durable documents are a **downstream** concern — a persistence layer subscribes
like any other member and serialises the last update from any one client to a
backing store, then replays it to the first client that rejoins. That layer lives
above the hub, not inside it, so the relay stays forward-compatible and small.
