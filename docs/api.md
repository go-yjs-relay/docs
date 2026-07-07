# Usage & API

```sh
go get github.com/go-yjs-relay/yrelay
```

## The three types

```go
// Hub multiplexes connections by room ID. One Hub per server instance.
func NewHub() *Hub
func (h *Hub) Join(roomID string, conn ConnID) *Membership
func (h *Hub) MembersCount(roomID string) int   // live peer count (0 if no room)

// ConnID is the opaque connection identifier the caller passes in
// (typically the remote address). Used only for logging / tie-breaking.
type ConnID string

// Membership is one client's view into a Room. Returned by Hub.Join.
func (m *Membership) Recv() <-chan []byte              // peer frames; closed on leave
func (m *Membership) Send(payload []byte)              // fan out to the rest of the room
func (m *Membership) Leave()                           // unregister; idempotent
func (m *Membership) LeaveOnContextDone(ctx context.Context)
```

`Room` is created and reclaimed internally by the `Hub`; callers only ever hold a
`*Membership`.

## Minimal relay

```go
hub := yrelay.NewHub()

alice := hub.Join("doc-42", "alice")
bob := hub.Join("doc-42", "bob")
defer alice.Leave()
defer bob.Leave()

alice.Send([]byte("y-update"))
frame := <-bob.Recv() // "y-update"; alice never receives her own frame
```

## Wiring a real WebSocket

A `Membership` is driven by two goroutines — a **reader** that pumps inbound
socket frames into the room, and a **writer** that ranges over `Recv()` and
writes peer frames back out. `LeaveOnContextDone` ties the membership to the
request context so it unregisters when the connection ends.

```go
func serve(hub *yrelay.Hub, roomID string, conn Socket) {
	m := hub.Join(roomID, yrelay.ConnID(conn.RemoteAddr()))
	m.LeaveOnContextDone(conn.Context()) // auto-leave when the request ends
	defer m.Leave()

	// writer: peer frames -> this socket
	go func() {
		for frame := range m.Recv() {
			_ = conn.WriteBinary(frame)
		}
	}()

	// reader: this socket -> the room
	for {
		frame, err := conn.ReadBinary()
		if err != nil {
			return // deferred Leave (and the context watcher) clean up
		}
		m.Send(frame)
	}
}
```

`Socket` here stands in for whatever WebSocket type you use — `yrelay` imports no
WebSocket library, so `Recv()` / `Send()` are the only seam a transport needs.

## Observability

`Hub.MembersCount(roomID)` returns the number of connections currently in a room
(0 when the room does not exist). The count **is** the bidirectional-sync
precondition: at least two members must be present for any cross-client frame to
flow, so it is a useful signal to surface in logs or metrics.
