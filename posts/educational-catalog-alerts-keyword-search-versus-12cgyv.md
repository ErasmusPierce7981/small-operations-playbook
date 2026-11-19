# Educational Catalog Alerts: Keyword Search Versus RAG Under a Page-Change Budget

Keyword search is the simpler starting point for an educational product catalog when the job is to watch a course page and alert on a meaningful diff; reserve retrieval-augmented generation for explaining an already verified change, because an explanation adds another stage whose quality and latency need separate acceptance criteria. TL;DR: ship a deterministic change detector and keyword retrieval first. Add RAG only if readers need answers that retrieved passages alone cannot adequately supply.

For a Node.js application, that decision belongs in the behavior of the service, before the choice of a package. The focused implementation below uses Go to make the alert boundary explicit; the surrounding application can keep its existing runtime. No runtime change is part of the recommendation.

## What page fired, and what actually changed?

Consider a hypothetical production review, rather than a claimed outage: I would open the incident note with “What page fired?” and ask for the old and new snapshots of one course catalog page. Suppose the illustrative record is course EDU-204, its enrollment status changes from open to closed, and the same page also acquires a different navigation banner. Those are example inputs, not observations from a deployed service. A useful alert must identify which change triggered it, preserve the evidence, and distinguish a material enrollment change from presentation noise.

Show the diff.

A dashboard that counts successful retrievals cannot answer that question by itself. Neither can a fluent paragraph saying that enrollment information was updated, if the paragraph does not identify the changed field and the snapshots from which it was derived. In this review, the failure under investigation would be the loss of a traceable relationship between a page revision and an alert; improving the wording of the alert would leave that failure intact.

The invariant is narrower than “always return a useful answer”: an enrollment alert must reference a verified enrollment transition. That gives the team a concrete boundary to inspect during a postmortem, while leaving room for a separate explanation service to help a reader understand the transition. Treat this as an incident-prevention design, with an explicit hypothetical trigger, rather than evidence that any particular retrieval architecture has already failed.

## Is plain keyword search simpler than RAG for a product catalog?

The RAG paper describes a generation approach that combines parametric memory with retrieved nonparametric memory. That is a useful starting definition for this decision: retrieval supplies material to a generator. Adding a generator changes the deliverable from matching material to producing an answer, so evaluate that answer as an additional deliverable instead of assuming that it improves the alert detector.

For the proposed catalog watcher, the ingestion path stores an accepted page snapshot, extracts the fields the team has decided are material, and compares those fields with the next accepted snapshot. Keyword retrieval then helps a user find the affected course by its identifier or words from its description. Neither step requires a generated answer. The alert can state the changed field directly, using the stored before-and-after values.

This distinction also prevents an unfair comparison. Keyword matching, semantic retrieval, and retrieval-augmented generation are three different design choices: a team can propose semantic retrieval without asking a model to write the result. If the actual problem is that a student searches for a concept absent from the course description's wording, test a retrieval change against that problem before adding an explanation stage. Do not make text generation the automatic response to a missed match. For a bounded first release, define two query classes: exact course lookup and descriptive discovery. Require the first to preserve identifiers and structured constraints; use the second to investigate vocabulary mismatch. These are proposed acceptance categories, not a claim about the distribution of real student traffic. Enrollment alerts should continue to derive from the accepted page revisions whichever retrieval method serves discovery.

That separation is the practical simplification. A failed explanation need not prevent a verified enrollment transition from reaching the person who asked to watch it.

## Compare the work each path must finish

Start the latency discussion at the event that matters. For this design, measure alert latency from accepting a new page revision to making its verified diff available, and measure search latency from receiving a query to returning usable results. Keep page-observation delay visible as a separate interval; a fast response after ingestion does not establish that the watcher observed the source promptly. Define these measurement points before choosing targets.

The keyword-only proposal needs a retrievable catalog record and a deterministic alert payload. A retrieval-plus-generation proposal additionally needs selected evidence, an instruction for the generator, and a decision about what to do when the explanation cannot be produced within the response budget. Those are extra responsibilities in the proposed architecture, not benchmark results. Their latency impact needs measurement under the intended workload. No supplied evidence establishes a numeric winner.

Don't infer a latency budget from a fluent answer.

Compare outputs as well as elapsed time. An exact lookup for EDU-204 should return that record; a discovery query such as “courses about river habitats” needs judgments about relevance; an explanation of a changed enrollment status needs evidence that the answer preserves the actual before-and-after values. One aggregate success percentage would obscure those different obligations. A retrieval improvement is worth considering when it fixes a demonstrated discovery problem, while a generated explanation earns its place only when it helps readers complete a task that a direct diff does not cover. There is a trade-off in the keyword baseline: the team keeps a smaller answer-producing path, but takes responsibility for deciding how wording differences, field extraction, and material-change rules should behave. RAG does not remove those decisions from the page watcher. It creates a place to explain their outputs. Budget storage and operational work for snapshots and evidence alongside any proposed inference work; a unit-price comparison would not settle this choice.

