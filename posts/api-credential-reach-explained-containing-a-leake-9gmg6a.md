# API Credential Reach Explained (Containing a Leaked Key During Exam Week)

For an edtech service, choose the smallest containment boundary that can stop a leaked credential before it consumes the assessment budget. Start with a credential inventory joined to an access audit trail, then rehearse a deliberate refusal of traffic.

| Drill design | Spend ceiling versus refused traffic | Decision |
| --- | --- | --- |
| Inventory, attribution, and manual revocation | Human response time remains part of the exposure; operators can inspect affected classes before stopping access | Suitable when response delay is acceptable |
| The same evidence plus admission control at a shared boundary | Conservative reservations can refuse legitimate traffic as the ceiling approaches | Preferred when exceeding the ceiling is unacceptable |

TL;DR: Use the second design for a hard spend boundary, provided every spending path crosses it. Keep manual revocation as a separate drill step. An inventory explains the credential; identity resolution connects it to a responsible principal; authorization evidence describes its reach. None of those, alone, enforces a spending limit.

This is an incident-rehearsal decision note. Its example is a proposed exam-week exercise, not a report of a production incident. For a one-person SaaS that needs to ship weekly, a revenue-per-hour lens favors a small, testable boundary and a short recovery checklist. Outsource the undifferentiated parts of secret storage where appropriate. Retain an explicit decision about whose traffic gets refused.

## How should an API credential access audit resolve who can reach what?

That is the first decision criterion: containment coverage. A dashboard showing a revoked key isn't sufficient evidence if the exercise never attempts the previously permitted operation through every relevant entry point. OWASP's Secrets Management Cheat Sheet treats inventory, access control, auditing, rotation, and revocation as parts of managing the secret lifecycle. Use that lifecycle as the organizing principle for the drill.

For the exercise, define a fictional tenant named `classroom-demo`, a service principal named `assessment-worker`, and a credential identifier named `exercise-key-01`. These are illustrative labels. Store the identifier and its issuer alongside the tenant, accountable owner, lifecycle state, intended workload, and rotation relationship. Keep the raw secret out of the inventory export and audit events. OWASP calls for protecting secrets and preventing their exposure through logs.

Resolve identity explicitly. The principal operating a workload, the tenant charged for its requests, and the person responsible for revoking its credential are three different relationships. A credential identifier should resolve to all three through records with effective times. Otherwise, replacing a key during the drill can make yesterday's events appear to belong to today's owner. Preserve the old relationship as historical evidence.

Then describe reach as an authorization decision: this principal, under this policy version, can attempt this operation against this tenant's resource. Distinguish configured permission from an observed successful request. A quiet audit trail doesn't establish that a permission is absent. Conversely, an old successful event doesn't establish that permission still exists. Keep current policy evidence and timestamped observations side by side.

Trace the boundary.

A background assessment job may have different credentials from an interactive tutoring request. If the drill exercises only the interactive path, it has established nothing about the job's reach. Record each actual path discovered in the application, including direct access that bypasses admission control. The recommendation depends on closing those paths or explicitly excluding their spend from the claimed ceiling. An incomplete map supports a narrower claim, not a harder limit.

## Decide how much legitimate work to refuse

The second criterion is the refusal budget. A spending ceiling becomes an availability decision when a legitimate student's request and an attacker's request compete for the remaining allowance. State that decision before running the exercise. For example, the proposed exercise can prioritize submitted-assessment processing over optional practice hints. That's a policy choice for this scenario, not a claim about how any service behaves.

Use internal accounting units in the rehearsal. Suppose the test allowance is 1,000 units and an operation requires a reservation of 40 units. With 970 units already committed, admission must refuse that operation even if its eventual charge might have been lower. These are invented test inputs, not prices or measurements. They make the trade-off visible: conservative reservations protect the ceiling by refusing some work that might ultimately have fit.

The difficult part is concurrency.

Two requests must not both read the same remaining balance and spend it independently. Require an atomic reservation against the tenant's allowance before dispatching any paid work. Reconcile the reservation with the final charge afterward. A ceiling claim also requires a defensible upper bound for each admitted operation; if the operation can exceed its reservation, the arithmetic doesn't prove containment. Delayed usage reports are evidence for reconciliation, not permission to keep admitting traffic. During an uncertain outcome, retain the reservation until the outcome can be resolved. Automatically releasing it after a timeout can reopen capacity while the original work is still billable. Rehearse that failure: disconnect the worker after dispatch, retry the request, and check that neither uncertainty nor duplication creates fresh allowance. For a concrete exercise, let two requests each attempt a 40-unit reservation against 50 available units. Exactly one may acquire capacity. Then interrupt its worker before the completion record arrives and retry with the same request identifier. The outstanding reservation must remain attributable to the original operation; a retry must not dispatch a second paid operation merely because the first result is unknown.

