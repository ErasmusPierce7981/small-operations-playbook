# Compromised Credential Reporting Protects Billing Attribution (With Live API Key Rotation)

Keeping a property-management service online favors a controlled credential handover; suspected key compromise favors immediate containment. Resolve that conflict by treating a confirmed exposure as an incident, recording the credential's billing scope while rotating or revoking it, and shortening any overlap according to the threat. TL;DR: reporting a compromised API key creates an accountable investigation; quietly rotating changes access but leaves billing attribution dependent on whatever evidence already exists. Neither action, by itself, proves who made a charge.

## What page fired, and whose usage is on the bill?

Consider a tabletop production scenario: a property-management service uses a shared outbound credential for maintenance-request processing across 2 managed buildings. Its suspected exposure triggers a decision about rotating that key without interrupting service. These are exercise boundaries, not an account of an observed breach. The incident starts with a question that a green dashboard cannot answer: what page fired, and what evidence connects it to this credential?

An exposure report and an unexplained usage increase justify different initial hypotheses. An exposed secret establishes a reason to contain access; an increase in requests establishes a reason to investigate activity. Calling either one a complete explanation of the bill collapses suspicion, authentication, and financial responsibility into the same claim, which is precisely the shortcut an incident record should prevent.

Preserve that distinction.

For this exercise, the useful record identifies the old credential by a non-secret identifier, its production environment, the two building accounts it could reach, the last known legitimate deployment, and the time the exposure became known. It also names the person responsible for the next decision. Store links or restricted references to evidence, rather than copying a usable key into a ticket. The OWASP secrets-management guidance supports lifecycle management, auditing, rotation, and revocation; the building-level reconciliation proposed here is an application design, not a promise that an upstream service supplies those fields.

The postmortem question is narrower than whether the rotation succeeded: could a reviewer reconstruct which workload had access during each disputed usage interval? A dashboard showing recovered availability would be poor evidence for that question. It measures a different outcome.

## The invariant is a traceable handover

A defensible billing investigation needs a durable relationship between credential identity, workload identity, authorization scope, and time. Rotation should preserve that relationship across the handover. Authentication with the old key only establishes use of that key; when a secret is shared, it doesn't establish which building, worker, or human originated the request. A replacement credential cannot recover distinctions that were never recorded.

Suppose a maintenance worker submits work for building A while a scheduled worker submits work for building B. The application should derive each building's billing account from trusted job ownership, then retain the credential version and an internal operation identifier alongside the usage event. Accepting an arbitrary billing-account label from a request is a concrete trap: the label may look precise in a report while carrying no reliable attribution. These are proposed controls for the exercise, not claims about an existing metering system.

Keep secrets out.

For the incident record, use explicit time boundaries and distinguish observations from estimates. The discovery time is not necessarily the exposure time. A deployment acknowledgment is not necessarily the last request from an old worker. When the exposure boundary is uncertain, record the uncertainty and identify what evidence could narrow it; don't quietly replace it with the time someone clicked a rotation button.

That invariant also changes alert design. Proposed pages should distinguish continued use of a retired credential, authentication failures after cutover, and missing attribution fields on usage events. They need different owners and different responses. The exact thresholds belong to the service's traffic and on-call policy; an invented universal threshold would add confidence without evidence.

## What does reporting a compromised API key add to billing reconciliation?

Reporting means creating an internal incident record and escalating through the relevant security or service-support channel when that party holds evidence or can contain access. It doesn't mean publishing a secret or notifying every tenant before the impact is understood. External notification obligations require their own assessment; this workflow is not a legal determination.

| Response | What it contributes | What remains unresolved |
| --- | --- | --- |
| Quiet rotation alone | Replaces the credential used by the service | Who owns the investigation, which evidence to retain, and how disputed usage will be reviewed |
| Incident record plus rotation | Connects the handover to scope, timing, decisions, and evidence references | Whether available logs can distinguish legitimate activity from misuse |
| Immediate revocation plus incident handling | Removes the old credential's authorization when revocation takes effect | Service recovery and the same attribution investigation |

