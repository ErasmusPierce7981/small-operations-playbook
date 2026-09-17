# Node.js Mobile Auth 2026: 4 Refresh-Token Rotation Controls for Healthtech

Move refresh-token rotation into the Node.js backend before migrating social sign-in, and treat every refresh as a single-use state transition rather than a stateless JWT check. For a healthtech B2B mobile app adding Google and GitHub login, the deciding constraint is incident containment: a copied refresh token needs a bounded life, while a delayed mobile retry must not turn an ordinary network interruption into an account-wide lockout.

**Short answer:** issue a short-lived access token and an opaque, high-entropy refresh token; store only a digest of the refresh token; group refresh records into a family; replace the current token atomically on every successful refresh; and revoke the family when an already-consumed token is replayed outside a narrow, documented retry path. Social-provider tokens do not belong in the mobile session format. They prove something at an upstream boundary; they are not the application's refresh tokens.

The useful page is not "login failures increased." It is "refresh-family replay crossed the threshold for one tenant after a consumed token arrived from a new device context." The first message hides the failure mode. The second gives the responder somewhere to start.

## How should a mobile app login API rotate a refresh token?

A mobile client can send a refresh request, lose the response while changing networks, and retry with the token the server has just consumed. A strict implementation sees reuse and revokes the whole family. That contains theft, but it punishes an expected transport failure. Accepting old tokens indefinitely avoids that false positive and defeats rotation.

The server cannot infer intent from the old token alone. The same value can arrive because the legitimate app missed a response or because an attacker copied it. Build that ambiguity into the protocol: serialize rotation per family, allow an exact duplicate only inside a short measured retry window, and never retain successor tokens in plaintext merely to make retries convenient. An encrypted, expiring response envelope keyed by an idempotency digest is possible. Requiring interactive authentication after an ambiguous refresh stores less replayable material but creates more sign-ins.

Consider the exact race. Request A locks generation 7, writes generation 8, commits, and returns a new token; the radio drops before the app receives it. Request B then presents generation 7. If B arrived while A still held the row lock, it must wait and re-read the committed state rather than validate against a stale snapshot. If B arrived afterward, the server needs the same answer it would have produced after waiting: classify the request through the retry policy, not through timing luck. A process-local lock cannot preserve that rule across two Node.js instances, and accepting both requests would create two generation-8 descendants. The database transition, response-recovery policy, and replay alert therefore form one design decision even though they live in different parts of the system.

Timing is not identity.

There is no universal grace period. Measure retry delay in your own telemetry and choose the smallest interval that covers expected duplicate delivery. A grace window is a security decision.

Keep four controls together:

1. One active refresh generation per session family.
2. An atomic compare-and-swap that consumes generation N and creates N+1.
3. Family revocation when a consumed generation appears without a valid duplicate-request record.
4. A session version or revocation timestamp checked before minting another access token.

Miss one and two holders may advance the same session while a dashboard stays green. Bad silence.

## Put the trust boundary behind Node.js

Google and GitHub are upstream identity sources here. The backend completes the authorization-code flow, validates the result under OAuth or OpenID Connect rules, maps the external subject to an internal identity, and issues an application session. The mobile app uses Authorization Code with PKCE. Native apps use an external user-agent rather than an embedded one, and the verifier and challenge bind the authorization request to its token exchange.

Do not link records merely because two providers report the same email string. Use issuer plus stable provider subject as the external key, then attach it to an internal user through an explicit authenticated linking flow. Tenant membership, account status, and authorization remain local decisions. Social sign-in says who authenticated upstream; it does not decide which clinic, workspace, or patient data that person may access.

Migration off a managed provider sharpens the boundary. Preserve internal user IDs and tenant memberships, add external-identity mappings, and allow old and new validation paths during a measured overlap. Existing sessions can expire under the old issuer while new logins enter the new family store, provided every request has one unambiguous validator and rollback cannot let both systems rotate the same family.

Self-managed rotation has a real limitation: a team without round-the-clock ownership, transactional storage expertise, key management, and a rehearsed revocation path should not build this state machine merely to remove a dependency. A managed identity service can reduce operational surface, while a standards-based library can reduce protocol mistakes, but either choice still needs local authorization and migration tests. The trade-off is control against incident burden, not a feature-count contest.

Choose that burden deliberately.

The critical transition belongs in a database transaction, not a process-local mutex. This Go-shaped example omits duplicate-response recovery deliberately; adding it without encrypted, expiring storage would encourage plaintext successor retention.

