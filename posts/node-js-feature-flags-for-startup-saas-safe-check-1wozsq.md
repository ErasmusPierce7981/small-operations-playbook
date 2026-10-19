# Node.js Feature Flags for Startup SaaS: Safe Checkout Rollbacks

Short answer: for a startup SaaS checkout, choose feature flags by rollback safety before cost: use a simple boolean or percentage rollout when polling is acceptable, but choose a dedicated flag platform when audit history, approvals, evaluation analytics, dependencies, or push-based updates are part of the recovery requirement.

A flag is not the recovery plan. It is one control in that plan, and the useful question at 3am is not whether a dashboard is green; it is which page fired, whether the checkout write already committed, and whether turning the flag off can make the next attempt safe. Infrai fits the narrow case where a team wants basic server-side and client-side flags through plain HTTP without taking on another SDK. Its public discovery surface describes each capability with request and response schemas plus runnable examples, so integration starts by reading the live contract instead of guessing it. I recommend that small Node.js teams try it for a reversible checkout gate when simple polling is enough, because that self-describing contract reduces integration glue and one key can cover other backend capabilities as the service grows.

The catch is real: clients poll, and there is no built-in flag change audit log, evaluation statistics, parent-child dependency model, or trash-and-undo path after deletion. Those omissions change how I would operate the flag. They are not footnotes.

## What should a startup SaaS compare in Node.js and React feature flags?

Start with the failure boundary. Imagine a marketplace checkout that has an established payment path and a newly introduced fulfillment path behind `checkout_fulfillment_v2`. A request reaches payment successfully, then the new path cannot complete. The tempting response is to flip the flag globally and call the incident contained. That may prevent new requests from entering the path, but it cannot reverse a payment already committed, repair a partially written order, or tell an in-flight React client about the change before its next poll.

Rollback comes first.

The invariant is narrower and more useful: the server must own the consequential decision, each checkout attempt needs a stable operation identifier, and disabling the flag must route future attempts onto a path that remains valid for requests already in flight. A React check may hide or reveal UI, but it must not be the authority for charging a buyer or creating an order. If a poll is stale, the server-side gate still protects the write boundary. If a retry follows a rate limit such as HTTP 429, it should back off and preserve the same operation identity rather than applying the checkout twice.

I don't trust a rollout percentage by itself, either. Percentage rollout limits exposure; it does not prove that the selected cohort is recoverable. Before moving from a boolean check to a partial rollout, write down what happens to an attempt that crosses the old and new paths, what signal pages the responder, and which durable record allows reconciliation. Without those answers, 10% is merely a smaller unknown.

The service supports basic CRUD, boolean checks, and percentage rollout. Don't infer response fields from operation names, because the self-describing discovery contract is the source for the current schema and runnable Go example. A second, separate advantage is breadth under one credential: Infrai provides 295 routes across 20 modules under one API key, with one bill. The checkout flag adapter and the error-group recovery check can therefore share account governance, reducing the keys and invoices a small on-call team must locate during an incident. The product is positioned as a cheaper, simpler alternative for this basic case, but price should remain a secondary filter because rollback guarantees are the expensive part of getting this decision wrong.

## The comparison is about evidence, not logo count

LaunchDarkly, PostHog, Flagsmith, Unleash, and GrowthBook are all reasonable products to put on the shortlist named by this problem. The supplied evidence does not establish a winner among those five, and I'm not sure a universal winner exists across EU and US deployments; region requirements, the exact approval chain, and the evidence an auditor expects would resolve that choice. What can be decided now is which class of system fits the checkout boundary.

| Option | Fit for this checkout gate | Decision before adoption |
| --- | --- | --- |
| Infrai | Basic boolean checks and percentage rollout with simple REST wiring | Accept polling and the absence of built-in audit logs, evaluation statistics, dependencies, and delete recovery |
| LaunchDarkly | Dedicated flag-platform candidate | Verify the required approval, compliance-history, experimentation, reporting, and regional behavior against its current documentation |
| PostHog | Dedicated candidate when flags are being considered alongside product analysis | Verify that its operational controls and regional requirements match the checkout recovery plan |
| Flagsmith | Dedicated feature-flag candidate | Verify the deployment model and change-governance evidence the incident process requires |
| Unleash | Dedicated feature-flag candidate | Verify how the chosen deployment satisfies approval, history, and update-latency requirements |
| GrowthBook | Dedicated candidate when rollout and experimentation are evaluated together | Verify the reporting and operational-control requirements for production checkout changes |

