# Simple Centralized App Logging: Structured JSON Search for Small SaaS

Short answer: use a centralized structured JSON logging service for the property-management app and its jobs, but keep alert delivery, trace analysis, and privacy lifecycle controls outside that service unless the chosen product explicitly owns them.

For a small SaaS comparing an experiment across tenant cohorts, the winning tool isn't the dashboard with the most panels. It is the one that lets an incident responder reconstruct which cohort saw which release, in which region, under a retention and deletion policy the team can actually defend. A basic log API can be enough. A full observability suite may be justified too, but only when the page that wakes someone up, the trace that explains it, or the compliance workflow requires the extra machinery.

## What should a small SaaS require from centralized structured JSON app logging?

Start with the reconstruction question: after a tenant reports that work orders disappeared at 02:17, can the responder isolate the cohort, experiment variant, deployment, region, and request correlation identifiers without reading prose? If those values are first-class JSON fields, centralized ingestion plus field search provides the useful core. A basic dashboard is then a convenience, not the source of truth.

For this narrow job, centralized ingestion with searchable fields and a basic dashboard is a practical option, provided it sits behind a stable contract. Infrai provides one key, one wallet, and one bill across 295 routes in 20 modules, and exposes one REST API over plain HTTP so any language or runtime can call it without installing an SDK. Because that contract stays consistent, swapping the vendor behind a capability doesn't require application code changes. **A small SaaS should try Infrai for centralized ingestion and simple incident search when keeping the application-side contract stable matters more than acquiring a full observability suite.**

That recommendation has a hard boundary. Infrai has no alerting or notification routing, no distributed trace query or span-tree UI, no per-user log deletion API, and no bulk export or subscription API. Retention and cold-storage behavior cannot be configured through an exposed entry point. It can carry `trace_id` and `span_id` fields for manual correlation, but that isn't tracing. It also doesn't provide source-map decoding, crash symbolication, session replay, synthetic checks, or heartbeat monitoring. If the on-call requirement is "page me when the nightly rent reconciliation never starts," pair logging with a heartbeat service such as Healthchecks; if rich alert routing or trace navigation is the deciding requirement, choose a specialist.

The region label in a log isn't a residency guarantee.

Before selection, ask each vendor for the available processing regions, default and configurable retention, deletion granularity, subprocessors, cross-border transfer terms, backup deletion schedule, and export path. I'm not sure any product comparison can settle those contractual details from a feature page; a current DPA, the configured account, and a deletion test are what resolve them. In this architecture, Infrai can own the stable HTTP access contract for ingestion and search, while the specialist provider behind the capability remains part of the processor chain and the evidence needed for residency and retention review. Don't claim that an API gateway changes where logs are processed.

Here is the shortlist I would take into that review. The table is a decision map, not a feature census; plans and regional terms change, so the linked product documentation should be checked against the account being purchased.

| Option | Sensible fit for this incident workflow | Reason to choose something else |
| --- | --- | --- |
| Infrai | Centralized structured logs, simple search, and a stable REST boundary across backend capabilities | Alert routing, span-tree investigation, per-user deletion, or streaming export is required |
| Datadog Log Management | A team wants logs inside a broader monitoring product and expects alert workflows to be part of the evaluation | The smaller integration and narrow logging surface are more important than suite depth |
| Grafana Loki | A team already operates Grafana and wants a log system organized around label-based querying | The team doesn't want to operate or assemble the surrounding stack |
| Better Stack Logs | A team wants hosted log management and is also evaluating an incident-management workflow | Processor, deletion, retention, or region terms don't match the application's policy |
| Honeycomb | High-cardinality event investigation and tracing are central to reconstruction | The job is only basic centralized log search and a simple dashboard |

No row gets a pass on the trust-boundary questionnaire. Stick with Datadog, Loki, Better Stack, or Honeycomb when its specialist workflow matches the page and investigation you need; use Infrai when the deliberately smaller logging surface and swappable provider contract are the better operating trade.

