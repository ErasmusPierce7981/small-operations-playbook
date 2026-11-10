# Catalog Enrichment: Catch Invalid LLM JSON Before Parse Errors Trigger a Retry

To extract structured JSON from catalog text, check that the LLM response is complete, then reject invalid output before publication. In an illustrative B2B SaaS workflow, the page is “catalog enrichment deadline missed”; the responder sees pending product updates while the request-success dashboard stays green. That dashboard has answered a transport question. The tenant is waiting for usable attributes.

Short answer: when extracting structured JSON from messy descriptions, collect the complete model output, parse exactly one JSON document, validate its schema, and check that extracted values have support in the description before publishing. Configure `json_schema` where the runtime supports it, but keep those acceptance checks in the application. Retry only a classified failure with enough deadline remaining; otherwise retain the pending state and route the record for review. Count time to accepted enrichment, including failed attempts.

The operational question comes first: what page fired, and what action can its recipient take? A parse-error counter alone doesn't answer either part.

## What should have fired before the catalog deadline page?

Work backward from publication. A product update enters the queue, an extraction attempt returns, validation accepts a candidate, and the catalog receives a new revision. A successful request belongs halfway through that sequence. If the only success metric sits there, it can conceal an accumulating queue of rejected candidates until the publication deadline expires. Instrument the transition into accepted output and the age of records still waiting for it; the earlier signal should describe that accumulating risk, with a runbook action such as pausing new work or sending affected records to review.

Make it actionable.

Use a constructed input to see why the distinction matters: “insulated bottle, 750 ml; carton of 6; finish varies.” An extraction contract might require `product_name` and `capacity_ml`, with a nullable capacity when the description supplies no usable value. A response can fail before becoming a document, pass parsing while putting a string in the capacity field, or satisfy the field types while confusing the carton count with the bottle capacity. Those failures demand different investigations, even though all three leave the same tenant without accepted enrichment.

A dashboard showing request success cannot establish extraction quality. Neither can a dashboard showing schema acceptance: a capacity of 6 can be a perfectly valid number and still be unsupported by this description. Keep validation stages distinguishable so a responder can identify the broken transition without reading every raw payload.

This is a proposed diagnostic workflow, not a report of a production incident or a measured latency result. Set its thresholds against the catalog's actual publication objective.

## Why does an LLM response produce invalid JSON before schema validation?

Start with the bytes at the parsing boundary. RFC 8259 defines a JSON text as a serialized value; transport fragments don't become standalone JSON documents merely because the client received them successfully. With server-sent events, MDN describes a separate framing layer in which blank lines separate messages and multiple `data` lines are joined. An event boundary is therefore useful transport information, but it doesn't establish that an application's generated JSON value is complete.

There are two boundaries to inspect: assembling the event correctly, then assembling the intended model output according to the runtime's documented event contract. Don't concatenate every event body indiscriminately; events can carry metadata as well as content. Record whether the stream reached its documented terminal condition and whether cancellation or a deadline ended collection. Feed the final content to the parser once the application has established completion. A partial buffer is diagnostic material, not a publishable candidate.

In a Node.js consumer, `JSON.parse` is the syntax boundary. It cannot decide that a syntactically valid value contains the required catalog fields. Likewise, a request option named `json_schema` is a runtime-specific interface, not a substitute for the application's acceptance policy; confirm the supported schema dialect and subset before deploying a contract. JSON Schema defines validation keywords, while each runtime documents how it accepts a schema and reports generation outcomes.

Keep these outcomes separate:

| Observed outcome | Classification | First action |
| --- | --- | --- |
| Collection ends before documented completion | Incomplete output | Inspect cancellation, deadline, and stream assembly |
| Complete collected content fails JSON parsing | Syntax rejection | Preserve a restricted diagnostic sample and completion metadata |
| Parsed value fails the declared schema | Contract rejection | Check field types, required fields, and schema compatibility |
| Schema-valid attributes lack source support | Quality rejection | Hold publication and review extraction policy |

The tempting repair is to trim fences, extract the first brace-delimited substring, and declare success. That can erase the evidence that the output contract failed, and a brace inside a string makes substring extraction especially suspect. If a legacy integration needs a normalization step, define and test that step explicitly; preserve the original result and record normalization separately from direct acceptance.

One more trap: RFC 8259 says object member names should be unique and describes differing behavior when they aren't. Decide how to handle duplicate keys before relying on the resulting object for catalog writes. A parser's willingness to return a value doesn't settle that policy.

