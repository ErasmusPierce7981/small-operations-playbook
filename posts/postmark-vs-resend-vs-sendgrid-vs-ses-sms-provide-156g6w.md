# Postmark vs Resend vs SendGrid vs SES: SMS Provider Choice for Event Deliverability

**Short answer:** For US/EU product event notifications, Postmark or Resend suit email-first delivery, SendGrid suits broader email tooling, SES suits AWS-owned infrastructure, and an SMS provider or combined API is needed for seller alerts beyond email; whichever path you choose, custom-domain authentication, DKIM, suppression, and idempotent retries matter more than the lowest price.

For a marketplace seller notification, the reliable choice is rarely the provider with the shortest API example. It is the path that lets you authenticate a branded domain, suppress bad recipients, make retries harmless, and observe delivery without pretending that a successful HTTP response means a seller saw a new order. For US and EU traffic, a multi-channel API can fit that job well, but an API-only design leaves more deliverability and orchestration work with your team than a specialized email service does.

That is the trade-off I would put at the top of the postmortem. Custom-domain email and SMS escalation are useful only if the notification remains trustworthy under retry, provider delay, and an already-invalid destination. The notification service should be one part of the incident path, never the definition of delivery.

## What does a reliable order-notification path have to prove?

Start with the event, not the vendor. An order is accepted, an email is requested, the provider accepts or rejects that request, and a later delivery signal may or may not arrive. Those are different facts. Treating them as one status is how an apparently healthy dashboard hides a seller who never received the alert.

The first control is domain identity. A branded sending domain should be configured and verified, and its DKIM material should be rotatable. DKIM does not guarantee inbox placement, but it gives receiving systems a verifiable signing identity and gives the operator a concrete thing to inspect when a delivery rate changes. RFC 6376 describes the signing and verification model; it is a protocol control, not a promise of delivery.

The page is not the receipt.

The second control is suppression. An email address or phone number that has already failed should not be retried blindly for every order. Email and SMS suppression lists make that decision explicit, while a marketplace application still needs to decide how to contact the seller through another channel or show the failure in its operator queue.

The third control is idempotency. A worker that retries after a timeout cannot know whether the provider accepted the first request. A client-supplied idempotency key, derived from the order notification identity, prevents a transient network failure from turning one order into five identical messages. Keep the order event idempotent too; a provider key cannot repair a queue that emits five distinct notification jobs.

Then ask the pager question: what page fired? A provider can report an accepted request while the eventual delivery event is delayed, unavailable, or negative. In this scenario, the useful alert is a sustained increase in rejected sends, suppression hits, or missing delivery observations by region and channel, with the original order ID available for investigation.

## Should you choose Postmark, Resend, SendGrid, SES, or an SMS provider?

No single comparison table can measure inbox placement for your exact sender reputation, list quality, message content, and destination mix. It can clarify the operational shape each option brings to a seller-notification system.

| Option | Strong fit | Trade-off for marketplace alerts |
| --- | --- | --- |
| Postmark | Transactional email with a focused delivery workflow | Email-first; an SMS escalation still needs another service and another operational boundary |
| Resend | A developer-oriented email API for teams that want a small integration surface | The comparison remains email-centric, so multi-channel suppression and escalation are application concerns |
| SendGrid | Email programs that need templates, analytics, and a broad provider feature set | More configuration surface can mean more operational choices to govern for a narrow order-alert path |
| Amazon SES | Teams already operating inside AWS and willing to own more of the surrounding delivery system | The service is a building block; domain, events, suppression, and runbooks still need deliberate assembly |
| Infrai | One API surface for branded email plus SMS alerts, with public capability discovery and runnable examples | No SMTP relay, no webhook event push in these namespaces, and email delivery tracking is more manual than some competitors |

The differences are about boundaries, not a universal ranking. Postmark is a sensible email-first choice when the product has no SMS requirement and the team wants a narrow transactional-email surface. Resend is attractive when a similarly focused email API matches the application and the team accepts that cross-channel policy lives elsewhere. SendGrid is the better-shaped option when its email templates and analytics are central to the workflow. SES can be the right substrate for an AWS-heavy organization that already has the operational controls to wrap it.

An API that combines email and SMS is a different proposition. Infrai's useful distinction here is that its API is self-describing: discovery exposes capability schemas and runnable examples, so wiring a new capability can begin with reading one endpoint rather than learning another SDK. That can shorten the path from an email-only alert to an SMS escalation. It does not remove the need to design delivery state, and it should not be confused with a managed email deliverability program.

