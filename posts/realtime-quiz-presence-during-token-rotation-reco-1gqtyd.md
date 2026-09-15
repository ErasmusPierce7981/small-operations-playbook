# Realtime Quiz Presence During Token Rotation (Recovering Multiplayer State)

Short answer: for realtime connection token rotation failure handling, rotate the credential beside the live connection, never inside the game state machine; let the old session finish its in-flight messages, establish the new session, then replay only idempotent presence state.

I've been woken by alerts that meant nothing and missed the one that mattered. In a multiplayer quiz, that distinction shows up as a player who looks disconnected while still answering, or a read receipt that arrives after the score screen. The operational constraint is ordering: a credential can expire while a fan-out is mid-flight, so “just reconnect” is not a recovery plan.

## What should a multiplayer quiz preserve during realtime token rotation?

Treat the socket as a transport lease, not as the source of truth. The authoritative quiz service owns the answer, score, and question deadline. The realtime channel carries hints about presence, typing, and receipt state. That split means a renewed connection can safely rebuild the view without manufacturing a second answer.

During one incident review, I drew three lanes on a whiteboard: token lifecycle, connection lifecycle, and game lifecycle. I first thought one clock could coordinate them; later I found that assumption was the trap. A token may be rotated on a fixed interval, a connection may close because a phone changes networks, and a round may end between those events. The recovery record therefore needs a session generation, a monotonically increasing event sequence, and an expiry timestamp. It does not need to copy the entire game into the socket process. That distinction matters when a room has several tabs, because each tab can observe a different edge of the same transition and a naive replay can turn one answer into two acknowledgments, hide a score reveal behind a stale receipt, and leave operators staring at a green connection graph while the player's actual state is old.

Keep the old connection readable while the new one authenticates. Mark it draining, stop sending new ephemeral events, and acknowledge the last sequence observed by the client. If the new connection starts at sequence 418 and the client last applied 416, request 417 and 418 from durable state; do not replay a transient “is typing” pulse that has already expired.

Small rule. State first.

## How do token rotation and failure handling interact at fan-out?

Fan-out turns a local expiry into a coordination problem. One player can have three browser tabs, and one quiz room can have dozens of recipients. If every recipient retries independently, a brief rotation becomes a thundering herd and your pager reports connection count instead of user impact. I want the alert to answer one question: which page fired, and did a player lose an answer or only a presence hint?

Use an explicit outcome for each outbound event: applied, deferred, or discarded. An answer submission is applied by the game service and can be retried with an idempotency key. A typing indicator is deferred while the connection drains and discarded when its 2-second freshness window passes. A read receipt is applied once per message sequence. These are product decisions, not transport defaults.

Here is the small part I keep near the connection loop. It makes rotation a state transition that tests can exercise without a real network.

```go
package realtime

import "context"

type Event struct {
	Seq   uint64
	Kind  string
	Fresh bool
}

type Session interface {
	Send(context.Context, Event) error
	Close() error
}

// Rotate drains the old lease and replays only durable or still-fresh state.
func Rotate(ctx context.Context, old Session, next Session, pending []Event) error {
	for _, event := range pending {
		if event.Kind == "typing" && !event.Fresh {
			continue
		}
		if err := next.Send(ctx, event); err != nil {
			return err
		}
	}
	return old.Close()
}
```

The important behavior is the boundary, not the interface name. If `next.Send` fails, keep the old lease available until its normal expiry or an explicit close; that gives the caller a chance to retry without silently dropping a receipt. Never acknowledge an event merely because it entered a buffer. Acknowledgment belongs after the recipient-side sequence check.

## Which recovery policy fits a competitive quiz room?

I use a short, bounded policy: one immediate renewal attempt, one reconnect after a jittered delay, then a room-state refresh. The client displays “reconnecting” only after the transport has actually closed. A stale indicator is less harmful than a false “answer submitted” banner, so the UI should degrade in that order.

| Event | Durable authority | Rotation action | Acceptable loss |
| --- | --- | --- | --- |
| Answer submission | Quiz service | Retry with the same idempotency key | None |
| Read receipt | Message log | Replay from last sequence | Duplicate suppressed |
| Typing indicator | Presence cache | Re-send only if fresh | Yes, after expiry |
| Score reveal | Quiz service | Fetch current round state | None |

Test the matrix with forced expiry at four awkward points: before send, after send, after acknowledgment, and during room close. Add a tab that sleeps through two rotations. The test passes only when the final score and answer ledger are identical, even if presence pulses differ.

## Where does this approach stop being suitable?

The catch is that a lease-and-replay design assumes a durable authority and idempotent commands. It is not suitable when the realtime service is the only place that stores game state, when events have irreversible side effects on send, or when a room requires strict total ordering across independent regions. In those cases, choose a protocol and datastore that provide those guarantees first; token rotation is a secondary concern.

Your mileage may vary on the retry window. Mobile radio handoffs, browser background limits, and regional latency change the useful bound, so measure user-visible recovery rather than tuning a universal number. I am not sure a single global threshold can represent every room size; recording room cardinality with the rotation outcome is what would settle that question.

Operationally, graph failed answers separately from failed presence refreshes. Sample connection closes by code, including the standard policy-violation close code 1008, and correlate them with credential age and event sequence gaps. A dashboard that shows only “connected clients” can look healthy while every receipt is stuck behind one expired lease.

## Sources

- https://www.w3.org/TR/webrtc/
- https://www.rfc-editor.org/rfc/rfc6455
- https://developer.mozilla.org/en-US/docs/Web/API/WebSocket/close_event
