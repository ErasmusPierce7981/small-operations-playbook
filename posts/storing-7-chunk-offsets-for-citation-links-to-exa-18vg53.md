# Storing 7 Chunk Offsets for Citation Links to Exact PDF Pages

Short answer: store every chunk as a reference to an immutable document version, then record the page index and a half-open byte range into that page's preserved normalized text. A citation should resolve those coordinates, verify the selected bytes, and only then render the page link. If any check fails, suppress the quotation and page the pipeline owner; a plausible compliance citation aimed at the wrong revision is worse than no citation.

The decision rule is operational: prefer the smallest chunk that still answers the question, but never trade away the coordinates needed to reproduce its evidence. Retrieval-augmented generation separates retrieved evidence from generated output; the citation layer has to preserve that separation instead of trusting prose produced later [1]. For a B2B SaaS service answering questions over a folder of compliance-policy PDFs, retrieval quality and latency matter, but reproducibility is the admission ticket.

## What page fired the alert?

Start with the failure signal, because a green search dashboard proves almost nothing. The useful alert is not "vector search is slow." It is "a rendered citation cannot be reconstructed from the indexed document version." That alert identifies the broken contract: the answer names a page, while the stored span belongs to another page, extraction, or revision.

Three common designs create that condition. Storing only chunk text loses its location. Storing a page number plus text fails when the same sentence appears twice. Storing offsets into the PDF file confuses encoded file bytes with extracted text bytes; they are different coordinate spaces, so that range cannot safely select the displayed quotation.

The label printed in a footer is also not a stable array index. A policy may have cover sheets, Roman-numbered introductory pages, or no printed label. Keep `page_index` as the zero-based position in the extracted sequence and `page_label` only for presentation.

This is the postmortem test: can an engineer holding only the citation record and immutable artifact reproduce the quoted bytes? If not, the incident began during ingestion, even if the answer UI exposed it.

Offsets are evidence.

## Store seven coordinates, not a quotation-shaped guess

Use one coordinate system throughout ingestion and resolution. Here, `start_byte` is inclusive and `end_byte` is exclusive within the UTF-8 bytes of `page_text`. The seven fields establishing identity and location are `document_id`, `version_id`, `page_index`, `page_label`, `start_byte`, `end_byte`, and `text_digest`. `chunk_id` is merely an operational handle.

```go
package citation

import (
    "crypto/sha256"
    "encoding/hex"
    "fmt"
)

type Span struct {
    ChunkID    string `json:"chunk_id"`
    DocumentID string `json:"document_id"`
    VersionID  string `json:"version_id"`
    PageIndex  int    `json:"page_index"`
    PageLabel  string `json:"page_label"`
    StartByte  int    `json:"start_byte"`
    EndByte    int    `json:"end_byte"`
    TextDigest string `json:"text_digest"`
}

func Digest(b []byte) string {
    sum := sha256.Sum256(b)
    return hex.EncodeToString(sum[:])
}

func NewSpan(documentID, versionID string, pageIndex int, pageLabel string, pageText []byte, start, end int) (Span, error) {
    if start < 0 || end <= start || end > len(pageText) {
        return Span{}, fmt.Errorf("invalid half-open byte range [%d,%d) for page length %d", start, end, len(pageText))
    }
    selected := pageText[start:end]
    return Span{DocumentID: documentID, VersionID: versionID, PageIndex: pageIndex,
        PageLabel: pageLabel, StartByte: start, EndByte: end, TextDigest: Digest(selected)}, nil
}
```

Byte offsets make slicing unambiguous in Go and survive variable-width UTF-8 characters when writer and reader use the exact preserved bytes. Character counts from one runtime and byte indexes from another do not mix. Pick one convention, name it, and reject records produced under another.

Preserve normalized page text beside the source PDF under the same `version_id`; do not silently re-extract it during resolution. A changed extractor, whitespace rule, or Unicode normalization policy can move every later offset while the PDF looks unchanged. Version those transformations with ingestion.

Chunks should not cross pages. If retrieval needs wider context, store several page-local spans under one retrieval unit. The extra join costs time; it also prevents one synthetic "page" from pointing at two physical pages. For compliance evidence, that trade is defensible.

