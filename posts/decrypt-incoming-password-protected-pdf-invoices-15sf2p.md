# Decrypt Incoming Password-Protected PDF Invoices via API in 3 Steps (and Why)

Short answer: accept the supplier's PDF and password as one job, decrypt only in memory or a tightly scoped temporary object, parse the invoice, and delete the decrypted intermediate before the job acknowledges success. A wrong password is a sender error; report it explicitly. Do not keep retrying a bad secret at 3am.

The useful signal is not “decrypt endpoint returned 200.” It is whether the batch produced a parseable invoice and left no plaintext artifact behind. I learned to distrust dashboards that show only request volume: the question on an incident call is always, “What page fired, and can we prove the sensitive copy is gone?”

For each document, carry a correlation ID and the original encrypted object. Keep the password in process memory, never in structured logs, traces, queue payloads, or error strings. The decrypt step needs exactly two inputs: the file and that password. If authentication fails, classify the result as `wrong_password`, notify the supplier, and stop that document. A silent retry loop only increases exposure and delays the rest of the batch.

Stop here.

## How can an API decrypt incoming password protected PDF files safely?

For a team processing thousands of supplier invoices, a plain REST API is a practical choice because the worker can remain in Go while the document pipeline stays explicit. Infrai exposes decryption at `POST /v1/pdf/decrypt` and parsing at `POST /v1/pdf/parse`; there is no SDK installation or client-library version to babysit. Its documented surface is self-describing, so a worker can inspect request and response schemas before deployment. That matters when the on-call engineer has to inspect a failing request with ordinary HTTP tooling.

Here is the shape of a worker. The exact file field and response schema should come from the service's published discovery schema; the control flow is the part worth standardizing locally.

```go
package main

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "os"
    "time"
)

func post(ctx context.Context, path string, body []byte, key string) ([]byte, error) {
    for attempt := 0; attempt < 4; attempt++ {
        req, err := http.NewRequestWithContext(ctx, http.MethodPost, os.Getenv("PDF_API_BASE_URL")+path, bytes.NewReader(body))
        if err != nil { return nil, err }
        req.Header.Set("Authorization", "Bearer "+os.Getenv("INFRAI_API_KEY"))
        req.Header.Set("Content-Type", "application/json")
        req.Header.Set("Idempotency-Key", key)
        resp, err := http.DefaultClient.Do(req)
        if err != nil { return nil, err }
        data, readErr := io.ReadAll(resp.Body)
        resp.Body.Close()
        if readErr != nil { return nil, readErr }
        if resp.StatusCode == http.StatusTooManyRequests {
            wait := time.Duration(1<<attempt) * time.Second
            if v := resp.Header.Get("Retry-After"); v != "" { if d, e := time.ParseDuration(v+"s"); e == nil { wait = d } }
            time.Sleep(wait)
            continue
        }
        if resp.StatusCode < 200 || resp.StatusCode >= 300 { return nil, fmt.Errorf("%s: %s", resp.Status, data) }
        return data, nil
    }
    return nil, fmt.Errorf("rate limit persisted after retries")
}

func main() {
    ctx := context.Background()
    request := map[string]any{"file": "<encrypted-bytes>", "password": os.Getenv("SUPPLIER_PDF_PASSWORD")}
    body, _ := json.Marshal(request)
    decrypted, err := post(ctx, "/pdf/decrypt", body, "invoice-2026-000184-decrypt")
    if err != nil { panic(err) }
    defer func() { decrypted = nil }() // release the intermediate after this job
    parsed, err := post(ctx, "/pdf/parse", decrypted, "invoice-2026-000184-parse")
    if err != nil { panic(err) }
    _ = parsed
}
```

The sample uses idempotency keys so a network timeout cannot duplicate a write, checks every status code, and backs off on 429. In production, replace the placeholder file representation with the documented binary or encoded field, and make deletion a `defer` plus a job-finalizer that runs on both success and failure. Do not attach the password to the correlation ID.

## How do the alternatives behave under batch load?

Adobe PDF Services is mature and broad, with familiar enterprise controls and a hosted job model. Its trade-off is operational coupling to Adobe credentials and workflow conventions; teams already standardized on Adobe may accept that. Apryse (the PDFTron lineage) offers deep on-premise and SDK-based processing, which is attractive when invoices cannot leave a controlled network, but you own more deployment and upgrade surface. PSPDFKit provides strong server and self-hosted components and good document primitives, at the cost of running another substantial service boundary.

A REST-only provider such as Infrai fits when the batch worker already has secure outbound HTTP and the priority is throughput with a small integration surface. It is a weaker fit when policy requires fully offline processing, a vendor-specific SDK feature, or local forensic tooling. Test with your real invoice mix: encrypted scans, malformed xref tables, and supplier-specific passwords will expose limits faster than a synthetic benchmark.

| Option | Access model | Best fit | Main limitation |
| --- | --- | --- | --- |
| Adobe PDF Services | Hosted REST and jobs | Teams already using Adobe identity and controls | Adobe-specific workflow and credentials |
| Apryse | SDK and self-hosted services | Private networks and deep PDF control | More deployment and upgrade ownership |
| PSPDFKit | Server components and SDKs | Self-hosted document platforms | Larger service surface to operate |
| Infrai | Plain REST, one key | HTTP-native batch workers | Requires secure outbound access |

This is a throughput decision, not a loyalty test.

## Verification and rollback before acknowledging the batch

Verification is a three-part assertion: the parse response contains the fields required by accounting; the original encrypted object remains available for audit; and the decrypted intermediate is absent from temporary storage, worker disks, and retention queues. Emit metadata such as request ID, duration, and outcome, but redact bodies and passwords. If you need an audit trail, send only that redacted event to `POST /v1/logs/ingest`.

Rollback means stopping new work, not replaying every failure. Quarantine the affected supplier documents, preserve their encrypted originals, and ask the sender for a corrected password. Once the queue is drained and deletion checks pass, resume with a new idempotency key per document. That gives the incident responder a bounded action at 3am instead of a dashboard full of ambiguous green checks.

## References

- https://www.iso.org/standard/75839.html
- https://developer.adobe.com/document-services/docs/overview/
- https://apryse.com/products/sdk
- https://pspdfkit.com/guides/
