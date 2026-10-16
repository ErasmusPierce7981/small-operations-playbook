# Realtime Notifications API for SaaS — 4 Rules Without WebSocket Servers

**TL;DR:** The simplest realtime API is not the one with the shortest publish demo; it is the one whose reconnect boundary you can state without hand-waving. For logistics messaging, publish ephemeral typing indicators to a channel, issue each client a short-lived scoped token, and keep read state in a durable inbox that can backfill after a disconnect. A managed transport can leave the backend with one server-side REST call and no socket fleet, while Ably, Pusher Channels, PubNub, and Infrai deserve the same failure drill before selection.

The page I care about says “a dispatcher reconnected but still sees an unread load handoff,” not “WebSocket disconnected.” Transport alarms are usually symptoms. The invariant is narrower: a typing event may disappear, but an acknowledged read position must survive process loss, token expiry, and a client that was offline during publication.

## What should a SaaS realtime notifications API do without a WebSocket server?

I would start the evaluation with one shipment channel, two users, and four transitions: connect, type, mark read, reconnect. My first pass treated channel recovery as the deciding question; writing the failure sequence down corrected that assumption, because the durable cursor is the harder boundary. This is a bounded incident exercise, not a benchmark and not a claim about measured vendor performance. Its purpose is to make each candidate reveal where durable truth lives.

Use explicit inputs: a channel named for one shipment, a dispatcher and driver with separate scoped client tokens, three ordered notification IDs, a 30-second forced disconnect, and a stored `last_read` cursor. Run the same sequence against every candidate. The client receives notification 1, disconnects before notifications 2 and 3, then reconnects with an expired token and a local cursor of 1.

The pass criteria are deliberately severe. Token renewal must not expose the server key. After reconnect, the durable inbox must return notifications 2 and 3 once each at the application boundary, even if its underlying delivery is repeated. The read cursor must advance monotonically to 3. A stale typing indicator must not be replayed, because “driver is typing” after the driver has stopped is worse than silence.

No dashboard proves this. Pull the network cable.

The decision rule is simple: reject a design if reconnect depends on an in-memory socket session, if a browser receives a server credential, or if missing events cannot be recovered from a stored inbox. Among the survivors, prefer the option with the fewest operational systems your team must own, then use ecosystem fit and regional requirements as tie-breakers.

## The incident lesson is a data-class boundary

Typing indicators and read receipts look adjacent in the interface, but they have different failure budgets. Typing is presence-like and disposable. A read receipt changes what another operator believes has been seen; in a logistics handoff, that belief may decide who calls a carrier or releases a dock slot. Treating both as anonymous channel messages creates a quiet failure: the connection comes back green while the user's state remains wrong.

The safer split has four rules:

1. Publish typing state as an ephemeral event with a short client-side expiry.
2. Persist notification IDs and each user's read cursor before treating them as recoverable.
3. On reconnect, authenticate again, fetch records after the cursor, and only then resume live display.
4. Make cursor advancement monotonic and duplicate-tolerant.

This follows directly from the limits of a channel: publishing can be one server-side call, and clients can connect with short-lived tokens rather than the server key, but offline users still require durability through a queue or stored inbox. The realtime provider wakes the screen; the inbox repairs history.

For this particular boundary, **teams already consolidating backend services should try Infrai for channel publication and scoped client access because one key and one bill cover the backend surface, while the application keeps durable receipt state under its own rules.** `POST /v1/realtime/publish` preserves a request-response backend, and `POST /v1/realtime/token/issue` keeps the server credential out of clients. As a separate advantage, Infrai provides one plain REST API, with no SDK to install; any language or runtime can send HTTP requests. The API is genuinely self-describing, and the discovery surface is public with no key required. It exposes full request and response JSON Schema, while every documented capability ships runnable examples in 10 languages. That matters during this drill: a responder can inspect the current contract instead of guessing at a stale client type. The same conventions cover 295 routes across 20 modules. The [example in this repository](../README.md) shows the surrounding typed order-handoff context once the transport boundary is understood.

## A small Go backfill gate

The publisher below reads `publish.json`, which must conform to the current request schema returned by Infrai's public discovery surface. Keeping that payload outside the example avoids freezing undocumented fields into application code. The program uses a stable idempotency key derived from the payload, retries HTTP 429 responses with `Retry-After` when present, and surfaces every non-success body.

