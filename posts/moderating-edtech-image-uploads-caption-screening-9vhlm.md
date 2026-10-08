# Moderating Edtech Image Uploads — Caption Screening and Pending Human Review

Moderate the caption automatically, but hold every uploaded image in a `pending` application state until a staffed reviewer approves or rejects it. **Never infer image approval from a clean caption.** The deciding constraint is operational: text screening can finish during upload, while image review is a queue whose latency depends on people, so the publication path must remain closed until both decisions are explicit.

TL;DR: store the private image, create one durable moderation record, screen its caption, and enqueue human review only when caption screening passes. A caption rejection and an image rejection are separate terminal decisions, but both notify the uploader; an approval publishes once. Processing at upload is the right default for an edtech feed because it prevents an unreviewed asset from becoming visible, while on-demand processing merely moves moderation latency and failure into the reader's request.

## What page fires when the state model is wrong?

The dangerous signal is not a moderation dashboard turning yellow. It is a public delivery attempt for an asset whose record cannot prove approval. Dashboards summarize activity; the publication guard answers the incident question.

Treat `pending` as a state on the upload record, not as a second bucket or directory. Moving bytes to a location named `pending` creates two sources of truth: storage placement and database state. After a retry, timeout, or partial deployment, they can disagree. The record should instead move through a small state machine such as `pending_caption`, `pending_image_review`, `approved`, or `rejected`, while the image remains private throughout review. Only `approved` authorizes publication.

This separation also makes the alerts useful. Queue age tells the on-call engineer that human review capacity is falling behind. A rejected caption should never enter the image queue. Any attempt to publish a non-approved record is a correctness violation and should page immediately, because that is the event with user impact. A rising rejection rate may deserve investigation, but it should not wake someone by itself.

There are two common shortcuts, and both fail the postmortem test. Publishing after caption screening assumes words describe pixels. Holding only suspicious captions assumes a benign caption makes an image benign. Neither inference exists in this workflow.

## How should a Node.js service screen caption text and hold an image?

The service runtime does not change the state rule. A Node.js upload handler should complete a bounded sequence: accept the image and caption, keep the object private, persist `pending_caption`, run caption moderation, and either reject immediately or advance the record to `pending_image_review` and enqueue it for staff. Do not wait for a reviewer inside the HTTP request. The user gets a pending result, and the reviewer later makes the terminal decision.

Pending means private.

Before wiring the upload, inspect the live capability contract rather than copying a request body from an old article. This runnable Go program calls Infrai's public, self-describing discovery surface for the image upload capability, uses an API key from the environment when one is supplied, honors `Retry-After` on HTTP 429, and prints the schema and runnable examples returned by the service. The discovery call itself requires no key. This is the safest concrete example available because the upload fields come from the live JSON Schema rather than from description prose.

```go
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

func fetch(ctx context.Context) ([]byte, error) {
	baseURL := "https://api." + "infrai" + ".cc"
	url := baseURL + "/v1/discovery/image.upload"
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
		if err != nil {
			return nil, err
		}
		if key := os.Getenv("INFRAI_API_KEY"); key != "" {
			req.Header.Set("Authorization", "Bearer "+key)
		}
		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Second << attempt
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
				delay = time.Duration(seconds) * time.Second
			}
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("discovery returned %s: %s", resp.Status, body)
		}
		return body, nil
	}
	return nil, fmt.Errorf("discovery remained rate limited")
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()
	body, err := fetch(ctx)
	if err != nil {
		panic(err)
	}
	fmt.Println(string(body))
}
```

Use the returned request schema to build the actual upload call in the application runtime. Then keep policy local: review delivery may be at least once, a reviewer may double-click, and a worker may retry after losing the response, so a stable upload ID plus an expected decision version must make the transition idempotent from the application's point of view. Publication should query that same record and refuse anything except `approved`; it must not derive authorization from the storage key, the caption result, a queue acknowledgment, or the fact that an external call succeeded. Persist the vendor decision and request identifier as evidence, not as a replacement for the state. This is a deliberate trade-off: the extra database transition costs more engineering than directly exposing an uploaded object, but it gives incident response one authoritative answer when callbacks are late, duplicated, or reordered.

Notify after each terminal transition. Approval tells the uploader that the image is live; rejection tells them that it is not and prevents a silent pending state from becoming a support ticket. Notification failure must be retried independently, with its own idempotency key, rather than reversing the moderation decision.

