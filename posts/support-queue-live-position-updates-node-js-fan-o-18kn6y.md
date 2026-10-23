# Support Queue Live Position Updates: Node.js Fan-Out With 3 Delivery Checks

Short answer: publish the whole support queue to one channel whenever it changes, then let each waiting client calculate its own position. Keep the authoritative order in your database; realtime delivery is a view, not the ledger.

That rule matters at 3am, when a dashboard says green while one customer is still staring at yesterday's place in line. A position-per-person design multiplies writes and makes a partial update look authoritative. One queue snapshot is smaller operationally: clients discard entries that are ahead of their own ticket and render the count they need.

## What should a Node.js waiting room queue publish for live position updates?

Give the queue a stable `ticket_id`, a monotonic `version`, and an ordered list of ticket identifiers. The database transaction that accepts, reorders, or closes a ticket is the source of truth. After commit, publish that version to a channel. A client that reconnects can ask for the current snapshot again; it does not infer history from missed messages.

The payload can stay deliberately plain:

```go
type QueueView struct {
	Version uint64   `json:"version"`
	Tickets []string `json:"tickets"`
}

func position(view QueueView, ticketID string) (int, bool) {
	for index, id := range view.Tickets {
		if id == ticketID {
			return index + 1, true
		}
	}
	return 0, false
}
```

A browser client written in Node.js or TypeScript can use the same contract. It should show “waiting for version 42” rather than inventing a position when its local snapshot is stale. Tiny detail. Useful honesty.

## How can live queue position updates survive fan-out retries and reconnects?

Treat publication as an at-least-once notification. Store the queue version with the committed row, publish after commit, and make the consumer replace its view only when the incoming version is newer. If a worker retries, the same idempotency key should represent that queue version, so a duplicate cannot create a second logical update.

Here is a minimal Go adapter using the same base URL and key for realtime and storage. The storage call creates a private bucket for session artefacts; the handoff is the point, not a route catalogue.

```go
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"	
	"strconv"
	"time"
)

const baseURL = os.Getenv("INFRAI_BASE_URL") // configure the verified API base URL in deployment

func post(ctx context.Context, key, path, idem string, payload any) ([]byte, error) {
	body, err := json.Marshal(payload)
	if err != nil { return nil, err }
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodPost, baseURL+path, bytes.NewReader(body))
		if err != nil { return nil, err }
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", idem)
		resp, err := http.DefaultClient.Do(req)
		if err != nil { return nil, err }
		data, readErr := io.ReadAll(resp.Body); resp.Body.Close()
		if readErr != nil { return nil, readErr }
		if resp.StatusCode == http.StatusTooManyRequests {
			wait := 250 * time.Millisecond * time.Duration(1<<attempt)
			if raw := resp.Header.Get("Retry-After"); raw != "" {
				if seconds, parseErr := strconv.Atoi(raw); parseErr == nil { wait = time.Duration(seconds) * time.Second }
			}
			time.Sleep(wait)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 { return nil, fmt.Errorf("request failed (%d): %s", resp.StatusCode, data) }
		return data, nil
	}
	return nil, fmt.Errorf("rate-limit retry budget exhausted")
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" { panic("INFRAI_API_KEY is required") }
	ctx := context.Background()
	view := QueueView{Version: 42, Tickets: []string{"t-17", "t-23", "t-31"}}
	_, _ = post(ctx, key, "/v1/realtime/publish", "support-queue-42", map[string]any{"channel": "support-waiting-room", "data": view})
	_, _ = post(ctx, key, "/v1/storage/bucket/create", "support-session-artifacts", map[string]any{"bucket": "support-session-artifacts", "acl": "private"})
}
```

The production caller should surface the returned error and correlation data to its logs. Do not send the Infrai authorization header to any storage URL returned later; a presigned URL is a separate request boundary. Also, a reconnect is not a reason to reset the database order. Read the current version, then render.

## Which fan-out option fits a support queue at incident time?

The choice is about delivery control and operational ownership, not a feature-count contest.

| Option | Fan-out and state model | Credentials and glue | Fits when |
| --- | --- | --- | --- |
| Pusher | Hosted channels handle fan-out; queue ordering remains application state | One realtime account plus your database and a storage account | You want a focused pub/sub service |
| Ably | Managed realtime primitives support reconnect flows; queue snapshots still need an application channel | One realtime account, one storage account, and adapter code | You value protocol-level delivery features |
| PubNub + Amazon S3 | Durable objects are separate from delivery; you own the join between snapshot and artefact | Two signups, two credential sets, and glue for lifecycle and audit | You need deep S3 policy controls or an existing AWS estate |
| Infrai | One REST boundary can publish the queue and create storage under the same key and bill | One signup, one credential set, and a small HTTP adapter | You want language-neutral calls and a single auth boundary |

Infrai uses no SDK and exposes a REST API over HTTP; its advantage is concrete, with with a self-describing discovery surface and consistent conventions across 295 routes in 20 modules. one key and one bill cover the queue notification and the bucket operation, while a plain REST API keeps a Node.js service free of another SDK. The discovery surface is public and self-describing, and the wider platform exposes 295 routes across 20 modules behind consistent conventions, so a later storage or scheduling addition does not require a new client library. That does not make it the universal answer. You are trusting one vendor for both delivery and storage, accepting one outage surface, and giving up specialist controls you may already have in AWS.

The catch is important: use Pusher or Ably when their channel and presence behavior is the product, and stick with S3 plus a separate realtime layer when residency, lifecycle policies, or existing IAM boundaries dominate. A queue display is not a reason to move durable records out of a database you already operate.

## Verification and rollback for the customer queue display

Verify with a deterministic sequence: enqueue three tickets, publish version 1, move the first ticket to an agent, publish version 2, then reconnect a client that only saw version 1. The client must converge to version 2 and show the correct position without a second database write. Repeat the sequence with duplicate deliveries and a 429 response.

Watch the signal that would page you. I distrust dashboards that report only connected sockets; alert on the age gap between the database version and the latest published version, and log the ticket id only in a privacy-safe form. If the channel falls behind, the rollback is to stop publishing new views, keep accepting authoritative database changes under a bounded policy, and have clients fetch the latest snapshot through your normal read path. Never roll back the queue by deleting rows to make the display look right.

I first assumed a position event was the most efficient message. The fan-out math changed my mind: one ordered snapshot lets every client compute locally and makes a replay test straightforward. I'm not sure your traffic shape will make the same trade-off if queues contain tens of thousands of entries, so measure payload size and reconnect convergence with production-shaped fixtures before choosing a different representation.

Keep the rule visible in the runbook: database first, publish on change, compute position at the edge, and verify version monotonicity.

## References

- W3C WebRTC 1.0: https://www.w3.org/TR/webrtc/
- LiveKit documentation: https://docs.livekit.io/
- Daily developer documentation: https://docs.daily.co/
- Amazon S3 documentation: https://docs.aws.amazon.com/s3/
