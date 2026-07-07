# go-yjs-relay documentation

**A pure-Go (no cgo) Yjs y-websocket relay hub** — it multiplexes
collaborative-editing connections into rooms and fans each client's binary
frames out to the rest of the room. The module path is
`github.com/go-yjs-relay/yrelay`; the package is `yrelay`.

It is **transport-agnostic**: it imports no WebSocket library. The caller owns
the socket and pushes/pulls binary frames through a `Membership` handle, so
`yrelay` drops into any Go WebSocket stack (`nhooyr.io/websocket`,
`gorilla/websocket`, `net/http`'s own upgrader, or an in-memory pipe in tests).

!!! success "Status: relay hub complete"
    `Hub` / `Room` / `Membership` with room multiplexing, echo-suppressed
    **fan-out broadcast**, non-blocking **slow-peer drop**, automatic **room GC**
    on the last leave, **idempotent** `Leave`, and **context-driven**
    `LeaveOnContextDone`. Pure stdlib (`context` + `sync`), CGO-free, **100% test
    coverage under `-race`**, `gofmt` + `go vet` clean, CI green across the six
    64-bit Go targets (amd64, arm64, riscv64, loong64, ppc64le, s390x).

## A relay, not a server-side CRDT

`yrelay` does **not** decode Yjs updates server-side. The clients reconcile among
themselves via Yjs's update + sync-step algebra; the server only shuttles opaque
binary frames.

> **The server shuttles frames; the clients own the CRDT.**

That split is what makes the relay **forward-compatible** with new, versioned
y-protocols messages for free — and a few hundred lines instead of a
several-thousand-line Go Yjs implementation. It is also the relay-only deployment
mode the upstream y-websocket server uses; persistence is bolted on downstream by
serialising the last update from any one client to a backing store.

## Quick taste

```go
hub := yrelay.NewHub()

alice := hub.Join("doc-42", "alice")
bob := hub.Join("doc-42", "bob")
defer alice.Leave()
defer bob.Leave()

alice.Send([]byte("y-update")) // fan out to the rest of the room
frame := <-bob.Recv()          // bob reads it; alice never sees her own frame
```

## Repositories

| Repo | What it is |
| --- | --- |
| [`yrelay`](https://github.com/go-yjs-relay/yrelay) | the relay hub — `Hub` / `Room` / `Membership`, fan-out, slow-peer drop, room GC, context-driven leave |
| [`docs`](https://github.com/go-yjs-relay/docs) | this documentation site (MkDocs Material, versioned with mike) |
| [`go-yjs-relay.github.io`](https://github.com/go-yjs-relay/go-yjs-relay.github.io) | the organization landing page (Hugo) |
| [`brand`](https://github.com/go-yjs-relay/brand) | logo and brand assets |

## Principles

- **Pure Go, `CGO_ENABLED=0`** — trivial cross-compilation, a single static
  binary, no C toolchain.
- **A relay, not a CRDT** — the server is forward-compatible with new y-protocols
  versions because it never parses them.
- **Transport-agnostic** — no WebSocket library baked in; the `Membership` handle
  is the seam.
- **Robust under load** — slow peers are dropped, not blocking; empty rooms are
  GC'd; `Leave` is idempotent and context-driven.
- **100% test coverage** under `-race` is the target, enforced as a CI gate.

## Where to go next

- [Why a relay, not a CRDT](why.md) — the design split and what it buys.
- [Usage & API](api.md) — `Hub`, `Room`, `Membership`, and wiring a real socket.
- [Lifecycle & concurrency](lifecycle.md) — fan-out, slow-peer drop, room GC,
  idempotent and context-driven leave.
- [Roadmap](roadmap.md) — what is done and what is downstream by design.

Source lives at
[github.com/go-yjs-relay/yrelay](https://github.com/go-yjs-relay/yrelay).