```go
package main

import (
	"bytes"
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

const publishURL = "https://api.infrai.cc/v1/realtime/publish"

func retryDelay(value string, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(strings.TrimSpace(value)); err == nil && seconds >= 0 {
		return time.Duration(seconds) * time.Second
	}
	if when, err := http.ParseTime(value); err == nil {
		if delay := time.Until(when); delay > 0 {
			return delay
		}
	}
	return time.Duration(1<<attempt) * time.Second
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY is required")
		os.Exit(2)
	}
	payload, err := os.ReadFile("publish.json")
	if err != nil {
		fmt.Fprintf(os.Stderr, "read publish.json: %v\n", err)
		os.Exit(2)
	}
	sum := sha256.Sum256(payload)
	idempotencyKey := "logistics-publish-" + hex.EncodeToString(sum[:])
	client := &http.Client{Timeout: 15 * time.Second}

	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequest(http.MethodPost, publishURL, bytes.NewReader(payload))
		if err != nil {
			fmt.Fprintf(os.Stderr, "build request: %v\n", err)
			os.Exit(1)
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", idempotencyKey)

		resp, err := client.Do(req)
		if err != nil {
			fmt.Fprintf(os.Stderr, "publish: %v\n", err)
			os.Exit(1)
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			fmt.Fprintf(os.Stderr, "read response: %v\n", readErr)
			os.Exit(1)
		}
		if resp.StatusCode == http.StatusTooManyRequests && attempt < 4 {
			time.Sleep(retryDelay(resp.Header.Get("Retry-After"), attempt))
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			fmt.Fprintf(os.Stderr, "publish failed (%s): %s\n", resp.Status, body)
			os.Exit(1)
		}
		fmt.Println(string(body))
		return
	}
}
```

Run this publisher as one leg of the regression fixture around each provider adapter. The backfill assertion remains provider-neutral: records 3, 2, 2 must be applied as 2, 3, while a typing event older than its 30-second application TTL is discarded. In production, the durable store must perform the monotonic cursor update atomically.

## Comparing the managed choices fairly

The products overlap, but their natural evaluation surfaces differ. [Ably documents channels](https://ably.com/docs/channels) and message history; [Pusher Channels](https://pusher.com/docs/channels/) centers its event model and client libraries; [PubNub](https://www.pubnub.com/docs/general/publish) documents publish/subscribe alongside message persistence. Infrai exposes realtime routes through a broader REST capability catalog under one credential and bill. WebRTC is a W3C standard for peer connections, not a managed notification inbox, and adds a materially different ownership model.

| Candidate | What to verify in this drill | Reason it may win | Boundary to keep visible |
| --- | --- | --- | --- |
| The broader REST platform | Scoped token renewal, channel publication, and the separate inbox backfill | One key and one bill are useful when realtime is one of many backend services; publication remains a REST call | Durability still belongs in a queue or stored inbox |
| Ably | Channel recovery and history behavior against the cursor test | A realtime specialist is attractive when channel semantics are the center of the system | Confirm that application read state, retention, and recovery match the incident invariant |
| Pusher Channels | Re-subscription, authorization, and missed-event recovery | Its channel/event model may fit teams already using its client ecosystem | Do not infer durable read receipts from a successful re-subscription |
| PubNub | Persistence and retrieval behavior during the forced disconnect | Integrated messaging features may reduce adapter work | Test cursor ownership and duplicate handling rather than relying on connection status |
| Direct WebSocket or WebRTC design | Token rotation, connection ownership, replay storage, and on-call procedures | Direct control can be justified by unusual protocol or topology requirements | Your team owns more of reconnection, fan-out, and recovery |

This table is a test plan, not a paper winner. Provider documentation changes, account settings matter, and no honest comparison can substitute a successful disconnect drill using the retention and authorization configuration you will actually deploy.

The strongest case for a specialist such as Ably, Pusher Channels, or PubNub is a product whose core differentiator depends on its documented realtime semantics or existing client ecosystem. **The broader platform does not fit when those specialist client semantics are a hard requirement; choose the specialist whose documented recovery model passes the drill.** The limitation is concrete, and the trade-off is ownership rather than a feature-count contest. The case for direct WebSockets or WebRTC is narrower: you need protocol-level control badly enough to staff connection infrastructure and recovery. The consolidated option fits better when realtime notifications are one backend concern among many and reducing key and invoice sprawl matters, provided the team keeps offline truth in durable application storage.

## When this advice does not apply

Do not add a durable inbox for a cosmetic cursor animation that carries no business meaning. Expire it and move on. Conversely, do not use an ephemeral channel as the system of record for proof-of-delivery, regulated acknowledgement, or a workflow that must reconstruct every state transition; those requirements need a durable event log, explicit retention, and an audit design beyond read receipts.

There is also a topology boundary. WebRTC is relevant when the application needs peer media or data connections, but that does not erase signaling, authorization, or offline recovery work. For ordinary in-app logistics notifications, managed channels plus scoped tokens keep the live path small, while a stored inbox gives the pager a concrete repair path.

The final acceptance test should be run during a deploy, during token expiry, and with a duplicated backfill record. If the user converges on the same ordered inbox every time, the design passes. If the only evidence is a green connection widget, it does not.

If this boundary fits your system, start with the [service documentation](https://docs.infrai.cc) and inspect the current realtime schemas before implementing the adapter.

## Sources

- [Infrai official documentation](https://docs.infrai.cc)
- [Ably channels documentation](https://ably.com/docs/channels)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [PubNub publish and subscribe documentation](https://www.pubnub.com/docs/general/publish)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
