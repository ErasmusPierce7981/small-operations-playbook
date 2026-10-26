# Batch Audio Transcription API: Long Recordings and Retry-Safe Completion Integrity

Short answer: for long game-support recordings, choose the transcription path whose completion contract you can replay safely, then measure quality on the answers your private knowledge base must produce. A fast partial transcript that loses a speaker turn or triggers duplicate indexing is not low latency in any operationally useful sense. It is deferred work for the person on pager duty.

The decision is therefore a two-stage control loop: get an accepted job into durable storage, then make every completion notification idempotent before it can update search. Keep the transcript and its evidence separate from the knowledge-base answer, and make a slower quality tier available for recordings where a wrong answer would send a player to the wrong recovery procedure.

That is the contract.

I carry the pager, so I distrust the green dashboard tile that says “webhook delivered.” The question at 3am is narrower: which page fired, what state was committed, and can I replay the event without making the answer worse?

## How should a batch audio transcription API handle long recordings?

A webhook is a notification, not the job result. The receiver needs a durable job record, an authenticated event, and a way to fetch or verify the resulting transcript. The useful state machine is small: `queued`, `running`, `succeeded`, `failed`, and `expired`. A notification can arrive twice, arrive after a timeout, or arrive out of order. Your state transition rules must make those cases boring.

The limitation is unavoidable: a long recording gives you more context, but it also gives segmentation, speaker overlap, and delayed-failure paths more room to matter. A batch job is a better fit when completeness matters more than an immediate partial result; a short synchronous path is the better trade when an agent needs a rough excerpt now and will verify it.

For a game-support archive, the durable key should be an internal recording ID plus a provider-independent attempt or event ID. Do not use the callback timestamp as identity. Do not let a retry create a second document. Store the source object version, language choice, segmentation settings, transcript checksum, and completion time beside the text. That metadata is what lets an on-call engineer distinguish “the same result was delivered twice” from “two different attempts produced two results.”

HTTP semantics matter here. RFC 9110 describes idempotent methods and retry semantics, but it does not turn an application webhook into an idempotent workflow for you. That boundary belongs in your storage model.

The first write should be an inbox record with a uniqueness constraint on the event identity. The second should advance the job only if the incoming version is newer. The transcript indexer should consume an outbox event after the job transaction commits. That ordering prevents a search document from pointing at a transcript state that the job table never recorded.

Replay it.

## Why is quality versus latency a support decision?

A private knowledge base changes the failure cost. The output is not merely a caption; it may become retrieval context for an answer about account recovery, moderation, or an in-game purchase. A transcript can look readable while omitting the one noun that changes the procedure. Word-level accuracy is a useful diagnostic, not the acceptance criterion.

I would score a candidate path against labeled support excerpts with three separate checks: retrieval recall for known articles, answer correctness against a reviewer rubric, and time from upload to searchable status. Keep the corpus bounded and representative: for example, 100 recordings split across accents, noisy voice chat, code names, and the languages the support team actually handles. That is a test design choice, not a benchmark claim.

I initially thought the fastest completed job would be the best default. Then I followed the text into retrieval and found the real boundary: a 90-minute archive can be searchable quickly and still be unfit for an automated answer if the missing term is an account identifier. That is why the policy needs a quality gate, not a single latency target.

A lower-latency tier makes sense for routine triage when an answer is always reviewed. A higher-quality tier is justified when the transcript feeds an automated response or when a missed entity can produce an irreversible account action. The cutoff belongs in a policy table, not in an engineer’s intuition during an incident.

| Recording class | First useful result | Quality gate | Permitted action |
| --- | --- | --- | --- |
| Short, clear support excerpt | Fast | Lightweight spot check | Draft answer for an agent |
| Long call with account or payment terms | Deferred | Entity and retrieval checks | Human approval before send |
| Noisy voice-chat escalation | Deferred | Speaker and terminology review | No automated action |

This is where “completed” needs a sharper definition. Audio-to-text completion means text exists. Product completion means the text passed the checks required by the next consumer. Mixing those states creates a dashboard that is technically green and operationally false.

Green is not done.

## The retry path I want to inspect at 3am

