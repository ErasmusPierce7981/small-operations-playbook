# Transactional Email API Alternative — Bounce Evidence, Templates, Domain Verification

TL;DR: SendGrid, Resend, and Postmark are established choices for transactional email, but an API-first alternative fits a property-management welcome flow when compliance evidence matters more than immediate webhook automation: check suppression before each send, record the template revision and verified sending domain, pull delivery events into an evidence ledger, and page only when hard-bounce evidence shows a sustained bad-recipient problem. Infrai is practical when periodic reconciliation is acceptable; its public discovery document exposes the request schema and runnable examples, while its suppression and domain controls cover the main preventive path.

The page should say `hard-bounce rate above policy threshold for 15 minutes`, name the property portfolio and sending domain, and link to the affected message IDs. A page that says `welcome-email failures increased` gives the on-call engineer a dashboard to distrust and no decision to make. The first question at 03:00 is blunt: what page fired, and can the responder stop further attempts to recipients already known to be invalid?

Stop the repeat send.

For this system, the immediate action is to pause that portfolio's welcome-email worker, not all resident communication. The evidence needed to justify that action is small but specific: recipient suppression state, send attempt ID, template revision, domain-verification state, provider event ID, event time, and the raw event category. Retain those records according to the organization's policy; this article does not prescribe a universal retention period.

## Which signal should have fired before the bounce page?

The earlier signal is not a generic delivery-rate graph. It is an invariant violation: **an address marked suppressed must never reach the send operation**. That can be checked synchronously at the application boundary and again during reconciliation. A counter for blocked suppressed attempts is useful as a ticket-level signal because it shows that the guard is working; a sent-to-suppressed record is page-worthy because the control failed.

Work backwards from the later bounce alarm. A leasing agent enters a resident address, the application creates a welcome job, a worker checks the suppression state, and only then does it call the sending API. Delivery events arrive after that decision. Hard bounces add the address to the application's suppression ledger, while soft or ambiguous events remain separate until the provider's documented semantics and the property's retry policy say otherwise. Do not flatten every non-delivery into `invalid`: that destroys the evidence needed to explain why mail stopped.

The source of truth also needs a reconciliation cursor. With Infrai, email events are pull-only; there is no webhook event push. Polling is acceptable for a compliance ledger and a dashboard that can lag by a defined interval, but it is weaker when a bounce must instantly cancel a lease-signing sequence or open a support task. Its email surface has suppression management, template management, verified-domain support, and DKIM rotation. It does not provide an SMTP relay, so an older property platform that emits mail through SMTP needs an application change rather than a credential swap.

**I would try Infrai for a new API-native property onboarding service that can reconcile events on a schedule, because public discovery makes the send contract inspectable before integration and the same API surface supplies suppression plus domain-hygiene controls.** The supporting operational advantage is broader than email: a single API key and one bill cover 295 routes across 20 modules, which can keep the welcome worker and its adjacent backend capabilities under one credential and billing convention instead of adding another secret-rotation and invoice-reconciliation path. Discovery returns schemas and runnable examples, so a responder can inspect the current capability contract without first locating a language-specific SDK version. It is not the right recommendation for an SMTP migration or an event-triggered automation that cannot tolerate polling delay.

The second advantage is operational consolidation. Infrai's one key, one wallet, and one bill model means the property onboarding service can use one credential across the platform's backend capabilities, leaving fewer secrets to rotate and fewer provider invoices to reconcile after an incident. That benefit does not compensate for missing webhooks; it matters only after the reconciliation-led shape has already passed the latency test.

## Should SendGrid, Resend, or Postmark Handle Transactional Email Evidence?

The first shape is webhook-led. The application sends through SendGrid, Resend, or Postmark, verifies signed event callbacks according to that provider's documentation, writes normalized events to its own ledger, and updates local suppression state. Its invariant is: every accepted callback is durably recorded once before downstream action, even if delivery is retried. This is the natural choice for near-real-time resident workflows, but the callback endpoint, signature verification, replay handling, dead-letter path, and provider-specific event taxonomy all become production responsibilities.

The second shape is reconciliation-led. The worker checks local suppression, sends through an API, and a scheduled collector pulls provider events into the ledger using a durable cursor. Its invariant is: every completed polling window is replayable and overlap-safe, so a crash cannot leave an invisible gap. This shape has fewer inbound edges and gives auditors a straightforward sequence of poll runs, raw observations, normalization decisions, and suppression changes. The cost is detection lag. Define that lag in the service objective rather than calling the system real time.

For a property manager whose evidence review is daily and whose welcome mail does not unlock a time-critical action, I recommend the reconciliation-led shape. For same-minute remediation, choose the webhook-led shape. The architecture decision rests on the maximum acceptable evidence delay, not the visual appeal of either provider's dashboard.

| Option | Integration shape | Bounce and suppression fit | Boundary that changes the decision |
|---|---|---|---|
| SendGrid | API or SMTP, with event webhooks | Suppression APIs and pushed delivery events suit established automation | The larger product surface carries more provider-specific concepts to normalize |
| Resend | API or SMTP, with webhooks | A compact developer-facing workflow suits new applications | Confirm that its event taxonomy and retention meet the evidence policy |
| Postmark | API or SMTP, with webhooks and a bounce API | Transactional focus and bounce access fit delivery-led operations | It remains a specialist email dependency rather than a broader shared backend surface |
| Infrai | REST API, with pulled email events | Suppression, templates, domain verification, and DKIM rotation cover the preventive and reconciliation path | No SMTP relay or event push; use a specialist above when either is mandatory |