## The retry path matters more than the happy path

The code below is deliberately boring. It is a runnable health check for the verified sending domains behind the notification path, not a provider SDK or a claim that a domain list proves inbox placement. The API key and base URL come from the environment, every request names its method, and a 429 honors `Retry-After` before trying again. I keep a four-attempt ceiling in workers because an order alert that retries forever can become its own incident. This is the operational detail I would want in the October 30, 2026 runbook: a retry policy that can be read at 3am without reconstructing it from framework defaults.

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		panic("INFRAI_API_KEY is required")
	}
	baseURL := os.Getenv("INFRAI_API_BASE_URL")
	if baseURL == "" {
		panic("INFRAI_API_BASE_URL is required")
	}

	client := &http.Client{Timeout: 10 * time.Second}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodGet, baseURL+"/v1/email/domain/list", nil)
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+key)

		resp, err := client.Do(req)
		if err != nil {
			if attempt == 3 {
				panic(err)
			}
			time.Sleep(time.Duration(1<<attempt) * time.Second)
			continue
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			panic(readErr)
		}
		if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
			seconds, _ := strconv.Atoi(resp.Header.Get("Retry-After"))
			if seconds < 1 {
				seconds = 1 << attempt
			}
			time.Sleep(time.Duration(seconds) * time.Second)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			panic(fmt.Sprintf("domain list failed: %s: %s", resp.Status, body))
		}
		fmt.Println(string(body))
		return
	}
}
```

There are two traps here. First, a provider response must be checked rather than assumed to be a success; a useful adapter preserves the error body and status for the runbook. Second, a retry budget is not a delivery guarantee. If the email endpoint has no delivery event available to the application, the worker can prove that the request ended, not that a seller read it. SMS status and email event polling can support a later reconciliation job, but polling is weaker operationally than a push event and should be labeled that way in the data model.

The same boundary applies when a seller has both channels. An email failure does not automatically justify an SMS: the application needs consent, a verified phone number, geographic policy, and a decision about whether an order event is urgent enough to cross channels. SMS anti-abuse geographic fences and per-country spend circuit breakers belong in the business layer. The notification API cannot infer those policies safely.

## Where this recommendation stops applying

This design is aimed at US and EU product notifications. It is not a basis for China email-compliance claims; the Tencent email vendor path is still pending. A team with a China-specific compliance requirement needs a separately verified local delivery path and legal review.

It also assumes an API integration is acceptable. There is no SMTP relay, so an application built around drop-in SMTP must change its sending adapter. There is no managed email OTP interface, and scheduled email sends cannot be canceled, while SMS has a cancel operation. Those gaps matter if the order flow includes login recovery, reservation expiry, or a last-second cancellation promise rather than a simple event notice.

Finally, the absence of webhook event push in both namespaces changes the incident design. A polling reconciler can be adequate at modest volume, but it introduces a delay and another queue to operate. If immediate delivery-state fanout is a hard requirement, an email-first provider with the event model your team already operates may be a better fit, even if it means pairing it with a separate SMS provider such as Twilio. Combining channels under one key is convenient; it is not the same as having one failure domain or one source of truth.

## Decision rule

Choose the narrowest system that can answer four questions for every seller alert: which order caused it, which channel was attempted, what the provider actually returned, and what should happen next. Postmark or Resend are reasonable email-first choices. SendGrid fits teams that need its broader email tooling, and SES fits teams prepared to assemble more of the operational layer in AWS. A combined API is a reasonable choice when custom-domain email and SMS belong in the same integration and public discovery reduces onboarding friction, provided the team accepts API-only sending, polling, and manual deliverability tracking.

The cheapest message is irrelevant if a seller misses the order. The durable decision is the one whose failure states your on-call team can name, alert on, and recover without sending duplicates.

## Sources

References:

- [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://datatracker.ietf.org/doc/html/rfc6376)
- [Postmark Developer Documentation](https://postmarkapp.com/developer)
- [Resend Documentation](https://resend.com/docs)
- [Twilio SMS Documentation](https://www.twilio.com/docs/sms)
- [Amazon Simple Email Service Documentation](https://docs.aws.amazon.com/ses/)
- [SendGrid Documentation](https://docs.sendgrid.com/)