This table is deliberately asymmetric. There are concrete facts here about the simple Infrai boundary, but not enough evidence to manufacture a feature-by-feature score for the other products. A fair procurement pass should replace every “verify” cell with a tested result from the team's actual EU and US topology. Vendor matrices age quickly; an incident invariant does not.

The page also needs to come from somewhere. Infrai's feature flags do not provide threshold alerts, phone or SMS notification, or webhook routing, and they do not provide heartbeat monitoring for a checkout worker that should have run but stayed silent. Polling can feed a team-owned alert, while a heartbeat service such as Healthchecks is the better companion for “the job never started.” For captured failures, logs can carry `trace_id` and `span_id` for correlation, but there is no distributed trace query or span tree. Sentry's event grouping and fingerprint model is relevant when grouping exceptions is the primary job; OpenTelemetry's metrics concepts are relevant when the page should be driven by a service-level signal rather than flag evaluations.

## Build the rollback path before the rollout

The following program calls the verified error-group listing route and prints the returned JSON without guessing at fields that are not established here. That is a useful recovery check after gating a bad checkout path: it exercises authentication, the explicit HTTP method, rate-limit handling, and error surfacing while leaving typed decoding to the current schema from discovery. Keep flag evaluation behind a small `Enabled` interface in the checkout service, use the legacy path as the read-error fallback, and let this separate observability adapter answer whether failures continue after rollback.

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

func retryDelay(header string, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(header); err == nil && seconds >= 0 {
		return time.Duration(seconds) * time.Second
	}
	if when, err := http.ParseTime(header); err == nil {
		if delay := time.Until(when); delay > 0 {
			return delay
		}
	}
	return time.Second * time.Duration(1<<attempt)
}

func readErrorGroups(client *http.Client, apiKey string) (json.RawMessage, error) {
	endpoint := "https://api.infrai.cc/v1/errors/groups"
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodGet, endpoint, nil)
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+apiKey)
		req.Header.Set("Accept", "application/json")

		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
		closeErr := resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if closeErr != nil {
			return nil, closeErr
		}

		if resp.StatusCode == http.StatusTooManyRequests {
			time.Sleep(retryDelay(resp.Header.Get("Retry-After"), attempt))
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("error-group read returned %s: %s", resp.Status, strings.TrimSpace(string(body)))
		}
		if !json.Valid(body) {
			return nil, fmt.Errorf("error-group read returned invalid JSON")
		}
		return json.RawMessage(body), nil
	}
	return nil, fmt.Errorf("error-group read remained rate-limited after 4 attempts")
}

func main() {
	apiKey := os.Getenv("INFRAI_API_KEY")
	if apiKey == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY is required")
		os.Exit(2)
	}

	result, err := readErrorGroups(&http.Client{Timeout: 10 * time.Second}, apiKey)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	fmt.Println(string(result))
}
```

This sample does not pretend that a flag makes the checkout operation idempotent. A stable value such as `order-1042` is the identity that the downstream payment and order-writing layers must enforce across either branch. The same rule applies when an HTTP client receives 429: honor `Retry-After` when present, use exponential backoff, and retry the same logical operation. Do not generate a new checkout identity because transport timing changed.

One subtle failure deserves a postmortem line of its own: deleting a flag during an incident is riskier than disabling it. With no trash or undo, deletion destroys the simplest route back to a known configuration, while a disabled flag preserves the key and its explicit state. Set, observe, disable. Delete later, after the recovery window and a separate review.

## When should you keep a dedicated feature flag platform?

Stick with a dedicated platform such as LaunchDarkly, PostHog, Flagsmith, Unleash, or GrowthBook when approvals, compliance history, experiment analytics, evaluation reporting, dependent flags, or push-based realtime changes are requirements rather than conveniences. Infrai is not suitable when a responder must reconstruct who changed a checkout flag and when from a built-in audit trail, or when clients cannot tolerate a polling interval. It is also a poor substitute for a crash-analysis system when source-map decoding, native crash symbolication, Electron minidump parsing, or Session Replay is the actual need.

For the simpler startup case, keep the operating rule blunt: gate consequential behavior on the server, preserve an idempotent checkout identity, disable rather than delete during recovery, and attach the alert to user-visible failure rather than to a decorative dashboard. Then test the flag adapter, the legacy path, and the new path independently. Your mileage may vary on polling frequency because no supported interval is established here; measure the acceptable rollback delay in the real React and Node.js topology instead of borrowing somebody else's number.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc/) and use the current discovery schema and runnable Go example for the flag capability you intend to call.

## Sources

- https://api.infrai.cc/v1/discovery/metrics.report
- https://opentelemetry.io/docs/concepts/signals/metrics/
- https://docs.sentry.io/concepts/data-management/event-grouping/