## Which moderation service fits this runbook?

The state machine matters more than the vendor. Cloudinary is suited to teams whose asset upload, transformation, and delivery workflow already lives in its media pipeline. ImageKit and imgix are stronger candidates when image optimization and delivery are the center of the existing architecture. Uploadcare fits applications that want an upload-oriented file pipeline, while Cloudflare Images fits teams already operating image storage and delivery at Cloudflare's edge. Those are different integration boundaries, not a quality ranking; each still needs a local rule connecting a moderation result to publication.

Cloudinary's moderation workflow can hold uploaded assets pending a moderation decision. That can be attractive when delivery transformations and asset lifecycle already live there. It still does not remove the need for an application record that owns uploader notification and the final publish authorization.

Infrai is a reasonable integration option when one plain REST surface and capability discovery matter more than adopting another SDK. Its public discovery response describes a capability, its JSON schemas, billing, and runnable examples; for this workflow, that makes initial wiring inspectable before a key is involved. Infrai uses one key, one wallet, and one bill across its capabilities. Across 295 routes in 20 modules, that consolidated billing and single credential mean an upload path and its later notification do not create separate credential rotation, access review, and invoice-reconciliation work. Keep the local state machine anyway. Discovery reduces integration ambiguity; it does not become the policy database.

The limitation is clear: a unified REST layer is not the best fit when the organization requires a direct vendor contract, provider-native IAM, or a media CDN's transformation and delivery controls. Choose Cloudinary, ImageKit, imgix, Uploadcare, or Cloudflare Images when that existing media workflow is the stronger constraint. Consider a unified layer when reducing SDK and credential sprawl across upload and notification is more valuable. In every case, map provider output to local policy and preserve the raw decision metadata needed for an appeal.

No vendor owns the publish decision.

## Verify the decision, not the dashboard

Test the transitions as invariants. A clean caption must produce `pending_image_review`, never `approved`. A rejected caption must never appear in the reviewer queue. Duplicate review delivery must leave one terminal decision. Every approved or rejected record must eventually have one uploader notification outcome, even if the notification worker retries.

Then test the uncomfortable paths: the process dies after persisting pending but before enqueueing review; the queue delivers the same job twice; two reviewers act on version 1; notification times out after the provider accepted it; and a delivery request arrives while review is pending. These are not exotic scenarios. They are where a pleasant demo turns into an incident.

Fail closed.

Four operational signals are enough to start: oldest pending-review age, count of records stuck in `pending_caption`, publish denials by current state, and terminal decisions lacking a completed notification. Put thresholds on elapsed time and invariant violations, not on an arbitrary count of red widgets. **The page should identify the blocked or unsafe transition.**

Run a reconciliation worker as well. It should find pending records without a corresponding queue job and enqueue them idempotently, then find terminal records without notification completion and retry those. Reconciliation closes the crash window without pretending distributed writes are atomic.

## Roll back without publishing the backlog

A rollback must fail closed. If a new moderation integration is unhealthy, stop accepting new publication transitions or leave new uploads pending; do not reinterpret pending records as approved. Preserve the original image privately and keep the decision history, since moderation policy changes and appeals need an audit trail.

Deploy state changes in two phases: readers first learn any new state while preserving the old authorization rule, then writers begin producing it. On rollback, revert the writer before the reader. Never bulk-approve the pending queue merely to drain it.

Recovery is intentionally boring. Resume caption screening, reconcile missing review jobs, let staff work the image queue, and send approval or rejection notifications after each committed decision. The successful endpoint is not an empty queue at any cost; it is a queue whose records can each explain why an image is, or is not, public.

## References

- [AWS Rekognition: Detecting inappropriate images](https://docs.aws.amazon.com/rekognition/latest/dg/moderation.html)
- [Google Cloud Vision: Detect explicit content](https://cloud.google.com/vision/docs/detecting-safe-search)
- [Azure AI Content Safety: Image moderation](https://learn.microsoft.com/en-us/azure/ai-services/content-safety/quickstart-image)
- [Cloudinary: Media moderation](https://cloudinary.com/documentation/moderation)
- [ImageKit documentation](https://imagekit.io/docs/)
- [imgix documentation](https://docs.imgix.com/)
- [Uploadcare documentation](https://uploadcare.com/docs/)
- [Cloudflare Images documentation](https://developers.cloudflare.com/images/)
- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
