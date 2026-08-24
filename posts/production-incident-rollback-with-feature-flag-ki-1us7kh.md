# Production Incident Rollback with Feature Flag Kill Switches for Checkout Failures

Short answer: use a dedicated feature flag as a production kill switch around each risky checkout dependency, but treat the flag as the actuator, not the incident-response system; a team still needs an alert, a named operator, and evidence that the safer path actually took over.

For a checkout failure, the first question is not which dashboard looks alarming. It is: **what page fired, and can the responder stop the failing path without shipping code?** A kill switch earns its place when it turns that answer into one bounded action. The application checks the flag immediately before the risky integration, declines that path when the switch is off, and uses a tested fallback such as queuing the work or omitting a nonessential enrichment step. Payment authorization itself may not have a safe fallback, so the switch boundary must be chosen before the incident.

This is rollback in the operational sense, not a substitute for version control. The code remains deployed; exposure to one behavior changes. That distinction matters during a postmortem because the team must reconstruct both the software version and the flag state that customers encountered.

## How should a production incident rollback use a feature flag kill switch?

Start with one switch per failure domain. `checkout.tax_enrichment.enabled` is useful because it names a behavior; `checkout_emergency` is not, because six months later nobody can tell which branch it controls. Assign an owner in the runbook, define the safe value, and write down the customer-visible consequence of changing it. Create dedicated switches for risky integrations, new code paths, and expensive background jobs rather than sharing one broad switch across unrelated behavior.

The guard belongs directly before the side effect. If the application evaluates a switch near request entry and spends another 800 milliseconds doing work before calling the dependency, the observed state and the action can drift apart. A local decision function should return a conservative result when the flag cannot be refreshed, while the exact conservative result depends on the operation: skipping recommendations is reasonable; guessing about payment authorization is not.

Be explicit about that choice.

The incident sequence is short: monitoring pages on a checkout symptom, the responder confirms the affected dependency, the incident commander authorizes the predefined switch change, and a second person verifies both the current flag state and the checkout outcome. Record the incident ID, actor, old value, new value, and timestamp in the incident timeline. That external record is required when the flag service has no built-in change audit history.

Infrai fits a team that wants a plain REST flag check alongside other backend capabilities under one key and one bill, without adding another SDK and credential set. Its flags have no native alert or notification routing, change audit history, evaluation statistics, dependency graph, or push client; clients poll. Those are material boundaries, not footnotes. The switch can stop a path, but the page, approval trail, and incident automation live elsewhere.

## Put the safe path in code before the page fires

The checkout handler should make the degraded behavior visible in its own telemetry. Log a stable switch name, the selected path, a checkout correlation ID, and the reason category; do not log card data, credentials, or other sensitive values. Preserve `trace_id` and `span_id` where they already exist so records can be correlated, while recognizing that those fields do not turn a log search into a distributed trace or span tree.

The following small Go program is an operator-side verification probe for `GET /v1/flags/is_enabled/{key}`. It deliberately prints the validated JSON response instead of guessing at fields that are not declared here. Set `FLAG_API_BASE_URL` to the API v1 base, keep the key in the environment, and pass the flag name as an argument. Every request has an explicit method, a 429 response honors `Retry-After` when it is present, and any other non-success status surfaces its body.

```go
package main

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"net/http"
	"net/url"
	"os"
	"strconv"
	"strings"
	"time"
)

func main() {
	if len(os.Args) != 2 {
		panic("usage: flagcheck <flag-key>")
	}
	baseURL := strings.TrimRight(os.Getenv("FLAG_API_BASE_URL"), "/")
	apiKey := os.Getenv("INFRAI_API_KEY")
	if baseURL == "" || apiKey == "" {
		panic("FLAG_API_BASE_URL and INFRAI_API_KEY are required")
	}

	body, err := readFlag(context.Background(), http.DefaultClient, baseURL, apiKey, os.Args[1])
	if err != nil {
		panic(err)
	}
	if !json.Valid(body) {
		panic("flag API returned invalid JSON")
	}
	fmt.Println(string(body))
}

func readFlag(ctx context.Context, client *http.Client, baseURL, apiKey, flagKey string) ([]byte, error) {
	endpoint := baseURL + "/flags/is_enabled/" + url.PathEscape(flagKey)
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet, endpoint, nil)
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+apiKey)

		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}

		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Second << attempt
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 {
				delay = time.Duration(seconds) * time.Second
			}
			select {
			case <-ctx.Done():
				return nil, ctx.Err()
			case <-time.After(delay):
				continue
			}
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("flag API status %d: %s", resp.StatusCode, strings.TrimSpace(string(body)))
		}
		return body, nil
	}
	return nil, errors.New("flag API rate limit persisted after four attempts")
}
```

Keep writes out of a generic probe. A toggle is state-dependent, and a blind retry during a noisy incident can restore the behavior the operator meant to disable. Use the provider's documented set operation or console with an explicit desired value, then run the read probe as independent verification. The production application should consume the documented response schema generated by discovery, cache the last accepted state for a deliberately short interval, and exercise the fallback in tests; the probe remains useful because it reports what the control plane says without pretending that a green control plane proves checkout recovery.

