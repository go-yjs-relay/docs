# Lifecycle & concurrency

`yrelay` is pure concurrent Go — one goroutine per connection reads the socket and
`Send`s; the fan-out runs under a per-room lock. The whole package is stdlib
`context` + `sync`, tested under the race detector at 100% coverage.

## Fan-out broadcast (echo-suppressed)

`Membership.Send` pushes the frame to every member of the room **except the
sender**, so a client never receives its own frame. Each member has a buffered
receive channel; the fan-out iterates the room's member set under the room lock,
which keeps a concurrent `Leave` from closing a receive channel mid-send.

## Slow-peer drop

Sends are **non-blocking**. If a member's receive buffer is full — a slow or
stalled peer — the frame is **dropped** rather than blocking the hub:

```go
select {
case m.recv <- payload:
default:
	// slow peer; drop. Yjs re-syncs it via the next sync-step handshake.
}
```

One straggler can never stall the room. The dropped client recovers on its own
through the protocol's state-vector exchange on the next message. The critical
section stays microseconds even at hundreds of members.

## Room GC

A `Room` is created on the first `Join` for its ID and **reclaimed when its last
member leaves**. `MembersCount` therefore reports 0 both for a room that never
existed and for one whose members have all left — the room object is gone either
way.

## Idempotent leave

`Membership.Leave` is safe to call more than once — a second call is a no-op. That
matters because a membership is typically left from two places at once: a deferred
cleanup in the request handler and a context watcher. The first call unregisters
the membership and closes `Recv`; the rest short-circuit.

## Context-driven leave

`LeaveOnContextDone(ctx)` spawns a tiny watcher goroutine that leaves the
membership when `ctx` cancels — saving the caller from writing that goroutine. If
the membership leaves on its own first, the watcher observes the closed membership
and exits cleanly, so it never leaks.

```go
m := hub.Join(roomID, conn)
m.LeaveOnContextDone(ctx) // leaves automatically when ctx cancels
```
