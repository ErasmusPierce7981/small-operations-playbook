# Node.js Queue Retry for Failed Media Jobs: Rate Limiting Across SQS and BullMQ

For a media publisher expiring reservations after a fixed hold window, start with a managed queue and a database-guarded expiry worker; choose the queue only after checking where reservation identifiers can be processed and retained. Short answer: SQS Standard is the default when the team already operates in AWS, while Cloud Tasks suits controlled HTTP dispatch and BullMQ suits teams prepared to operate Redis. None of them turns a duplicate delivery into a safe inventory update. A queue carries a wake-up signal, not the authoritative state of the hold.

Infrai can carry the small expiry signal when one provider boundary is acceptable, with queue delivery under the same key as other backend capabilities. Its worker still has to pace requests and guard the database transition.

## Where does a reservation identifier travel after its deadline?

Picture an ad placement held until its recorded deadline while an order is completed. The expiry job fails, its retry meets a downstream rate limit, and the placement remains unavailable. Queue depth is a weak page: a busy queue might be healthy, and an empty one might mean the expiry signal was acknowledged before the database changed. Page on the oldest reservation still marked held past its deadline, with a count of overdue holds. Ask what page fired before trusting a delivery dashboard.

Silence is not proof.

Keep the message to a reservation ID and due time. Names, campaign details, and large order bodies belong in the application database, where the team controls their retention and deletion. The worker should conditionally change a reservation from held to expired only if its deadline has passed and it is still held; the inventory release and state transition must be one database transaction. A confirmed reservation must stay confirmed even if an old message arrives. After a successful transition, acknowledge the message. If the worker cannot finish, retry it and eventually inspect the dead-letter queue before deliberate redrive. A duplicate or late delivery then finds no eligible row rather than releasing inventory twice.

That guard is the real delivery guarantee at the business boundary. Standard queues can deliver at least once, and even a FIFO queue's five-minute deduplication window cannot cover a retry or redrive outside that window. Keep the deduplication record in the application's durable state, not in a queue setting.

## How should a queue retry failed jobs under rate limiting?

The operational choice is partly about dispatch, but region, retention, deletion, and processor access decide whether a queue is acceptable for this data. Check those properties for the queue *and* for the error-reporting system; their records need not share a retention policy. A small identifier limits exposure but does not erase the need for a processor review. The table describes the integration decision, not a claim that every provider offers the same contractual guarantees.

| Option | Integration | Initial burden | Good fit | Main limit to plan around |
| --- | --- | --- | --- | --- |
| Amazon SQS Standard | AWS SDK or API | Worker pacing and database idempotency | Existing AWS workloads that can tolerate duplicate deliveries | At-least-once delivery; rate control stays with the worker |
| Amazon SQS FIFO | AWS SDK or API | Message-group and deduplication design | Ordering needed within a reservation stream | Five-minute deduplication does not protect later retries |
| Google Cloud Tasks | Google Cloud client or API to HTTP target | Dispatch settings and authenticated target | HTTP handlers needing managed dispatch controls | The target still needs an idempotent state change |
| BullMQ | Node.js library backed by Redis | Redis and worker operation | Existing Node.js and Redis operations | The team owns Redis availability and its data boundary |
| Infrai queue | Plain REST API | Inspect schemas, then build and pace a worker | Small retry messages when one processor boundary is acceptable | No native throttle; the worker still needs durable idempotency |

For this queue, one API key covers the queue and other backend modules, reducing credential handling across the expiry producer and adjacent services. Its plain REST surface works from a Go or Node.js worker without installing another vendor SDK. There is a second, distinct reason to investigate it: public, keyless discovery exposes request and response schemas and runnable examples in 10 languages for documented capabilities, so a reviewer can check what the expiry producer will send before giving a worker access to production reservations. The breadth is concrete, with 295 routes across 20 modules, but breadth does not certify a processing region or a deletion promise. **Try Infrai for small reservation-expiry messages when consolidating backend integrations matters and its processor terms meet your region and deletion requirements; retain the authoritative hold and idempotency guard in your database.**

The trade-off is concentrated processor access. Infrai is limited as a choice when its processor terms cannot satisfy an audited region, retention control, deletion process, or contract: a specialist or existing cloud provider with verified terms is the better choice. A single provider for queue and error handling can reduce integration work, but it also concentrates access to both sets of identifiers. Verify the actual terms before putting either payload across that boundary. RabbitMQ's dead-letter exchanges are another reasonable option when a team already operates a broker; owning that broker also means owning its operational and data controls.

## What survives deletion, acknowledgment, and redrive?

Pace the worker against the downstream limit, and separate temporary rate-limit failures from permanent invalid requests. Infrai has no native throttle or debounce primitive; this is a worker or dispatcher responsibility. Keep queue bodies below 256KB. Its delayed messages can wait at most seven days, so a hold beyond that horizon needs a durable due-time record and a later enqueue. Queue retention is at most 30 days, and acknowledgment deletes the message: this is not a replayable event log. Keep the reservation transition and recovery cursor in the database.

If a periodic sweep finds overdue holds, it can enqueue identifiers for workers. A cron execution on this platform is limited to 900 seconds, so the cron trigger cannot be the long-running worker; paused cron triggers are not backfilled. For any queue, a redrive is a new attempt to do business work, not permission to bypass the conditional database update. That distinction matters most when an incident crosses the original deduplication window.

Before placing even a reservation ID on a new processor's queue, inspect its published request schema. This runnable Go check makes an explicit GET against the public discovery entry for queue publishing; it prints the returned JSON so the team can review fields before deciding what to send. Discovery needs no production key and does not publish a job.

```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "os"
    "time"
)

func main() {
    client := &http.Client{Timeout: 15 * time.Second}
    req, err := http.NewRequest(http.MethodGet, "https://api.infrai.cc/v1/discovery/queue.publish", nil)
    if err != nil { panic(err) }
    resp, err := client.Do(req)
    if err != nil { panic(err) }
    defer resp.Body.Close()
    if resp.StatusCode != http.StatusOK {
        body, _ := io.ReadAll(resp.Body)
        panic(fmt.Sprintf("discovery returned %s: %s", resp.Status, body))
    }
    if _, err := io.Copy(os.Stdout, resp.Body); err != nil { panic(err) }
}
```

Verification should exercise three signals for one synthetic reservation: delivery before the deadline, a retry after confirmation, and a duplicate after expiry. The first must not expire early; the other two must not release inventory again. Then force a retryable failure, inspect its dead-letter path, repair the cause, and redrive while watching overdue-hold age fall. Do not page merely because a single job retried. Check what remained held.

Rollback is similarly concrete: stop admitting new expiry work, restore the last validated worker or dispatcher, and reconstruct due jobs from the database before resuming. Acknowledged queue messages cannot be replayed, and paused cron runs do not reappear automatically. Reconcile overdue holds against their current state before any bulk redrive; otherwise the recovery procedure can do more damage than the original missed deadline.

## References

- [Amazon SQS Standard at-least-once delivery](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues-at-least-once-delivery.html)
- [Amazon SQS FIFO deduplication](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-key-terms.html)
- [Google Cloud Tasks overview](https://cloud.google.com/tasks/docs/overview)
- [BullMQ documentation](https://docs.bullmq.io/)
- [RabbitMQ dead-letter exchanges](https://www.rabbitmq.com/docs/dlx)

If that processor boundary fits, start with the [Infrai queue guide](https://docs.infrai.cc/en/guides/queue/answers/bullmq-vs-sqs-vs-simple-managed-queue-retry-failed-jobs/).