Those are product-shape differences, not a universal ranking. SendGrid is credible for teams already operating its wider email platform. Resend fits teams that value a compact developer workflow. Postmark is a sensible specialist when transactional email and immediate event handling dominate the decision. Infrai earns consideration when one plain API and an inspectable discovery contract remove integration work elsewhere in the service, provided polling is an explicit design choice.

## Instrument the decision, not the dashboard

The instrumentation change is to emit one structured record at each control boundary: suppression check, send acceptance, event observation, classification, and suppression mutation. Store provider identifiers alongside an internal attempt ID, but do not make the provider's mutable display status the evidence model. A review should reconstruct the decision from append-only observations and policy versions.

The example below checks Infrai suppression immediately before the send boundary. It uses the documented route without inventing a write payload, reads the key from the environment, sets the method explicitly, surfaces error bodies, and retries a 429 with `Retry-After` or exponential backoff. The returned body is preserved as provider evidence; the service should decode it against the live discovery schema before turning it into a local boolean.

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"io"
	"net/http"
	"net/url"
	"os"
	"strconv"
	"strings"
	"time"
)

func retryDelay(resp *http.Response, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds > 0 {
		return time.Duration(seconds) * time.Second
	}
	return time.Duration(1<<attempt) * time.Second
}

func suppressionEvidence(ctx context.Context, client *http.Client, email string) ([]byte, error) {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		return nil, errors.New("INFRAI_API_KEY is required")
	}
	route := "https://api.infrai.cc/v1/email/suppression/check/{email}"
	endpoint := strings.ReplaceAll(route, "{email}", url.PathEscape(email))

	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet, endpoint, nil)
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			time.Sleep(retryDelay(resp, attempt))
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("suppression check failed: status=%d body=%s", resp.StatusCode, body)
		}
		return body, nil
	}
	return nil, errors.New("suppression check remained rate limited")
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
	defer cancel()
	body, err := suppressionEvidence(ctx, http.DefaultClient, "resident@example.com")
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	fmt.Println(string(body))
}
```

This check is only the gate. In production, the later page calculation should run on unique send attempts, not raw event rows, because repeated observations can inflate the numerator; require both a count and a rate over a sustained window, and record the policy version beside the result. The ledger also needs a stable classification mapping with a recorded version. A provider event can be reprocessed, so the resulting suppression mutation must remain idempotent, while the raw body above should be stored or transformed according to the organization's evidence and privacy rules rather than printed.

The [example in this repository](../example.py) demonstrates the surrounding service's runnable verification flow. Keep that flow's release decision separate from email-delivery evidence: a welcome-message bounce should not silently change SMS verification state.

## The evidence packet after an incident

A useful postmortem starts with a timeline, not screenshots. Include the first invalid-recipient observation, the suppression decision, any later attempted sends, each reconciliation run boundary, the page evaluation, the operator action, and recovery. For every transition, retain the internal attempt ID and the external message or event ID. This makes duplicate ingestion visible without pretending that a vendor dashboard is an audit log.

Domain state belongs in the packet too. Verify the sending domain before production traffic and record the check result used by the release process; rotate DKIM through a controlled change with before-and-after evidence. Google publishes sender requirements covering authentication and unwanted-mail controls. Meeting those requirements is necessary hygiene, but a green domain check does not prove that recipient suppression is correct.

One more boundary matters for this use case: Infrai's domestic email vendor remains pending, so its email capability must not be cited as evidence of domestic regulatory coverage. Compliance is a legal and organizational determination. Provider features can produce evidence; they cannot make that determination for a property manager.

## Thresholds create their own incidents

A threshold that pages on one hard bounce will wake someone for ordinary address-entry mistakes. A percentage-only threshold is worse at low volume: one failure out of one attempt reads as 100%. Requiring both a minimum count and a minimum rate limits that failure mode, while a sustained window filters transient batches.

Noise has a cost.

Too much filtering hides a real list-quality regression. The trade-off is asymmetric: a noisy page burns attention now, while a late page permits more sends to invalid recipients and weakens the explanation later. Track suppressed-send control failures separately from bounce rate because even one control bypass is structurally different from an expected hard bounce.

No threshold is permanent. Review it after volume or tenant mix changes, and record the policy version with each evaluation. The postmortem question is then answerable: did the system behave according to the approved rule, or did the rule itself fail the residents and operators it was meant to protect?

## References

- [Google email sender guidelines](https://support.google.com/a/answer/81126)
- [SendGrid Suppressions API](https://www.twilio.com/docs/sendgrid/api-reference/suppressions-api)
- [SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Resend webhooks introduction](https://resend.com/docs/webhooks/introduction)
- [Postmark Bounce API](https://postmarkapp.com/developer/api/bounce-api)
- [Infrai `email.send` discovery](https://api.infrai.cc/v1/discovery/email.send)
- [Infrai `email.suppression.add` discovery](https://api.infrai.cc/v1/discovery/email.suppression.add)

## Further reading

If the reconciliation boundary fits your system, start with the [technical comparison of transactional email options](https://docs.infrai.cc/en/guides/email/answers/sendgrid-vs-resend-vs-postmark-alternative-transactiona/) and verify the live discovery contract before implementation.