The callback handler should do very little. It validates the request, records the event, and returns an acknowledgement only after the inbox write has committed. A worker performs the slower fetch, validation, quality gate, and indexing work. The following Go sketch shows the important boundary without binding the design to a commercial service:

```go
type CompletionEvent struct {
\tEventID    string `json:"event_id"`
\tJobID      string `json:"job_id"`
\tAttempt    int    `json:"attempt"`
\tResultURI  string `json:"result_uri"`
\tSourceHash string `json:"source_hash"`
}

func AcceptCompletion(ctx context.Context, event CompletionEvent) error {
\tif event.EventID == "" || event.JobID == "" || event.ResultURI == "" {
\t\treturn errors.New("invalid completion event")
\t}

\t// InsertInbox is unique on EventID and commits before acknowledgement.
\tinserted, err := store.InsertInbox(ctx, event)
\tif err != nil {
\t\treturn err
\t}
\tif !inserted {
\t\treturn nil // A replay is already represented durably.
\t}

\treturn store.Enqueue(ctx, event.EventID)
}
```

The worker must also be idempotent. It can fetch the result twice, calculate the same checksum twice, and attempt the same outbox insert twice without producing two searchable records. Use conditional updates for job state, and retain rejected results with a reason. Deleting the bad transcript removes the evidence needed to explain an incorrect support answer later.

Some recordings should remain manual. Overlapping speakers, heavy game audio, and uncertain terminology can defeat an otherwise correct retry design; routing those cases to review costs latency but protects the answer contract.

A practical poison-event policy matters more than another dashboard panel. After a bounded number of validation failures, move the event to a review queue with the recording ID, event ID, failure reason, and last attempt time. Page on the queue’s age or size, not on every transient callback failure. Alerts that page on delivery volume alone train the team to ignore the one page that says the review queue is growing.

## How should the private knowledge base consume the text?

Indexing should preserve provenance at chunk level: recording ID, source offset when available, speaker label when trusted, transcript version, and the quality decision. Chunking only by character count can split a player’s error code from the sentence that explains it. Chunking only by speaker can make long monologues too large for retrieval. Test the boundaries against the support questions you care about.

The answer service should receive retrieved passages and their provenance, then return an answer that can be traced back to those passages. A transcript with low confidence should lower the permitted action, not merely add a warning string that nobody reads. For a private game-support corpus, the safe fallback is often “send this to an agent for review,” with the original audio and transcript available in the same case.

Prompt instructions can shape answer behavior, but they cannot repair missing audio or establish that a webhook was delivered exactly once. Keep prompt evaluation downstream of ingestion tests. The Prompt Engineering Guide is useful for describing that layer; it is not a substitute for testing retrieval, state transitions, or replay behavior.

## Which evidence should decide the operating mode?

Run a small acceptance matrix before choosing a default path. Include clean calls, overlapping speakers, domain terms, long silence, truncated uploads, duplicate callbacks, out-of-order events, and a result that fails validation. Record p50 and p95 time to searchable status, but pair those numbers with the rate of wrong retrieved articles and the number of cases requiring manual correction. Latency without the error surface is a vanity metric.

The comparison should be between capabilities and contracts, not brand names: maximum recording duration, asynchronous job states, callback authentication, replay behavior, result retention, language coverage, speaker handling, word or segment timestamps, and deletion controls. Verify each item against current documentation and a test account; names and limits change. A service that wins a clean-audio test can lose on noisy support calls, and a service with excellent text can still be a poor operational fit if its completion event cannot be safely replayed.

Do not make price the deciding argument. The expensive incident is the one that inserts a duplicate answer, loses the source evidence, or makes an agent trust a confident transcript that failed its quality gate. Set an explicit policy: which recording classes may be searchable immediately, which require review, how long failed jobs remain inspectable, and who owns the queue when the pager fires.

The invariant is simple: every accepted completion has one durable identity, one auditable transcript version, and a retry path that cannot create a second truth. Once that holds, quality and latency become tunable operating modes for game support rather than competing slogans.

## Sources

- https://www.rfc-editor.org/rfc/rfc9110
- https://www.promptingguide.ai