The middle option is useful when a bounded handover is acceptable, but its value comes from the record and follow-through, not from the word “reported.” Filing a ticket doesn't establish misuse, guarantee a billing adjustment, or make a shared key identify an individual actor. If the relevant upstream logs are unavailable, the record should say so and keep the disputed interval open rather than manufacture a precise allocation.

There is a real operational trade-off here. Separate credentials per workload can make a later investigation more discriminating, while increasing the number of credentials the team must issue, deploy, audit, and retire. Building-level credentials are useful only if the service can enforce those boundaries. Adding labels to one broadly shared secret is cheaper to implement operationally, but labels alone cannot create isolation.

## Put the incident reference in the rotation path

The preventative path should make an unrecorded planned handover difficult. The following Go example is an orchestration boundary: its implementations must persist the incident, deploy through the service's secret-management mechanism, and verify that every intended worker adopted the replacement before retirement. It deliberately passes non-secret references. It isn't a complete credential store or an upstream API contract.

```go
package rotation

import (
    "context"
    "errors"
    "fmt"
)

type Handover struct {
    IncidentID string
    OldKeyID   string
    NewKeyRef  string
}

type Control interface {
    RecordPlan(context.Context, Handover) error
    DeployAndVerify(context.Context, Handover) error
    RevokeOld(context.Context, Handover) error
    RecordComplete(context.Context, Handover) error
}

func PlannedHandover(ctx context.Context, c Control, h Handover) error {
    if h.IncidentID == "" || h.OldKeyID == "" || h.NewKeyRef == "" {
        return errors.New("incident and credential references are required")
    }
    if err := c.RecordPlan(ctx, h); err != nil {
        return fmt.Errorf("record handover plan: %w", err)
    }
    if err := c.DeployAndVerify(ctx, h); err != nil {
        return fmt.Errorf("verify replacement deployment: %w", err)
    }
    if err := c.RevokeOld(ctx, h); err != nil {
        return fmt.Errorf("revoke old credential: %w", err)
    }
    if err := c.RecordComplete(ctx, h); err != nil {
        return fmt.Errorf("record completion after revocation: %w", err)
    }
    return nil
}
```

The ordering matters. A failed verification leaves the planned handover incomplete and requires an explicit containment decision; a failed completion write occurs after revocation and must not be interpreted as permission to restore the exposed credential. Persist step outcomes in the implementation so retries reconcile actual credential state instead of assuming every returned error means nothing happened. A timeout is especially dangerous evidence to overread.

Before deployment, exercise the failure boundaries: a worker that keeps its old credential, a revoked key still configured in a scheduled job, and a completion-record failure after successful revocation. Verify service behavior and the resulting evidence trail together. A test that checks only successful requests can miss the billing question entirely. For queued maintenance work, ensure retry handling preserves the logical operation identity so an authentication transition doesn't turn one job into unexplained duplicate usage.

## When continuity must yield to containment

This planned path doesn't apply unchanged to active misuse. Waiting for a ticket write or a fleet-wide acknowledgment can leave an attacker authorized, so emergency revocation needs an independent path, with evidence collection and record completion proceeding alongside containment. Likewise, overlapping credentials are possible only when the underlying authorization system supports them. If it doesn't, a no-downtime handover requires a different service design or a consciously accepted interruption.

The billing decision rule is simple: report suspected compromise when somebody must investigate exposure or disputed activity, and retain enough context to explain the credential transition. Restore service with a fresh credential, reconcile the disputed usage against available workload evidence, and document the intervals that remain uncertain. A successful rotation closes an access path. Closing the incident requires an accountable explanation of what the evidence can support.

## Sources

References:

- OWASP Secrets Management Cheat Sheet — guidance on secret lifecycle management, auditing, rotation, and revocation: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