## Choose the control plane by the evidence you need later

The feature-management products below can all belong on a shortlist, but they optimize different operating models. Do not pick from a logo grid. Pick from the postmortem questions the team will have to answer: who changed the switch, when did each process observe it, which requests used the fallback, and what independent signal proved recovery?

| Option | Operational fit | Main trade-off for this checkout runbook |
|---|---|---|
| Infrai | Simple REST polling when one backend credential and consolidated billing matter | No native flag alert routing, audit history, evaluation statistics, dependencies, or push updates; keep the timeline elsewhere |
| LaunchDarkly | A dedicated feature-management control plane with documented kill-switch workflows | Adds a specialized vendor and operating surface; verify that its governance model matches the responder approval path |
| Unleash | Teams that value an open-source feature-management option and explicit activation strategies | Self-hosting transfers availability, upgrades, and on-call ownership to the team; managed use changes that calculation |
| Sentry | Application error investigation where stack context and source-map processing are central | It supplies incident evidence rather than the checkout kill-switch actuator, so pair it with a flag system |
| Grafana | Teams assembling metrics, logs, traces, dashboards, and alerting from chosen data sources | The integration and operational burden depends on the selected stack; a panel still does not perform rollback by itself |
| Better Stack | A combined monitoring and incident-management surface for teams that want fewer observability components | It remains separate from application branch control; verify the signal-to-action handoff in the runbook |

The catch is straightforward. Infrai is not suitable when push propagation, native alert-to-flag automation, built-in audit evidence, or flag dependency modeling is a hard requirement. Stick with a dedicated platform such as LaunchDarkly when its governance and incident integrations remove work your team would otherwise have to build. Consider Unleash when control of deployment is worth accepting control-plane operations. Sentry, Grafana, and Better Stack address the evidence and paging side of the runbook rather than replacing the branch-control mechanism; they matter because a switch with no trustworthy symptom signal is an operator button in the dark.

I'm not sure which propagation delay your checkout can tolerate; no product page can answer that. Resolve it with a failure-mode test that measures the interval from an authorized switch change to observed fallback behavior across every production instance. Your mileage may vary with poll interval, cache policy, process count, and network boundaries. Set the incident threshold from that measurement, not from an optimistic diagram.

## Verify recovery and preserve the incident record

Changing the switch is the start of mitigation. Verification should use a narrow synthetic or controlled checkout that traverses the safe path, plus service-level signals that reflect customer outcomes. A falling dependency error count is supporting evidence; a successful checkout result is stronger. A dashboard that merely shows the flag value proves almost nothing about the data plane.

Ask what page fired.

If the page was based on dependency errors, keep watching accepted checkout outcomes, latency, and the fallback counter for at least one normal evaluation window. If no page fired and a customer report started the incident, the missing detection becomes a postmortem action. Infrai lacks flag-linked alert thresholds, phone, SMS, webhook routing, synthetic checks, and heartbeat monitoring, so pair it with an alerting system and use a Healthchecks-style monitor when a scheduled job might never run and therefore emit no error event. Manual response is acceptable only when the runbook says who watches the signal and how quickly they must act.

Capture a compact timeline outside the flag system: alert time, declaration time, approved desired state, control-plane verification, first confirmed safe-path checkout, and full recovery. Include application deploy identifiers and correlation IDs, but follow OWASP logging guidance around sensitive data. Because there is no built-in evaluation history, emit an application event when the branch decision is made; aggregate that event carefully enough to answer how much traffic used each path without turning logs into a store of personal checkout data.

Rollback has a rollback. Re-enabling the risky path should require a fixed dependency, a canary cohort, a stop condition, and the same independent outcome check used during mitigation. Don't flip it globally because an upstream status page turned green. If the code path has been retired, remove the flag deliberately after confirming no deployed version reads it; deletion has no recycle bin, so retain the decision record in the normal change system.

## Run the drill before production needs it

A quarterly exercise is less useful than a drill attached to each new high-risk integration. Use a concrete rehearsal: make the staging tax-enrichment dependency unavailable, submit a checkout carrying a unique correlation ID, and wait for the same rule that would page production. The responder should be able to move from the page to the named switch without searching chat history, request the approved safe value, verify the control-plane response with the read probe, and then find the same correlation ID on a checkout that completed through the degraded branch. Check every application instance against the measured propagation objective; one process retaining the old decision is not a partial success. Restore the dependency only after the safe path is visible, re-enable through a canary, and confirm the normal branch with a new correlation ID. The drill is complete when the incident timeline contains enough evidence for someone who did not attend to reconstruct the order of events. This exposes ambiguous ownership and stale runbooks while nobody is waiting on a checkout.

No page, no switch.

The acceptance rule is blunt: the kill switch is ready only when a responder can name the page, the approver, the desired flag state, the customer consequence, the verification query, and the reversal condition. Five green dashboard tiles do not compensate for any missing answer.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- https://launchdarkly.com/docs/guides/releases/kill-switch
- https://docs.getunleash.io/reference/feature-toggles
- https://docs.flagsmith.com/deployment
- https://configcat.com/docs/advanced/caching/
- https://www.datadoghq.com/pricing/
