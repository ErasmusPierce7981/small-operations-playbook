# Running Video Generation Jobs: 4 Controls to Cancel, Audit, and Bound Cost

Canceling a running promo-video job needs four controls: a durable cancellation request, cooperative interruption inside the worker, a publication gate, and an append-only audit trail. For an edtech platform that turns instructor-uploaded images into promotional clips, I would moderate the images at upload time, before generation starts; on-demand moderation makes cancellation ambiguous because a worker may be rendering from an asset whose eligibility can change underneath it. An Express handler should acknowledge the request quickly, but the handler is not the cancellation mechanism. The worker owns that.

This is the short answer: return an accepted response after recording intent, stop new billable stages from starting, and let the worker report the final outcome. Never claim that work stopped merely because an HTTP request succeeded.

## 1. What page would fire if cancellation lied?

Start with a bounded incident scenario. A course editor uploads six JPEG lesson images, all six pass moderation, and a promo render begins. The editor notices the wrong course title and presses Cancel while the renderer is encoding scene four. The API records the request, but the renderer does not inspect it until encoding completes. Meanwhile a retrying publisher sees an output object and makes it visible.

No vendor failure is required for this incident. It comes from three local state machines pretending to be one: the Express request lifecycle, the worker lifecycle, and the publishing lifecycle. A green cancellation-response dashboard would be actively misleading. I want the page to fire on a more useful invariant: content became publishable after its job had entered `cancel_requested`, or another expensive stage started after that transition.

That is the trap.

The invariant is **a cancellation request closes the gate to future work and publication; it does not rewrite history about work already consumed**. That distinction matters for cost control. CPU or accelerator time spent before the worker observes cancellation is already gone, and an upstream engine may have a point after which interruption is impossible. The audit record must preserve both timestamps instead of collapsing them into a cheerful `canceled: true`.

Moderating uploaded images first gives this workflow a stable input set. The MDN image-format guide is useful at the ingestion boundary because extensions alone do not define what a decoder will receive; formats differ in browser support and capabilities. Decode the upload, constrain accepted media types and dimensions, create a normalized derivative, moderate that derivative, and keep it private until approved. Video generation then consumes immutable approved asset identifiers, never the mutable upload URL.

## 2. How should Node.js Express cancel a running video generation job?

HTTP semantics do not promise that `DELETE` will halt remote computation. RFC 9110 defines `DELETE` as a request to remove an association between a target resource and its current functionality, and notes that representations of the resource might not be destroyed. A command-oriented endpoint can be reasonable, but its method and path matter less than its state transition and authorization rules.

For a Node.js Express service, put the transition in the same database transaction as the audit event. Lock the job row; verify that the authenticated actor may cancel this course's render; change `running` or `queued` to `cancel_requested`; append an event containing the actor, request ID, reason, and server timestamp; commit; then return `202 Accepted`. Repeated requests should return the existing cancellation state without appending fictional extra transitions.

The response proves intent. Nothing more.

Keep the states small enough to reason about:

| State | May start expensive work? | May publish? | Meaning |
| --- | --- | --- | --- |
| `queued` | Yes | No | Accepted but not claimed |
| `running` | Yes, before cancellation | No | Worker owns a lease |
| `cancel_requested` | No | No | Durable intent; worker may still be stopping |
| `canceled` | No | No | Worker confirmed its terminal stop |
| `succeeded` | No | Yes, after a separate gate | Output completed before cancellation won |
| `failed` | No | No | Terminal error, retained for diagnosis |

There is a race at the finish line. If completion and cancellation arrive together, choose one transactionally enforced rule and document it. A defensible rule is that the first committed terminal decision wins: cancellation can move only a nonterminal row to `cancel_requested`, while completion can move only `running` to `succeeded`. Publication still rechecks the row, so a stale queue message cannot expose an output after cancellation.

Do not hold the Express connection open waiting for the encoder. Node.js provides `AbortController` and `AbortSignal` for canceling compatible asynchronous operations within a process, and the request's `close` event can tell you that the client disconnected, but neither fact is a durable business cancellation. A browser tab closing must not silently cancel a paid render, and a process restart must not erase an explicit cancellation.

## 3. Make the worker stop at metered boundaries

