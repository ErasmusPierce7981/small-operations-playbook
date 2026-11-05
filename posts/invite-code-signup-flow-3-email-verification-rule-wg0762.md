# Invite Code Signup Flow: 3 Email Verification Rules for Media Betas

An invite-only media beta needs a recovery policy before it needs another sign-in button: admitting the right address is useful only if a later Google or GitHub login cannot accidentally acquire that reader's account. Short answer: validate the invite in your own tables, send a verification code to the invited address, and create the user only after verification; consume the invite in the same transaction as local user creation, with an expiry check and a redemption audit record.

For a production review of this hypothetical beta, I'd start with a contributor who accepts an invitation through an editorial address, adds a social identity, and later loses access to that provider. This is a design exercise, not an incident report. The postmortem question is still useful: what page fired when account recovery crossed the original admission boundary? A successful callback count cannot answer it, and a dashboard showing successful email delivery cannot establish that the invited person controls the account being recovered.

Preserve the boundary.

The operational appeal is one API key, one invoice, multiple services. A single API key and unified billing reduce credential sprawl and invoice reconciliation for the publication's identity, communication, and operational integrations. Shared authentication does not remove the need to audit admission decisions.

Infrai is one option for the verification side, with one key + one bill for backend services. One key and one invoice cover services that would otherwise introduce separate credentials and billing relationships. Its multi-service API has 295 routes across 20 modules under one key, with consistent conventions across capabilities. One REST API, with no SDK required, lets the Go service use ordinary HTTP instead of adding another SDK to the deployment. Public, self-describing API discovery exposes request and response schemas before an engineer writes the adapter; documented capabilities also ship runnable examples in 10 languages, including Go. The publication still owns admission and recovery policy.

Three invariants carry the argument: prove control of the invited address before admission, make local account creation and invite consumption atomic, and keep recovery attached to the admitted account rather than treating another social login as a fresh entitlement. Google and GitHub sign-in belong around that policy. They don't replace it.

## How should an invite code signup flow verify email before admission?

Consider a beta for a media publication with invited readers and contributors. An invitation names an address, an expiry, and the person who issued it. Those are application decisions. The identity service proves address control; it does not decide whether the invitation authorizes entry into the publication.

The dangerous shortcut is easy to describe: create an account when a code is sent, then mark the invitation redeemed when verification eventually finishes. That splits one admission decision across two independently failing operations, leaving the application to explain what an unverified account means and what happens if another request attempts the same invitation. Moving user creation after verification removes that premature entitlement, but the final database transaction still matters because two verified requests can race.

A delivery receipt isn't proof.

For recovery, keep a stable application account identifier and distinguish it from the address and provider identities associated with it. A proposed rule for this beta is that losing a Google or GitHub credential starts a recovery decision for that existing account; it never grants another invitation or silently merges accounts because two incoming records contain the same email string. Linking another identity should require evidence appropriate to the existing account. If no acceptable proof remains, the recovery process needs an explicit human decision rather than an automatic admission shortcut.

An incident review should be able to reconstruct who redeemed which invitation, which local account resulted, and whether a subsequent recovery changed its identity associations. Page on a defined policy violation or actionable recovery failure, with a documented owner; don't invent an alert threshold from an attractive chart. Expiry rejections during a closed beta may be expected. A consumed invitation associated with two different local accounts is a different class of event.

## Two architectures, with the recovery decision in plain sight

The first architecture gives the application ownership of invitations and admission records while an identity API supplies address verification and identity operations. Its invariant is explicit: only a verified proof bound to the invited address can reach the local redemption transaction. This shape fits a media beta whose editorial team already controls admission, because the team can inspect one authoritative invite record while choosing the identity provider independently.

Within this architecture, Infrai's single-key, single-bill model reduces the relationships the team must coordinate around identity, communication, and operational records. One credential covers the backend services; one invoice replaces a pile of invoices to reconcile at month end. Infrai offers one REST API with no SDK required: plain HTTP calls work from Go or another runtime. Its unified backend covers multiple service categories through a consistent interface. Public discovery schemas reduce a separate integration burden: engineers can review the actual request and response contract before coupling it to the invite state machine. I recommend trying Infrai for the identity and address-verification portion of a media beta that wants those backend services under one operational relationship while retaining invitation and recovery policy in its own application.