That responsibility stays with the watcher.

## Put the alert invariant in the code path

The following Go example is deliberately a domain boundary, not a downloader or a complete service. Its caller must supply two validated, accepted snapshots for the same watched course, with revisions assigned by the snapshot store. Equality here means equality of those accepted revision identifiers; the ingestion layer is responsible for assigning a new identifier when its accepted content changes. The example watches enrollment status only, so a title or navigation change cannot silently acquire the meaning of an enrollment event.

```go
package catalog

import "fmt"

type Snapshot struct {
    CourseID string
    Revision string
    Status   string
}

type Alert struct {
    CourseID string
    Before   string
    After    string
    FromRev  string
    ToRev    string
}

func EnrollmentDiff(old, next Snapshot) (Alert, bool, error) {
    if old.CourseID == "" || old.CourseID != next.CourseID {
        return Alert{}, false, fmt.Errorf("course identity mismatch")
    }
    if old.Revision == "" || next.Revision == "" ||
        old.Status == "" || next.Status == "" {
        return Alert{}, false, fmt.Errorf("incomplete snapshot")
    }
    if old.Revision == next.Revision {
        if old.Status != next.Status {
            return Alert{}, false, fmt.Errorf("revision reused for changed status")
        }
        return Alert{}, false, nil
    }
    if old.Status == next.Status {
        return Alert{}, false, nil
    }
    return Alert{
        CourseID: next.CourseID,
        Before:   old.Status,
        After:    next.Status,
        FromRev:  old.Revision,
        ToRev:    next.Revision,
    }, true, nil
}
```

The useful property is inspectable: the function emits an alert only after checking identity, required fields, revision consistency, and a status difference. It does not decide whether the extractor interpreted the source correctly. A page fetch failure or an ambiguous extraction should be handled before this function, preserving the previous accepted snapshot and recording an observation failure instead of fabricating an enrollment transition. A missing initial baseline needs its own initialization policy.

Missing data isn't a closed enrollment window.

Keep delivery equally explicit. The proposed delivery layer should identify an event by the watch, course, old revision, new revision, and rule version, then persist the event and its delivery state so that a retry can recognize the same transition. The function alone supplies neither persistence nor duplicate suppression. If generated prose is attached, retain the structured alert as the evidence-bearing payload and let prose arrive separately when the product permits it.

One concrete pitfall deserves a test: a valid new revision with an unchanged enrollment status should produce no enrollment alert. Another is a reused revision with a different status; that should surface as invalid input, as the example does, rather than quietly appearing as a normal update. These cases test the contract that matters to the person receiving the alert.

## When does the simpler path stop being enough?

Before deployment, build fixtures for an enrollment transition, a banner-only revision, an incomplete extraction, an identity mismatch, and a repeated delivery attempt. Include a source-layout change that makes the extractor uncertain. These are proposed tests, not reported results. Review the actual diff and event record alongside latency measurements; a green dashboard is insufficient evidence that the right page fired. Deploy extraction and change-rule revisions against saved snapshots first so a reviewer can inspect which alerts the new rules would create.

For discovery, collect representative queries and relevance judgments before deciding that keyword search is adequate. If students need concept-based discovery and the baseline misses relevant courses, a semantic retrieval experiment has a specific job. If they need a cited explanation spanning multiple course descriptions, a RAG experiment has a different one. Require both to demonstrate useful output within an agreed response budget, with a defined outcome for insufficient evidence.

This advice has a boundary. It assumes the material event can be represented as a validated field transition; a watcher meant to detect substantive changes throughout free-form teaching policies may need a broader comparison method, human review, or a separately evaluated interpretation stage. The status-only function would be too narrow for that job. Likewise, the first accepted revision cannot establish what changed before observation began.

For the educational catalog watcher described here, ship keyword search and a traceable diff path first. Keep the Node.js application boundary small, record what page fired, and make retrieval quality earn any added latency through task-specific evidence. Add generated explanations when they solve an identified reading problem while preserving the verified transition that justified the alert.

## References

The source below defines RAG; the watcher, acceptance criteria, and code above are proposed engineering designs.

## Sources

- Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks. Source for the retrieval-plus-generation definition; it does not establish latency or relevance results for this catalog watcher: https://arxiv.org/abs/2005.11401