Cooperative cancellation works only when code checks at useful boundaries. Polling every frame creates needless database load; checking only after a ten-minute encode protects nothing. The right checkpoints sit immediately before a stage that reserves scarce capacity, submits external work, or publishes an artifact. A notification or queue wake-up can reduce latency, but the database remains the authority after reconnects and restarts.

The following Go sketch is deliberately engine-neutral. It shows the preventative path I expect even when the public API is implemented in Express: reload durable state before each stage, pass a cancelable context into local work, and refuse publication unless the final compare-and-set succeeds.

```go
package render

import (
	"context"
	"errors"
)

var ErrCanceled = errors.New("render cancellation requested")

type Store interface {
	CancellationRequested(ctx context.Context, jobID string) (bool, error)
	MarkCanceled(ctx context.Context, jobID string) error
	CommitSuccess(ctx context.Context, jobID, artifactID string) (bool, error)
}

type Stage func(context.Context) error

func Run(ctx context.Context, store Store, jobID, artifactID string, stages []Stage) error {
	for _, stage := range stages {
		stop, err := store.CancellationRequested(ctx, jobID)
		if err != nil {
			return err
		}
		if stop {
			if err := store.MarkCanceled(ctx, jobID); err != nil {
				return err
			}
			return ErrCanceled
		}
		if err := stage(ctx); err != nil {
			return err
		}
	}

	committed, err := store.CommitSuccess(ctx, jobID, artifactID)
	if err != nil {
		return err
	}
	if !committed {
		return ErrCanceled
	}
	return nil
}
```

`CommitSuccess` must condition its update on the job still being `running`. Store the artifact privately first. Only after that conditional update succeeds should a separate publisher make the artifact reachable, and that publisher should check the state again. Duplicate queue delivery is then dull rather than dangerous.

External generation changes the mechanics, not the contract. If an engine offers a cancellation operation, call it once after durable intent is recorded and retain its response as evidence; continue reconciling until local state is terminal. If it cannot interrupt an accepted render, stop downstream stages and publication anyway. Report the residual work honestly as consumed-after-request rather than labeling the entire job canceled at the request timestamp.

## 4. Audit the race, then alert on consequences

An audit event is not an application log line. Logs are useful evidence, but retention, sampling, redaction, and indexing can change them. The event belongs in durable storage with a unique event ID, job ID, course or tenant scope, actor identity, server-side timestamp, prior state, new state, reason code, and correlation ID. Keep free-form reasons optional and bounded because they can contain personal or sensitive data.

I distrust a dashboard that reports only cancellation counts. It can stay green while every canceled render finishes at full cost. Four measurements answer the operational questions:

1. cancellation observation latency, from committed request to worker acknowledgement;
2. work started after `cancel_requested`, which should be zero;
3. publication attempts rejected by the state gate;
4. consumed compute after the request, grouped by stage rather than guessed from request duration.

The page should fire on invariant violations and sustained user impact, not on every cancellation. A single rejected publication proves the gate worked and deserves investigation, while an artifact that became public after cancellation is urgent. Likewise, a queue-age alert needs the current job state; canceled jobs waiting for cleanup should not masquerade as customer renders waiting to start.

Test the race deliberately. Pause a worker before each checkpoint, commit cancellation, resume it, and assert that no later stage starts. Pause between private artifact creation and `CommitSuccess`; cancel; resume; verify the artifact stays private. Deliver the same job and cancellation message twice. Restart the API after commit but before response. Restart the worker after it observes cancellation but before `MarkCanceled`. These tests are worth more than a mock proving an Express route returned 202.

This design has real limitations and trade-offs: it costs extra database reads, adds states that operators must understand, and leaves private artifacts for asynchronous cleanup. It is not suitable for every media task. For short, cheap, in-process transformations, an `AbortSignal` and a single conditional publish may be enough; for a batch renderer with no safe interruption point, cancellation can only prevent later stages and publication, not reclaim work already underway. Hard interruption is also unsafe for encoders that write non-atomic output, so those need temporary objects and cleanup. The durable state machine earns its keep when generation crosses processes, has material compute cost, or can expose student- or instructor-supplied media.

The final review question is blunt: **can the audit reconstruct what was requested, what actually stopped, what was still consumed, and whether anything became public?** If the answer depends on a success response from the cancellation route, the system is not auditable yet.

## Sources

- https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
- https://www.rfc-editor.org/rfc/rfc9110.html
- https://nodejs.org/api/globals.html#class-abortcontroller
- https://expressjs.com/en/api.html
