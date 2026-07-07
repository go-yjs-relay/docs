# Roadmap

## Done — relay hub complete

- **Rooms & Hub multiplexing** — one `Room` per document, `Hub.Join` →
  `Membership`, `Hub.MembersCount` for observability.
- **Fan-out broadcast** — echo-suppressed delivery to every other member.
- **Slow-peer drop** — non-blocking sends; a full buffer drops the frame instead
  of stalling the hub.
- **Room GC** — a room is reclaimed when its last member leaves.
- **Idempotent `Leave`** and **`LeaveOnContextDone`** — safe, context-driven
  connection teardown.
- **Transport-agnostic core** — no WebSocket library imported; the `Membership`
  handle is the seam.
- **Quality gate** — 100% test coverage under `-race`, `gofmt` + `go vet` clean,
  CI green on all six 64-bit Go arches.

## Planned — downstream by design

- **Persistence & snapshotting.** The relay holds no durable state. Persistence is
  an opt-in layer that subscribes like any other member and serialises the last
  update to a backing store, replaying it to the first client that rejoins. It
  lives above the hub so the relay stays forward-compatible and small.
- **Awareness / presence surfacing.** `ConnID` already carries a per-connection
  identity for tie-breaking; a thin presence view over the member set could be
  exposed for cursors / user lists.

The rendering and CRDT merge logic stay on the clients — that is the relay's whole
premise, not a gap.