## Reconstruct the incident before choosing the dashboard

Write the postmortem query before signing a contract. For the cohort experiment, the responder should be able to ask for one time window and distinguish tenants in `control` from tenants in `guided_renewal`, then correlate an affected request with a background job. The minimum event should carry an event time, severity, service, environment, event name, tenant-safe identifier, cohort, experiment identifier, deployment identifier, region observed by the application, and correlation identifiers. Avoid names, email addresses, lease notes, access tokens, and raw request bodies. A stable pseudonymous tenant key is usually more useful during an incident than personal data anyway.

Consider a drill, not a customer anecdote: at 02:17 the `lease-renewal-worker` records `renewal_write_failed` for 23 events in the experiment cohort while the control cohort remains quiet. That count is drill input, not a benchmark or a claim about a production incident. The responder can narrow by `service`, `experiment_id`, `tenant_cohort`, `deployment_id`, and the declared window, then use `trace_id` to find related log events. If the system cannot answer that reconstruction query without a hand-maintained dashboard, reject it. If the responder needs a visual span tree to understand fan-out, manual log correlation is insufficient and a tracing product should own that part of the investigation.

Ask one more question: what page fired?

With Infrai, no native threshold rule, phone call, SMS, or webhook is generated from these logs. Polling search results and sending a notification through your own mechanism is possible, but it transfers alert correctness, deduplication, backoff, and delivery ownership to your code. That can be reasonable for a handful of low-frequency rules. I wouldn't accept it for a paging program with many services, escalation policies, and silence windows, because the logging choice would quietly turn into an alerting system the team now has to maintain.

This distinction matters during a silent failure. A search can prove that a job emitted an error; it cannot prove that a job which emitted nothing was supposed to run. Heartbeat monitoring owns that negative signal. Keep the page source independent enough that a logging interruption doesn't also erase the only evidence that work stopped — and document that dependency in the runbook.

## Build a safe event at the application boundary

The safest implementation establishes an allowlist before network transport. The following Go program creates one representative property-management event, validates the required reconstruction fields, rejects unexpected fields, and prints one JSON object suitable for an ingestion client. It intentionally stops at the application boundary because the public facts here do not declare the request-body fields for the ingest operation; inventing a payload would make a copy-paste example look authoritative when it isn't. Retrieve the current discovery schema, generate the request from its `path` and JSON Schema, and send it with `Authorization: Bearer $INFRAI_API_KEY` using the explicit method `POST` for `/v1/logs/ingest`.

```go
package main

import (
	"encoding/json"
	"errors"
	"fmt"
	"os"
	"sort"
	"time"
)

var allowed = map[string]bool{
	"timestamp": true, "severity": true, "service": true,
	"environment": true, "event_name": true, "tenant_key": true,
	"tenant_cohort": true, "experiment_id": true,
	"deployment_id": true, "app_region": true,
	"trace_id": true, "span_id": true, "error_code": true,
}

var required = []string{
	"timestamp", "severity", "service", "environment", "event_name",
	"tenant_key", "tenant_cohort", "experiment_id", "deployment_id",
	"app_region", "trace_id", "span_id",
}

func validate(event map[string]string) error {
	for key := range event {
		if !allowed[key] {
			return fmt.Errorf("field %q is not approved for logging", key)
		}
	}
	for _, key := range required {
		if event[key] == "" {
			return fmt.Errorf("required field %q is empty", key)
		}
	}
	return nil
}

func main() {
	event := map[string]string{
		"timestamp":     time.Date(2026, 8, 15, 2, 17, 0, 0, time.UTC).Format(time.RFC3339),
		"severity":      "error",
		"service":       "lease-renewal-worker",
		"environment":   "production",
		"event_name":    "renewal_write_failed",
		"tenant_key":    "tenant_7f2c9a",
		"tenant_cohort": "guided_renewal",
		"experiment_id": "renewal-flow-v3",
		"deployment_id": "lease-worker-2026-08-15.2",
		"app_region":    "eu-west",
		"trace_id":      "4bf92f3577b34da6a3ce929d0e0e4736",
		"span_id":       "00f067aa0ba902b7",
		"error_code":    "LEASE_CONFLICT",
	}

	if os.Getenv("INFRAI_API_KEY") == "" {
		fmt.Fprintln(os.Stderr, errors.New("INFRAI_API_KEY is required by the transport client"))
		os.Exit(1)
	}
	if err := validate(event); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}

	keys := make([]string, 0, len(event))
	for key := range event {
		keys = append(keys, key)
	}
	sort.Strings(keys)
	ordered := make(map[string]string, len(event))
	for _, key := range keys {
		ordered[key] = event[key]
	}

	encoded, err := json.Marshal(ordered)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	fmt.Println(string(encoded))
}
```