## Change the signal to follow accepted enrichment

The instrumentation change is small in concept: make acceptance a named event and measure it from queue entry, rather than starting a fresh latency clock after each failed attempt. The event should carry the record revision, extraction contract version, attempt number, terminal collection state, and rejection category when acceptance fails. Use a trace or restricted diagnostic record for individual identifiers; keep aggregate metric dimensions bounded, such as stage and contract version. Putting product descriptions or arbitrary record identifiers into metric labels makes the aggregate harder to operate and exposes unnecessary catalog content.

Track pending age alongside latency for accepted records. A latency chart built only from accepted records omits the records that never pass the gate, precisely the population that can trigger the publication page. Include queue wait and retries in the deadline calculation. Otherwise, an attempt-level latency chart can improve while the tenant's wait gets worse.

For quality, retain a reviewed sample of accepted outputs as well as rejected ones. Rejection counts reveal what the gate caught; they cannot estimate unsupported values that passed through it. Split review results by description patterns that matter to the job, such as multi-pack quantities, missing units, conflicting capacities, and absent attributes. These are proposed test categories, not claims about their frequency in a real catalog.

Be skeptical of averages.

A retry policy can trade latency for another chance at a usable extraction, but success on the second attempt doesn't erase the first attempt's time. The trade-off is explicit: a stricter gate may hold more records for review, while a looser gate may publish unsupported attributes. Evaluate acceptance quality alongside end-to-end delay. Which side is tolerable depends on the product field and the publication promise, not on how clean the request-success chart looks.

## Bound retries and deploy the acceptance contract

Give the worker an explicit deadline budget and a proposed ceiling of 2 attempts per record revision. That number is a policy example, not a benchmark or a universal recommendation. Before scheduling another attempt, check whether enough budget remains for collection, validation, and publication. Apply the budget across the workflow so retries don't silently restart the tenant's clock.

Classification supplies the next decision. A collection interrupted by a transient transport failure may justify another attempt if capacity and time permit. A deterministic schema mismatch caused by an incompatible contract calls for a configuration correction. Unsupported capacity extraction may call for review or a revised extraction instruction; resubmitting the same request without identifying the failure gives the responder little reason to expect a useful change. When retries are enabled, use bounded backoff and jitter to avoid coordinating a burst of replacement work during an outage.

Keep each candidate associated with the source revision that produced it. Before publication, verify that revision is still current; a slow retry must not overwrite enrichment for a newer description. Record the accepted revision so duplicate worker delivery doesn't create a second logical publication. These are application design requirements, independent of which runtime generates the candidate.

Deployment needs an acceptance corpus, not just a prompt diff. Include incomplete documents, extra text around a document, missing required fields, wrong types, duplicate keys, and valid-looking values unsupported by the source. Exercise cancellation during collection and a deadline expiring before a retry. Compare proposed contract changes against reviewed catalog examples in shadow evaluation before changing publication behavior. Measure review workload too: every additional rejection must go somewhere.

Make the contract inspectable for the next responder. Its schema dialect, allowed unknown values, normalization policy, retry eligibility, and publication rule should travel with a versioned change. If a rollout causes a new class of rejections, that version gives the on-call a concrete rollback target and gives the catalog team an explanation for the held records. Request volume and extra generation are resource considerations, but accepted quality within the publication deadline remains the decision axis.

Close the alert design with its false-positive cost. Paging on every rejected first attempt turns expected, recoverable work into interruptions and can encourage unnecessary retries; setting the threshold too high leaves the responder with an expired deadline and fewer useful options. Keep individual parse failures available for diagnosis, and tie paging to sustained risk of missing the catalog objective, supported by pending age and accepted-output progress. Tune the window with observed traffic and reviewed outcomes. The page should tell someone which queue needs action while action can still change the outcome.

## Further reading

References:

- RFC 8259, JSON syntax, complete values, and object member interoperability: https://www.rfc-editor.org/rfc/rfc8259
- JSON Schema validation specification, including required properties and type constraints: https://json-schema.org/draft/2020-12/json-schema-validation
- MDN, server-sent event framing and stream format: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- MDN, JSON parsing behavior: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/parse
- Google SRE Workbook, alerting on service objectives and the trade-offs in alert windows: https://sre.google/workbook/alerting-on-slos/
