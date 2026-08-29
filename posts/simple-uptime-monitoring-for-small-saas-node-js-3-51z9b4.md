# Simple Uptime Monitoring for Small SaaS Node.js: 3 EU-US Cron Health Signals

Short answer: use an external uptime service for the public health endpoint and a Healthchecks-style heartbeat for scheduled imports; keep application logs and metrics for evidence, because an observability API alone cannot ping your service or route a page.

At 03:07, the page I want is not “Node process is up.” It is “EU rental import has produced no accepted result since 02:30; rollback runbook attached.” That distinction matters in a small property-management SaaS where a Node.js endpoint can return 200 while the data behind it is quietly aging.

The decision is about rollback safety. A monitor is useful when its signal tells me which release I can safely stop, restore, or replay. A colorful dashboard that requires interpretation at pager o'clock is just deferred work.

## A page is a rollback decision, not a dashboard

Start with three independent facts. An external probe proves that a customer in the EU or US can reach the health endpoint. A completion heartbeat proves that the cron-triggered import finished and passed validation. An application freshness event proves that the accepted result is still within the business limit. None of these proves the others.

For example, an import expected every 15 minutes may normally take four minutes, while the business allows data to be 30 minutes old. Those numbers are policy examples, not defaults. The heartbeat should be emitted only after the durable commit, with the import key, region, release, and accepted timestamp. “Worker started” is not success. Neither is “one row parsed.”

The rollback test follows from that model: stop the new worker, restore the previous release, and replay the same import key. The monitor should stay quiet until a result is accepted, and a replay must not create a duplicate. If the only alert is a failed HTTP probe, you have no evidence about stale rental data. If the only signal is an application log, a silent cron job produces nothing to inspect.

No page, no confidence.

## How can small SaaS Node.js uptime monitoring protect a health endpoint?

Write the page payload before writing the query. It should include a signal name such as `rental_import_stale`, the region, the last accepted timestamp, and a rollback link. Keep unique run IDs in logs; use stable, low-cardinality labels for metrics, following Prometheus naming guidance. RFC 5424's severity semantics are a useful reminder that a log level is context, not an alert route.

The signal contract is deliberately boring:

| Signal | What it answers | Rollback use |
|---|---|---|
| External uptime probe | Can a user reach the endpoint from outside the deployment? | Separate edge/process failure from data staleness |
| Completion heartbeat | Did the scheduled import finish after validation? | Detect a missing run before a customer reports it |
| Freshness metric or log | Is the accepted result recent enough to serve? | Decide whether the old release can continue serving data |
| Diagnostic logs and metrics | What did the application observe? | Explain the page: supplier, release, duration, and region |

This ordering changes how you instrument the worker. Emit the heartbeat and freshness evidence from the same success branch, after commit. A retry after a rate limit must carry the same idempotency key. A rollback then has a concrete safety property instead of a hopeful dashboard trend.

## How do external monitors and app telemetry divide the job?

Healthchecks.io is the direct fit for a dead-man switch: the job reports completion, and silence becomes the event. Better Stack is a reasonable choice when managed endpoint checks and an incident workflow matter more than a narrowly focused heartbeat. UptimeRobot is useful for straightforward reachability checks, but a successful HTTP response can still hide a missed import. Prometheus with Alertmanager fits a team that already operates metric collection and rule evaluation; it also means owning that operational machinery.

| Option | Strongest role | Rollback-safety question | Poor fit when |
|---|---|---|---|
| Healthchecks.io | Cron completion heartbeat | Can the page identify the missing import and grace period? | You only need public endpoint checks |
| Better Stack | Managed uptime plus incident workflow | Does the alert preserve region and runbook context? | A single heartbeat is the whole requirement |
| UptimeRobot | External health endpoint reachability | Can it distinguish endpoint failure from stale data? | You need dead-man monitoring without another tool |
| Prometheus + Alertmanager | App-owned metrics and routing | Can rules and the service be rolled back independently? | Nobody will operate a metrics control plane |
| App logs/metrics alone | Post-page investigation | What detects an event that emitted no log? | The cron job can fail silently |