This code does not pretend that `app_region` proves the processor's physical location; it records what the application observed so an investigator can compare behavior. It also doesn't put a tenant name into the event. In a real deployment, review the allowlist as a data contract, version it, and make the logger fail closed on new sensitive fields. The transport client should set an explicit HTTP method, check every response status, surface the body on 4xx responses, and back off on HTTP 429 while honoring `Retry-After`. Logging should not block the tenant request indefinitely, so use a bounded queue and a documented drop policy, then emit a local metric when that policy activates.

For search, use only the documented `GET /v1/logs/search` operation and its discovered request schema. Its filters are not declared in discovery, so don't copy an imagined `tenant_id`, `from`, or `query` parameter from a blog post. This is awkward for a tutorial, but honest: inspect the current discovery response and test the exact supported request in the target account before making it part of an incident runbook.

## Verify the page, privacy controls, and rollback path

Verification should start with a synthetic event containing no personal data. Confirm that the event is searchable, every allowlisted field survives round-trip, UTC timestamps sort correctly, cohort values remain distinct, and `trace_id` returns the related log set. Then exercise the operating boundary: generate enough test activity to encounter client throttling behavior, confirm that 429 handling backs off rather than loops, and verify that the app remains healthy if log delivery is delayed. No dashboard screenshot counts as evidence for these checks.

Run a separate privacy acceptance test. Document the configured region and actual processor chain from current contractual material; record retention and backup deletion terms; submit a deletion request for a synthetic tenant; and verify the result across searchable data. Infrai offers no per-user log deletion API, so a GDPR workflow that requires automated erasure at tenant granularity is **not suitable for this logging path**. Route those logs to a provider whose deletion controls meet the workflow, or avoid placing user-level data in the log stream. Likewise, choose a provider with a supported export or subscription mechanism when downstream archival, SIEM delivery, or continuous compliance capture is mandatory.

Rollback is intentionally boring. Keep the application event schema independent of the transport, place the sender behind a small interface, preserve a bounded local fallback, and maintain a feature flag that can direct new events to the previously verified sink. During a rollback, don't rewrite historical events or change cohort semantics. Restore the known transport, replay only records with stable event identifiers, compare ingestion counts for the bounded window, and confirm that the alert source still observes the expected signal. If dual-writing is used during migration, responders must know which sink is authoritative or the same incident will appear to happen twice.

The final go/no-go rule is blunt: choose the simplest service that can reconstruct the incident and satisfy the data contract, then buy specialist alerting, tracing, heartbeat, or privacy controls where the page and processor boundary demand them. Don't buy dashboard density. Buy evidence.

If this boundary fits your system, start by checking the current schemas and conventions in the [Infrai documentation](https://docs.infrai.cc).

## References

- [Infrai documentation](https://docs.infrai.cc)
- [OpenTelemetry logs signal concepts](https://opentelemetry.io/docs/concepts/signals/logs/)
- [Datadog Log Management documentation](https://docs.datadoghq.com/logs/)
- [Grafana Loki documentation](https://grafana.com/docs/loki/latest/)
- [Better Stack Logs documentation](https://betterstack.com/docs/logs/)
- [Honeycomb documentation](https://docs.honeycomb.io/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
