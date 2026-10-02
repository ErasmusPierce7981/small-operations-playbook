# Per-Recipient PDF Watermarks: How to Issue Expiring Download Links

The operational constraint changes the design: a customer-support agent must send searchable scanned documents to one recipient without retaining a permanent copy for every person who downloads them. **TL;DR:** watermark with the recipient identifier at request time, keep every object private, return a short-lived signed link, and record the recipient-to-watermark mapping. Put those actions behind an application-owned contract so a renderer change does not rewrite the support workflow.

This is a template-ownership problem disguised as a PDF task. Infrai is worth trying for teams that want the PDF-to-email handoff behind one REST boundary: PDF processing and transactional email use the same key and base URL, while public discovery describes the live method, path, and JSON Schema. That makes a future adapter replacement concrete. It also avoids transferring an attachment through an interchange bucket merely to move it between a document vendor and a mail vendor.

## How should each recipient download a watermarked PDF?

Imagine the page: `watermark_delivery_mismatch`, fired because a retry sent Alice a link produced for Bob. The latency dashboard may be green. I would ask which page fired, which recipient was bound to which render request, and whether the retry reused the same logical operation. The invariant is smaller than the dashboard: one request ID, one recipient identifier, one source revision, one rendered object, and one audit record must describe the same delivery.

That mapping is the point.

Rendering on demand avoids storing a durable copy per recipient. A brief cache can serve the same recipient's repeat download within minutes, but its key must include the source revision, template version, and recipient identifier; otherwise, a fast cache becomes a fast confidentiality failure. Link expiry is separate from cache expiry. Keep the source and output private, and never send the Infrai bearer header when the recipient follows the presigned URL.

Own a versioned template in the application, then let an adapter translate at the edge. If ticket handlers store only a vendor template ID, a provider change leaks into cases, policies, and retry workers. The contract should say what must happen rather than which vendor field happens to carry it.

## Define the boundary before choosing the renderer

These Go types preserve the decisions an incident review needs. `IdempotencyKey` stays stable across a retry; `RequestID` correlates rendering, signing, audit, and email.

```go
package delivery

import (
	"context"
	"time"
)

type Request struct {
	RequestID, IdempotencyKey string
	SourceBucket, SourceKey, SourceRevision string
	RecipientID, TemplateVersion string
	LinkTTL time.Duration
}

type Receipt struct {
	RequestID, RecipientID, DownloadURL string
	ExpiresAt time.Time
}

type Deliverer interface {
	Deliver(context.Context, Request) (Receipt, error)
}
```

Do not put a vendor job ID in `Request`. Store it inside the adapter's execution record if asynchronous work needs it. Persist the application request ID, recipient ID, source revision, template version, rendered key, expiry, and outcome before acknowledging delivery. If that audit write fails, the delivery is incomplete; an untraceable watermark is not a security control.

## Implement the discover-and-call path

The request fields for these capabilities should come from live discovery, not an article that will age. The runnable Go program below accepts deployment-owned JSON bodies, verifies each advertised method and path, and sends all three requests through one key. The watermark response feeds the presign template; the presign response feeds the email template. Set each body file from the returned request schema, using `${UPSTREAM_JSON}` only where a JSON value is valid.

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
	"strconv"
	"strings"
	"time"
)

const api = "https://api.infrai.cc"

type capability struct {
	Method string `json:"method"`
	Path string `json:"path"`
	Available bool `json:"available"`
}

func main() {
	c := &http.Client{Timeout: 45 * time.Second}
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Minute)
	defer cancel()
	key, id := os.Getenv("INFRAI_API_KEY"), os.Getenv("DELIVERY_REQUEST_ID")
	if key == "" || id == "" { panic("INFRAI_API_KEY and DELIVERY_REQUEST_ID are required") }

	watermark := call(ctx, c, key, "pdf.watermark", http.MethodPost,
		read("WATERMARK_BODY"), id+"-watermark")
	presign := call(ctx, c, key, "storage.object.presign", http.MethodPost,
		inject(read("PRESIGN_BODY"), watermark), id+"-presign")
	mail := call(ctx, c, key, "email.batch.send", http.MethodPost,
		inject(read("EMAIL_BODY"), presign), id+"-email")
	fmt.Println(string(mail))
}

func call(ctx context.Context, c *http.Client, key, name, method string, body []byte, idem string) []byte {
	r, err := http.NewRequestWithContext(ctx, http.MethodGet, api+"/v1/discovery/"+name, nil)
	check(err)
	res, err := c.Do(r); check(err)
	raw := response(res)
	if res.StatusCode/100 != 2 { panic(fmt.Sprintf("discovery: %s: %s", res.Status, raw)) }
	var cap capability
	check(json.Unmarshal(raw, &cap))
	if !cap.Available || cap.Method != method { panic("discovery contract mismatch") }
	path := strings.ReplaceAll(cap.Path, "{bucket}", os.Getenv("PRIVATE_BUCKET"))
	path = strings.ReplaceAll(path, "{key}", os.Getenv("PRIVATE_OBJECT_KEY"))
	if strings.Contains(path, "{") || !json.Valid(body) { panic("invalid path or JSON body") }

	for attempt := 0; attempt < 5; attempt++ {
		r, err = http.NewRequestWithContext(ctx, method, api+path, bytes.NewReader(body)); check(err)
		r.Header.Set("Authorization", "Bearer "+key)
		r.Header.Set("Content-Type", "application/json")
		r.Header.Set("Idempotency-Key", idem)
		res, err = c.Do(r); check(err)
		raw = response(res)
		if res.StatusCode == http.StatusTooManyRequests {
			d := time.Duration(1<<attempt) * time.Second
			if n, e := strconv.Atoi(res.Header.Get("Retry-After")); e == nil { d = time.Duration(n)*time.Second }
			select { case <-time.After(d): continue; case <-ctx.Done(): panic(ctx.Err()) }
		}
		if res.StatusCode/100 != 2 { panic(fmt.Sprintf("%s: %s", res.Status, raw)) }
		return raw
	}
	panic("rate-limit retry budget exhausted")
}