```go
func Rotate(ctx context.Context, db *sql.DB, in RotateInput) (RotateResult, error) {
	tx, err := db.BeginTx(ctx, &sql.TxOptions{Isolation: sql.LevelSerializable})
	if err != nil {
		return RotateResult{}, err
	}
	defer tx.Rollback()

	family, err := loadFamilyForUpdate(ctx, tx, in.FamilyID)
	if err != nil {
		return RotateResult{}, err
	}
	if family.Revoked || in.Now.After(family.AbsoluteExpiry) {
		return RotateResult{}, ErrReauthenticationRequired
	}
	if subtle.ConstantTimeCompare(family.CurrentDigest, in.TokenDigest) != 1 {
		if err := revokeFamily(ctx, tx, family.ID, in.Now, "refresh_reuse"); err != nil {
			return RotateResult{}, err
		}
		if err := tx.Commit(); err != nil {
			return RotateResult{}, err
		}
		return RotateResult{}, ErrReauthenticationRequired
	}

	nextRaw, nextDigest, err := newOpaqueRefreshToken()
	if err != nil {
		return RotateResult{}, err
	}
	if err := advanceGeneration(ctx, tx, family.ID, family.Generation, nextDigest, in.Now); err != nil {
		return RotateResult{}, err
	}
	access, err := mintAccessToken(family.UserID, family.TenantID, in.Now)
	if err != nil {
		return RotateResult{}, err
	}
	if err := tx.Commit(); err != nil {
		return RotateResult{}, err
	}
	return RotateResult{AccessToken: access, RefreshToken: nextRaw}, nil
}
```

A transaction serialization failure is retryable infrastructure work, not evidence of theft. Those outcomes need different metrics and different client behavior.

## Test the incident, not the happy path

A unit test that exchanges one valid token proves little about rotation. Race two requests carrying the same generation and assert that only one advances the family. Commit a rotation, discard the simulated response, and submit the old token again; the result must match the documented retry policy. Present a consumed token after that allowance and verify family revocation.

Then cross migration boundaries. Verify that identities from both providers can be explicitly linked without email auto-merging. Verify that disabling a user or removing tenant membership prevents refresh from producing fresh authorization. Verify that a provider outage does not invalidate an existing application session unless policy requires reauthentication.

| Exercise | Invariant | Observable signal |
| --- | --- | --- |
| Concurrent refreshes | At most one generation advances | One success; one conflict or defined retry |
| Consumed-token replay | Family cannot continue | Reuse event tied to family and tenant |
| Lost response | Behavior matches retry contract | Duplicate classification, not generic failure |
| User disabled | No new access token | Denial reason without secret material |
| Legacy issuer drain | One validator per session | Counts by issuer and expiry cohort |

Use synthetic accounts with no clinical data. Exercise both providers, but alert on application invariants: replay, rotation conflicts, unexpected issuer acceptance, and refresh success falling sharply by app version. Provider login errors belong in diagnostic context; they should not be the only page.

## Deploy and roll back without forking sessions

First ship recognition of the new metadata and telemetry for issuer, family, generation, and outcome. Enable new issuance for an internal cohort, then one tenant cohort, while legacy sessions drain. Move refresh traffic only after concurrency and lost-response tests pass against the production database topology.

Rollback is safe only if the old path cannot accept and rotate a family already owned by the new path. Use an issuer or migration-state marker enforced by both systems, and make ownership transfer one-way unless an explicit compensating migration runs. Otherwise two valid successors can emerge from one logical session, and responders cannot identify the authoritative lineage.

Derive thresholds from baseline traffic, not invented percentages. Page on sustained replay crossing tenant or device boundaries, inability to commit rotations, and acceptance of a retired issuer. Ticket isolated reauthentication and expected duplicate delivery unless context changes the risk. Ask what page fired. If the answer is only "auth is down," instrumentation is unfinished.

Support revoking one family, every family for one user, or sessions issued before a timestamp. Account-wide revocation is broad; audit it and do not automate it for every timeout. Already-issued access tokens remain usable until expiry unless resource servers check session state, so select their lifetime with that exposure understood.

## Record the choices the dashboard cannot show

Write down access-token lifetime, idle and absolute refresh expiry, retry logic, entropy source, digest method, revocation rule, identity-linking proof, issuer overlap, and rollback ownership. Attach a test to every invariant. State which events page, which open a ticket, and which are counted.

The practical decision in 2026 is whether the team can explain and test the whole session state machine, operate through an upstream-provider failure, and reverse migration without creating two authorities. Atomic state, bounded replay handling, local authorization, and rehearsed revocation provide that evidence. A feature checklist does not.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://www.rfc-editor.org/rfc/rfc8252
- https://www.rfc-editor.org/rfc/rfc7636
- https://www.rfc-editor.org/rfc/rfc6749
- https://openid.net/specs/openid-connect-core-1_0.html