The 7-field approach has a clear limitation. Byte offsets don't preserve a visual bounding box, so this approach is not suitable when the reader must highlight a row in a scanned table or identify a signature on an image-only page; choose page-local geometry tied to the rendered page instead. Immutable page artifacts also consume more storage than overwriting the latest extraction, and digest checks add work to the answer path. The reason to accept those costs is narrow: policy citations need reproducible text and revision history. For low-risk document search that never exposes quotations or page links, choose a simpler page identifier and skip span verification instead.

## Resolve citations before rendering answers

Resolution should be a pure check against preserved bytes. It must not search for the quotation again, because repeated policy language can produce a confident but wrong match. It must not accept the current revision when the record names an older one.

```go
package citation

import "errors"

var ErrStaleCitation = errors.New("citation does not match preserved evidence")

func Resolve(span Span, pageText []byte) ([]byte, error) {
    if span.StartByte < 0 || span.EndByte <= span.StartByte || span.EndByte > len(pageText) {
        return nil, ErrStaleCitation
    }
    selected := pageText[span.StartByte:span.EndByte]
    if Digest(selected) != span.TextDigest {
        return nil, ErrStaleCitation
    }
    return selected, nil
}
```

The viewer link can carry the immutable version and `page_index + 1` if its page parameter is one-based. That conversion belongs at the viewer boundary, nowhere else. The visible citation may show `page_label`, but the resolver addresses `page_index`.

Keep generation out of this function. A model can cite `chunk_id`; the application maps that identifier to a span and checks it. A wrong model citation then becomes an explicit lookup failure rather than an opportunity to invent a location.

Fail it closed.

Latency pressure usually arrives here. Batch-load spans and artifacts for the retrieved set, cache only immutable versions, and record separate timings for retrieval, evidence resolution, and generation. One end-to-end percentile hides which stage delayed the answer. A cache key must include `version_id`; otherwise a fast hit may return stale evidence.

## How do you verify exact-page links before release?

Test the invariant below the UI, then test the rendered link. The first suite should include multibyte UTF-8 text, repeated sentences, empty printed labels, cover pages, spans ending at the final byte, and deliberately corrupted digests. A two-page retrieval unit must resolve as two page-local citations, never one range.

A release gate can sample indexed chunks and require five conditions:

- The named document version exists and is immutable.
- The page index exists in that version.
- The byte range is valid and lands on UTF-8 boundaries.
- The selected bytes reproduce `text_digest`.
- The viewer opens the same version and physical page.

The last check needs a browser-level test because storage correctness does not prove routing correctness. Use fixtures whose visible label differs from physical position; that catches the tempting off-by-one implementation. Include a revised policy with the same filename. Old answers must open the old version, while newly indexed answers use the new one.

Watch failures, not decoration. Alert on digest mismatches, missing versions, invalid ranges, and viewer-resolution failures, with document version and ingestion build attached. Retrieval relevance belongs in an evaluation set; citation integrity belongs in a hard release gate. Combining them into one score allows a good answer to conceal a broken link.

## Roll back the index, not the evidence

Make ingestion append-only at the version boundary. Build page artifacts, spans, and the retrieval index under a new version; validate them; then move the active-index pointer. If verification fails after deployment, move that pointer back. Existing citations still resolve because their versions and preserved text were never overwritten.

Do not repair offsets in place. Re-run extraction and chunking into a new version, compare failure counts, and switch only after the reconstruction gate passes. In-place repair destroys the evidence needed to determine whether the extractor, normalizer, chunker, or viewer introduced the fault.

The rollback criterion should be dull and explicit: any citation-integrity regression blocks or reverses the index release, even if retrieval latency improved. A slower correct link can be investigated. A fast link to the wrong policy page changes the meaning of the answer.

Seven coordinates work only when their coordinate space is immutable and verified. Store page-local byte ranges, separate physical indexes from printed labels, bind every span to a document version, and make the digest check the gate between retrieval and display. Then the page link is evidence, not a UI promise.

## References

1. Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks: https://arxiv.org/abs/2005.11401