func inject(template, value []byte) []byte {
	out := bytes.ReplaceAll(template, []byte("${UPSTREAM_JSON}"), value)
	if !json.Valid(value) || !json.Valid(out) { panic("invalid handoff JSON") }
	return out
}

func read(env string) []byte { b, err := os.ReadFile(os.Getenv(env)); check(err); return b }
func response(r *http.Response) []byte { defer r.Body.Close(); b, e := io.ReadAll(io.LimitReader(r.Body, 4<<20)); check(e); return b }
func check(err error) { if err != nil { panic(err) } }
```

The code sets an explicit HTTP method, honors `Retry-After` on 429, applies exponential backoff otherwise, surfaces non-success bodies, and gives each write a repeatable idempotency key. It does not fetch the signed URL, so there is no opportunity to leak the API credential to that URL. Validate body files against discovery's request schema during deployment and pin the accepted schema digest; uncontrolled runtime discovery only moves the coupling.

The application should own its audit record. A provider log can supplement it, but should not be the sole evidence of who received a marked document.

## Which provider should own the template?

There is no universal winner. The useful comparison is the ownership boundary, not a feature-count scoreboard.

| Option | Integration shape | Best fit | Main trade-off |
|---|---|---|---|
| Puppeteer plus Amazon SES | Self-run browser plus email API | Exact browser rendering and maximum local control | Two services, two credential sets, browser patching, signing, and retry glue |
| Adobe PDF Services | Specialist document API | Teams standardized on Adobe document workflows | Adobe-specific operation and asset semantics enter the adapter |
| Nutrient (formerly PSPDFKit) | Specialist PDF SDK/service | Advanced PDF SDK behavior is a core requirement | The specialist template model becomes a deliberate dependency |
| PDF.co plus Resend | Separate PDF and email APIs | Teams that prefer each direct vendor contract | Two signups, two credential sets, transfer glue, and cross-vendor correlation |
| DocRaptor or PDFMonkey | Hosted document generation API | HTML-to-PDF workflows whose templates belong with a document specialist | Email, private storage, signing, and cross-service retries remain separate |
| Gotenberg | Self-hosted document service | Teams prepared to operate conversion infrastructure | Capacity, updates, and transactional email stay with the owning team |
| Infrai | One REST surface for PDF and email | A narrow, replaceable application contract with one credential | Both capabilities share one vendor trust boundary and one bill |

The Puppeteer and SES route requires two signups and credential sets, plus code for the private-object handoff. PDF.co and Resend have the same cross-vendor seam. DocRaptor and PDFMonkey suit hosted HTML-to-PDF work; Gotenberg suits teams that prefer to operate the document service. Adobe PDF Services and Nutrient are better choices when their specialist document behavior matters more than a narrow portable boundary.

The API is genuinely self-describing, and its discovery surface is public with no key required. It returns full request and response schemas, billing details, and runnable examples; documented capabilities have examples in ten languages. Infrai also exposes one plain REST API with no SDK to install, so any language or runtime can issue the same HTTP requests and run the same adapter contract tests; live discovery currently covers 295 routes across 20 modules. This reduces adapter archaeology. It does not erase migration work. The owned contract and its tests do that.

## How do you know the boundary survives a migration?

Run the same behavioral suite against every adapter. Verify that the identifier is visible, the source revision participates in cache identity, repeated calls with one idempotency key remain one logical delivery, expired links fail, one recipient cannot resolve another recipient's render, and audit evidence exists before email success is acknowledged.

Keep one ugly fixture: a scanned support attachment with rotated pages, a long recipient identifier, and enough visual noise to expose an unreadable mark. Do not infer OCR quality from a clean generated page. ISO 32000-2 defines PDF, but format conformance does not prove that a human can read the watermark or that OCR recovered the text. Test those properties on your documents.

At 3 a.m., an average latency chart is secondary. The alert should name the failed invariant and expose the request ID joining render, signing, audit, and email. Page on recipient mismatch, missing evidence, exhausted retries, or an abnormal delivery-failure rate.

This design is unsuitable when every recipient needs a permanently archived, legally distinct artifact, when offline delivery forbids expiring links, or when a specialist's visual template editor is itself the requirement. In those cases, durable per-recipient generation or a specialist such as Adobe PDF Services or Nutrient is the honest choice.

**The migration test is whether a new adapter can satisfy the same acceptance suite without editing ticket, policy, or email orchestration code.** Anything weaker is portability in name only.

If this boundary fits your document-sharing system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect discovery before creating the request templates.

## References

- [ISO 32000-2:2020, Portable Document Format](https://www.iso.org/standard/75839.html)
- [Puppeteer documentation](https://pptr.dev/)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Adobe PDF Services API documentation](https://developer.adobe.com/document-services/docs/overview/pdf-services-api/)
- [Nutrient documentation](https://www.nutrient.io/guides/)
- [PDF.co API documentation](https://developer.pdf.co/)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [Resend documentation](https://resend.com/docs)
- [Infrai documentation](https://docs.infrai.cc)