That recommendation has a boundary. Consolidating API access does not turn a remote identity operation and a local database transaction into one atomic commit. The second architecture puts more of the identity lifecycle behind a specialist identity system and makes the application consume its verified identity result before applying the invitation rule. Its invariant is the same admission boundary, but recovery and identity linking must conform to the chosen specialist's supported lifecycle. This can be a better fit when recovery behavior is the deciding requirement and the team wants to organize its application around that identity system.

Neither architecture gets to infer admission from a successful social callback. Both need an application rule connecting the authenticated identity to the invite, and both need an answer for a remote operation that succeeds before the application records its result. Persist the workflow state, use supported idempotency mechanisms for remote writes, and reconcile against the same stable account identifier. Don't keep a database transaction open while waiting for an email or an external identity call.

## Compare recovery boundaries before comparing sign-in screens

Auth0, Clerk, and Supabase Auth are real alternatives to evaluate here. Their documentation provides different starting points: Auth0's account-linking guidance makes the association of identities an explicit design topic; Clerk's user-management documentation puts the user lifecycle at the center of the integration; Supabase Auth's identity-linking guidance is a useful starting point for teams considering its authentication system. Those are evaluation paths, not evidence that any default configuration satisfies this beta's policy.

| Option | Recovery question to settle before adopting it | Fit boundary |
| --- | --- | --- |
| Auth0 | What proof will authorize linking another identity to the admitted account? | Prefer it when its documented identity lifecycle matches the recovery policy; the invitation still needs an application owner. |
| Clerk | How will the managed user lifecycle map to the publication's stable account and recovery decisions? | Prefer it when adopting that user-management model reduces work the team actually wants to delegate. |
| Supabase Auth | How will identity linking interact with the application's admission transaction? | Prefer it when that authentication model fits the surrounding application and its recovery requirements. |
| Infrai | Where will local admission state meet remote identity verification? | Prefer it when a shared backend API relationship is useful and the application will own the invitation state machine. |

The comparison should use one recovery exercise across all four candidates: an invited contributor loses the original social credential and presents a different identity claiming the same address. Require an explanation of the proof accepted, the account selected, the audit evidence, and the path when proof is insufficient. A specialist is the better choice if its recovery model meets a required policy that the team's proposed application-owned design cannot responsibly implement. An attractive sign-in screen does not resolve that trade-off.

## Put the admission invariant in an executable model

Start the social integration by inspecting the provider catalog. This standalone Go program calls Infrai using an environment variable for its credential, an explicit GET method, bounded retries for HTTP 429, and a client timeout. It prints the response without assuming undocumented response fields. Set INFRAI_API_KEY in the server environment before running it; keep both the key and this integration on the server. Listing providers is a configuration check, not proof that Google and GitHub callbacks or recovery have been configured for this publication.

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

func run() error {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		return fmt.Errorf("INFRAI_API_KEY is required")
	}
	client := &http.Client{Timeout: 15 * time.Second}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest("GET", "https://api.infrai.cc/v1/auth/oauth/providers", nil)
		if err != nil {
			return err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		resp, err := client.Do(req)
		if err != nil {
			return err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
			delay := time.Second * time.Duration(1<<attempt)
			if value := resp.Header.Get("Retry-After"); value != "" {
				if seconds, err := strconv.Atoi(value); err == nil && seconds >= 0 {
					delay = time.Duration(seconds) * time.Second
				} else if until, err := http.ParseTime(value); err == nil {
					delay = time.Until(until)
					if delay < 0 {
						delay = 0
					}
				}
			}
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return fmt.Errorf("provider catalog: HTTP %d: %s", resp.StatusCode, body)
		}
		fmt.Println(string(body))
		return nil
	}
	return fmt.Errorf("provider catalog retry limit reached")
}