The smallest credible setup for this scenario is an external endpoint checker plus a heartbeat monitor, with app-side logs and metrics behind them. Stick with Prometheus when it is already your team's operating system for alerts. Choose a narrower heartbeat product when adding a full metrics stack would create more pages than it removes.

I'm not sure a static feature matrix can settle notification channels, regional probes, or retention for every EU-US deployment; those details change, so verify them in the current product documentation. The catch is operational ownership: every extra router and query path becomes another rollback dependency.

## Instrument the evidence layer without taking ownership of paging

Infrai provides log ingestion and simple service-status metrics through one REST API. The practical advantage here is plain HTTP: any runtime can call it without installing an SDK. A second, different advantage is operational consolidation: Infrai's "one key, one bill" model covers 295 routes across 20 modules, so the same credential can carry adjacent backend work without another client integration or invoice trail. Its public discovery surface is self-describing, so an operator can inspect the documented request and response schema before wiring a rollback test. It still does not provide synthetic uptime checks, cron heartbeat monitoring, threshold rules, SMS, phone, webhook notification routing, or distributed span-tree queries.

That boundary is healthy. Use the external service to decide that a page should fire; use logs and metrics to explain why. Trying to poll query APIs and build a notifier inside a two-person SaaS is a different project, and it moves the failure mode into code you now have to roll back.

Here is a minimal Go sender for an accepted-result log. The application supplies the JSON payload discovered for its account; the example focuses on transport behavior, not an invented schema.

```go
package main

import (
	"bytes"
	"context"
	"fmt"
	"io"
	"log"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 45*time.Second)
	defer cancel()
	if err := send(ctx, required("INFRAI_LOG_PAYLOAD"), required("INFRAI_API_KEY"), required("IMPORT_IDEMPOTENCY_KEY")); err != nil {
		log.Fatal(err)
	}
}

func send(ctx context.Context, payload, apiKey, idem string) error {
	client := &http.Client{Timeout: 10 * time.Second}
	for attempt := 0; attempt < 5; attempt++ {
		baseURL := required("INFRAI_BASE_URL")
		req, err := http.NewRequestWithContext(ctx, http.MethodPost, strings.TrimRight(baseURL, "/")+"/v1/logs/ingest", bytes.NewBufferString(payload))
		if err != nil { return err }
		req.Header.Set("Authorization", "Bearer "+apiKey)
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", idem)
		resp, err := client.Do(req)
		if err != nil { return err }
		body, readErr := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
		resp.Body.Close()
		if readErr != nil { return readErr }
		if resp.StatusCode >= 200 && resp.StatusCode < 300 { return nil }
		if resp.StatusCode != http.StatusTooManyRequests {
			return fmt.Errorf("ingest returned %d: %s", resp.StatusCode, strings.TrimSpace(string(body)))
		}
		delay := time.Second << attempt
		if seconds, parseErr := strconv.Atoi(resp.Header.Get("Retry-After")); parseErr == nil && seconds >= 0 { delay = time.Duration(seconds) * time.Second }
		select { case <-time.After(delay): case <-ctx.Done(): return ctx.Err() }
	}
	return fmt.Errorf("ingest remained rate limited after 5 attempts")
}

func required(name string) string {
	value := os.Getenv(name)
	if value == "" { log.Fatalf("%s is required", name) }
	return value
}
```

The code does not make the external heartbeat redundant. It records what happened after the import was accepted, which is exactly the evidence needed when deciding whether to roll back.

## Test the quiet failure before shipping the alert

A 15-minute schedule does not automatically mean a 15-minute page. Account for queue delay, supplier latency, validation, and the time an operator needs to choose a release. Page on the business freshness limit, then alert at a softer warning threshold if the team needs room to investigate.

Test both paths deliberately: make the endpoint unreachable from an outside region, then suppress the completion heartbeat while leaving the endpoint healthy. The pages must have different names and runbooks. If they look identical, the monitoring design has collapsed two separate rollback decisions into one noisy symptom.

## References

- https://prometheus.io/docs/practices/naming/
- https://datatracker.ietf.org/doc/html/rfc5424
- https://healthchecks.io/docs/
- https://betterstack.com/docs/uptime/
- https://uptimerobot.com/help/
