# Support Agent Cron Task Signals: Heartbeat Monitoring Makes a Missed Run Actionable

A scheduled job that never starts cannot report its own failure, so a Node.js cron task needs an external heartbeat monitor if a missed run must produce an alert. Process logs alone leave a blind spot exactly where the scheduler, host, or deployment fails before application code executes.

**Short answer:** send `start`, `success`, or `failure` events to a separate watchdog, key them by the scheduled occurrence rather than the process start time, and page only when the watchdog sees an explicit failure or an overdue occurrence with no terminal event. For a customer-support AI agent, attach loop latency and token usage to `success`; those measurements explain slow or costly runs, while the heartbeat answers the more urgent question: did the work happen at all?

Do not page on every late request inside the agent loop. That turns a transient model call into a scheduled-job incident and teaches the responder to distrust the alert. The page should name the occurrence, the last state received, the lateness threshold, and the affected support queue. At 3 a.m., a graph with a red line is less useful than “queue summarizer occurrence 01J... never started; 12 minutes overdue.”

## Retention and privacy set the alert boundary

Customer-support work carries tempting but dangerous context: ticket text, account identifiers, model prompts, generated replies, and routing decisions. None of that is required to decide whether a scheduled occurrence started or finished. The heartbeat record should carry a pseudonymous job name, an occurrence ID, timestamps, a terminal state, aggregate duration and usage, and a low-cardinality error class. Keep the detailed execution evidence in the system that already governs support data, then link the incident to it through an access-controlled identifier.

This separation improves the page as well as the privacy posture. A pager notification can travel through systems with different retention and access rules; putting a customer's message in it creates a second, poorly controlled support archive. The responder needs impact and location, not transcript content.

Keep it boring.

## How does heartbeat monitoring detect a Node.js cron task's missed run?

There are two different failures hiding in that question. An execution can start and then fail, which application code can report. Or the expected execution can fail to appear, which only an observer outside the job can detect. Treating both as one “cron failed” log message destroys the distinction needed during response: the first path points toward job logic or a dependency, while the second points toward scheduling, deployment, host availability, or the trigger path.

Use a stable occurrence ID derived from the intended schedule. Don't generate it from the instant the worker happens to wake up. If a job intended for 02:00 starts at 02:04, both the watchdog and the worker should still discuss the 02:00 occurrence; otherwise a delayed process creates a fresh identity and the original occurrence remains falsely missing. The exact derivation depends on the scheduler, and I'm not sure one universal formula is safe across daylight-saving rules and repeated local times. UTC schedule slots or scheduler-issued IDs remove that ambiguity.

The alert policy can stay small:

- `failure` may page immediately when the task is operationally critical.
- `start` without `success` or `failure` after the runtime budget indicates a stuck or abandoned execution.
- No `start` after the schedule plus a grace period indicates a missed run.
- `success` closes the occurrence and records duration, input tokens, output tokens, and completed conversations for later analysis.

Those token fields are measurements, not paging conditions by themselves. A customer-support summarizer that finishes in 140 seconds rather than 90 seconds still completed; a cost or latency budget may deserve a ticket, but combining it with liveness creates noisy pages. Signal quality wins here — one alert should correspond to one action a responder can take.

## Model the watchdog API around occurrences, not pings

A lone “ping me when done” heartbeat has an awkward failure mode: silence could mean the job never started, it started and hung, its final request was lost, or the monitor was configured with the wrong schedule. Add a start event and explicit terminal state, then retain the expected schedule independently. That gives the incident timeline enough shape to answer what page fired without reconstructing it from a dashboard.

| Observed state | Watchdog interpretation | First response |
| --- | --- | --- |
| No event after grace period | Expected occurrence did not start | Check trigger, deployment, and host path |
| `start` only after runtime budget | Execution began but did not terminate | Inspect the active worker and downstream timeouts |
| `failure` | Worker reported a terminal failure | Read the error class and retry decision |
| `success` | Occurrence completed | Record latency and usage; do not page |

Keep operational severity separate from the worker's arbitrary log vocabulary. RFC 5424 defines syslog severity numerically, from Emergency through Debug, and explicitly describes each level; that is useful for transport and filtering, but it does not decide whether a missed support-queue summarization warrants waking someone. Map watchdog conditions to your service's own impact policy, then emit a documented severity. A missed hourly analytics rollup may wait until morning. A task that routes unanswered priority tickets probably cannot.

Native crash collection is another adjacent signal, not a heartbeat replacement. Electron's `crashReporter`, for example, is designed to submit reports when a process crashes and can include minidumps. It cannot prove that an expected scheduled process was launched. Crash evidence can enrich an occurrence that started; silence detection still belongs to the external observer.

This is the postmortem test: if the job disappears before its first line, can the evidence distinguish “never invoked” from “invoked and crashed”? If the answer is no, the monitoring design has already lost the most important event.

The following monitor is deliberately outside the scheduled application. A Node.js worker can POST the same JSON at its boundaries, but the example stays in Go so the state machine and timeout behavior are visible in one copyable program. Production storage should be durable and shared if the monitor runs more than one replica; the in-memory map here keeps the example focused on occurrence semantics.