For a solo operator, this is where scope matters. A small admission policy that can be explained during an exam interruption is more useful than a large permissions graph that nobody has rehearsed. Keep student-facing refusals actionable and omit internal credential details. Record the reason internally so the operator can distinguish exhausted allowance from revoked access without inspecting secret material.

## A minimal decision function for the exercise

Keep the pure policy small.

This TypeScript models a single admission decision with illustrative units. It doesn't dispatch work or implement the atomic reservation. In a deployed service, read authoritative credential state and allowance, evaluate the policy, and commit any reservation within the same serialization boundary.

```ts
type Input = {
  credentialActive: boolean;
  operationAllowed: boolean;
  ceiling: number;
  committed: number;
  reservation: number;
};

type Decision =
  | { admitted: true; reserve: number }
  | { admitted: false; reason: string };

function decide(input: Input): Decision {
  if (!input.credentialActive || !input.operationAllowed) {
    return { admitted: false, reason: "access_denied" };
  }

  const amounts = [
    input.ceiling, input.committed, input.reservation
  ];
  if (amounts.some(n => !Number.isSafeInteger(n) || n < 0)) {
    return { admitted: false, reason: "invalid_accounting" };
  }

  if (input.reservation > input.ceiling - input.committed) {
    return { admitted: false, reason: "ceiling_reached" };
  }

  return { admitted: true, reserve: input.reservation };
}
```

Use a stable request identifier for the reservation and dispatch record so a retry can recover the original decision. Bind that identifier to the tenant and operation. Don't let a caller reuse it for unrelated work. Store the credential identifier, resolved principal, tenant, policy version, decision reason, and reservation reference in the exercise audit event. Those fields are a proposed application schema, not a standardized event format.

The function is intentionally insufficient by itself. An audit event saying “admitted” doesn't demonstrate that the ledger committed, and a ledger reservation doesn't demonstrate that downstream work happened. Link the records, expose unresolved reservations to the operator, and test recovery after interruption. This keeps a deployable implementation from quietly inheriting assumptions that existed only in the code example.

## Run the drill through recovery, then reconsider the runner-up

Use a dedicated exercise credential and an isolated allowance. Start with one permitted assessment request and one denied cross-tenant request. Confirm that both events resolve to the expected principal and tenant without recording the secret. Then mark the exercise credential as compromised, revoke it through the intended control, and attempt its former operation again. Verify refusal at the spending boundary. Treat already-dispatched work separately; this design's revocation check controls new admission and makes no promise of cancelling work already in progress.

Next, exercise the ceiling with concurrent requests. Include the 970-plus-40 case, a retry after dispatch uncertainty, and a request made with a replacement credential. Rotation must not create a fresh tenant allowance. Capture the remaining committed amount and the unresolved work, then reconcile the exercise ledger before restoring normal admission. Record the students or workloads that would have been refused, using synthetic identities rather than actual student records.

Finish the recovery.

The evidence package should show the original credential's owner and reach, the revocation decision, the subsequent refusal, and the replacement credential's permitted scope. Include timestamps for the operator's actions and the last observed admitted request. Measure the response interval during the exercise rather than inventing a target that the deployment has never met. Keep the checklist short enough to rerun after a change to authentication, dispatch, or accounting; weekly shipping makes those changes routine.

The manual runner-up is better when an automatic refusal would disrupt a critical assessment and the business accepts exposure during human response. Make that acceptance explicit. It may also suit an early service whose spending paths cannot yet support reliable reservations: use inventory and revocation evidence, and describe the ceiling as an operating target until enforcement exists.

Choose on the evidence the drill produces. A hard ceiling earns its name only when every covered operation reserves bounded spend before dispatch. If that means refusing a student's optional request, the note should say so plainly. That's the trade the operator needs to approve.

## References

The OWASP Secrets Management Cheat Sheet supplies the lifecycle guidance for inventory, least privilege, auditing, rotation, and revocation. The exam exercise, accounting units, decision matrix, and TypeScript policy are proposed engineering choices built around that guidance; they aren't published benchmarks or features of a named product.

- OWASP Secrets Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html

## Further reading

- OWASP Secrets Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