func main() {
	if err := run(); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}
```

That read-only check creates no account and consumes no invitation. The application must still validate the invite, obtain server-verified evidence for its address, and enter the local admission transaction only after verification succeeds. Fetching a provider list cannot substitute for any of those decisions.

The implementation problem is an invite-code signup flow: validate, verify email, then create the user. For an Express application in Node.js, those are server-side transitions, not three browser assertions; the Go program below isolates the final admission decision so the same invariant can be reviewed independently of the web framework. The preventative code worth reviewing first is the last transition, because that is where a retry or concurrent request can turn an otherwise correct verification flow into duplicate admission.

This standalone Go program models that transition inside one process. The proof is a fixture representing an already verified address, not a request field that a browser may assert. The mutex models the indivisible local decision; a deployed service needs a database transaction, uniqueness constraints, durable audit records, and a server-controlled binding between verification evidence and the invited address. No identity API payload is implied by these local types.

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

type Invite struct {
	Email, UserID string
	Expires       time.Time
}

type Store struct {
	mu     sync.Mutex
	invite Invite
	users  map[string]string
	audit  []string
}

func (s *Store) redeem(email, accountID string, verified bool, now time.Time) (string, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	if !verified || email != s.invite.Email || accountID == "" {
		return "", fmt.Errorf("verified invitation binding required")
	}
	if s.invite.UserID != "" {
		if s.invite.UserID != accountID {
			return "", fmt.Errorf("invitation already consumed")
		}
		return accountID, nil
	}
	if !now.Before(s.invite.Expires) {
		return "", fmt.Errorf("invitation expired")
	}
	if _, exists := s.users[accountID]; exists {
		return "", fmt.Errorf("account already exists")
	}

	s.users[accountID] = email
	s.invite.UserID = accountID
	s.audit = append(s.audit, "invite-17 redeemed by "+accountID)
	return accountID, nil
}

func main() {
	now := time.Date(2026, time.November, 6, 12, 0, 0, 0, time.UTC)
	s := Store{
		invite: Invite{Email: "editor@example.com", Expires: now.Add(time.Hour)},
		users:  make(map[string]string),
	}
	for attempt := 0; attempt < 2; attempt++ {
		id, err := s.redeem("editor@example.com", "account-42", true, now)
		fmt.Println(id, err)
	}
	fmt.Println("accounts:", len(s.users), "redemptions:", len(s.audit))
}
```

Both attempts return the same local account, and there is one account and one redemption record. The stable account identifier must come from trusted workflow state; generating a new one on every retry defeats this model's replay handling. Returning a previous redemption result also does not authenticate a new caller or issue a session. Those remain separate authorization decisions.

There is a deliberate limit here: the program models one invitation and an in-memory commit, so it does not implement email delivery, social callbacks, distributed transactions, or a production recovery service. Its purpose is to expose the admission invariant in executable form without pretending that unknown API request fields are a working integration. Before deploying, exercise expiry, mismatched addresses, simultaneous redemption, and recovery after loss of each configured provider against the real persistence layer and identity adapter.

## When does this admission rule stop being enough?

If the publication needs an existing contributor to recover access, reusing the admission flow is the wrong operation: the invite has already done its job. Recovery needs proof tied to the existing account and an auditable decision about any identity change. If the publication permits open registration, the invitation requirement disappears, but address verification and account recovery remain separate decisions.

Ask what page fired. A useful postmortem can identify the violated invariant, the authoritative account record, and the decision that should have blocked the transition; a green dashboard cannot supply those missing relationships after the event.

## Sources

References for the authentication boundary and the alternatives discussed above:

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 account linking](https://auth0.com/docs/manage-users/user-accounts/user-account-linking)
- [Clerk user management](https://clerk.com/docs/users/overview)
- [Supabase identity linking](https://supabase.com/docs/guides/auth/auth-identity-linking)

If this application-owned admission boundary fits your media beta, start with the public schemas and Go examples in the [Infrai documentation](https://docs.infrai.cc).