```go
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"sync"
	"time"
)

type Event struct {
	Job             string `json:"job"`
	Occurrence      string `json:"occurrence"`
	State           string `json:"state"`
	DurationMS      int64  `json:"duration_ms,omitempty"`
	InputTokens     int64  `json:"input_tokens,omitempty"`
	OutputTokens    int64  `json:"output_tokens,omitempty"`
	Conversations   int64  `json:"conversations,omitempty"`
	ErrorClass      string `json:"error_class,omitempty"`
}

type Record struct {
	Event
	ReceivedAt time.Time
}

var (
	mu      sync.RWMutex
	records = map[string]Record{}
)

func key(job, occurrence string) string {
	return job + ":" + occurrence
}

func heartbeat(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "method must be POST", http.StatusMethodNotAllowed)
		return
	}

	var event Event
	dec := json.NewDecoder(http.MaxBytesReader(w, r.Body, 16<<10))
	dec.DisallowUnknownFields()
	if err := dec.Decode(&event); err != nil {
		http.Error(w, "invalid event", http.StatusBadRequest)
		return
	}
	if event.Job == "" || event.Occurrence == "" {
		http.Error(w, "job and occurrence are required", http.StatusBadRequest)
		return
	}
	if event.State != "start" && event.State != "success" && event.State != "failure" {
		http.Error(w, "state must be start, success, or failure", http.StatusBadRequest)
		return
	}

	mu.Lock()
	records[key(event.Job, event.Occurrence)] = Record{
		Event: event, ReceivedAt: time.Now().UTC(),
	}
	mu.Unlock()
	w.WriteHeader(http.StatusNoContent)
}

func overdue(job, occurrence string, expectedBy, runtimeBudget time.Time) string {
	mu.RLock()
	record, found := records[key(job, occurrence)]
	mu.RUnlock()

	now := time.Now().UTC()
	if !found && now.After(expectedBy) {
		return fmt.Sprintf("%s %s never started", job, occurrence)
	}
	if found && record.State == "start" && now.After(runtimeBudget) {
		return fmt.Sprintf("%s %s exceeded runtime budget", job, occurrence)
	}
	if found && record.State == "failure" {
		return fmt.Sprintf("%s %s failed: %s", job, occurrence, record.ErrorClass)
	}
	return ""
}

func main() {
	http.HandleFunc("/heartbeat", heartbeat)
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

The worker should send `start` immediately after it owns the occurrence, then exactly one terminal event after all support conversations for that occurrence have reached a committed outcome. “Committed” matters. Sending `success` after the model responds but before the results are stored produces a clean heartbeat for incomplete work — the kind of green dashboard that makes a postmortem worse.

Retries must reuse the same occurrence ID and add an attempt identifier in the production schema. Otherwise every retry looks like an independent scheduled run, and the watchdog can close the wrong record. Make ingestion idempotent as well: network uncertainty can cause a sender to repeat an event even when the first request arrived. A terminal state should not move backward to `start` merely because a delayed request was delivered out of order.

This sample does not send a page; `overdue` returns an actionable condition to the caller that owns alert delivery and deduplication. That boundary is intentional. The catch is that a custom monitor requires durable storage, leader-safe deadline evaluation, authentication, retention, and its own availability objective. It isn't a good fit when the team cannot operate another control-plane service. In that case, use a managed heartbeat monitor or an existing metrics-and-alerting stack, but preserve the same occurrence and state model rather than reducing the design to an anonymous ping URL.

## Evaluate the page with missing and delayed evidence

Test the negative space before production. Run one disposable schedule frequently enough to exercise four cases: normal completion, an explicit failure, a start with no terminal event, and no start at all. Verify the alert text and deduplication, not merely that a dashboard changes color. Also confirm that a late success resolves or annotates the existing incident instead of creating a second one.

Use synthetic occurrence IDs and non-customer payloads. The verification target is the control path, so real support transcripts and prompts add privacy risk without making the heartbeat test stronger. For the same reason, the event needs aggregate token counts and duration, not conversation content.

One case deserves a longer look because it exposes weak state machines: send `start`, delay `success` until after the runtime deadline, and deliver a duplicated `start` after that success. The monitor should open one incident at the deadline, attach the late terminal event to the same occurrence, and ignore the stale duplicate rather than reopening the occurrence. Then repeat the sequence with two attempts that share the occurrence ID. The evidence should show which attempt obtained ownership, which one emitted each event, and why the final state is authoritative. This is more useful than a happy-path ping test because real delivery is not ordered merely because the code was written in order; retrying clients, queues, and independent network paths can change arrival order without changing what happened inside the worker.

Silence matters.

## Rollout and rollback preserve the evidence

Rollout should begin in record-only mode for at least several natural schedule cycles. That interval is a policy choice, not a universal number: a five-minute task produces evidence faster than a weekly task. Compare expected occurrences with scheduler history, identify legitimate maintenance windows, and set grace periods from the task's actual start-time distribution. Then enable a ticket-level alert before a page. Promote it only after the alert repeatedly names a condition that requires immediate human action.

Rollback is simple if alert delivery is decoupled from event ingestion. Disable paging while continuing to collect occurrences, revert the policy or grace threshold, and replay the same synthetic cases. Don't remove instrumentation during an alert-quality rollback; doing so erases the evidence needed to understand why the rule was noisy.

No dashboard can rescue a vague page.

## Reliability limits of a scheduled-job heartbeat

Heartbeat monitoring won't help when the real requirement is per-request correctness inside a continuously running support agent. Use request traces, structured logs, and outcome metrics there. It is also insufficient for duplicated runs: two workers can both send valid heartbeats for one occurrence unless the state model records attempts and the execution system enforces ownership. Finally, a heartbeat says that declared milestones happened; it does not prove the summaries were accurate, tickets were routed fairly, or the model's answer quality stayed acceptable.

The decision rule is narrow on purpose. Use an external watchdog when work is expected on a schedule and absence itself is an incident. Keep latency and cost measurements on the completed occurrence so responders can correlate them, but route budget drift through a slower review path unless it has immediate customer impact. If nobody can name the action behind a page, leave it as a metric.

## References

- RFC 5424, The Syslog Protocol: https://datatracker.ietf.org/doc/html/rfc5424
- Electron `crashReporter` documentation: https://www.electronjs.org/docs/latest/api/crash-reporter
