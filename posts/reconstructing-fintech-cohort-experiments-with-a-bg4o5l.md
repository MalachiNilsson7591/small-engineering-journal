# Reconstructing Fintech Cohort Experiments with a Feature Flag Kill Switch (Before Replay)

| Choice | What it can stop | Evidence needed for reconstruction |
| --- | --- | --- |
| Runtime feature flag kill switch | New entries into an experimental path after each worker observes the change | Decision revision, assigned cohort, effective variant, and a control event |
| Deployment rollback | Execution of a release after the replacement becomes effective | Release identity, deployment timing, and the decisions made during the transition |
| Pause and reconcile | Further work at an explicitly gated processing boundary | Accepted work, completed effects, and the reconciliation boundary |

TL;DR: For a fintech experiment comparing tenant cohorts, put a feature flag kill switch immediately before the experimental path. Preserve the assigned cohort when the effective variant changes. Record which control revision each request actually used. This is the least complex option when the stable path remains safe and available; use deployment rollback when the defect also affects that path.

Stop new exposure. Preserve the evidence.

The useful outcome is a defensible answer to a narrow incident question: did the treatment cohort improve after containment, or did its requests quietly start running the control behavior? A dashboard grouped only by assigned cohort cannot distinguish those explanations. Neither can a timestamp saying that someone clicked “disable.”

This note uses a hypothetical fintech eligibility experiment. The labels, revisions, and request identifiers below are illustrative, not measured production results. The decision axis is incident reconstruction: contain the experiment while keeping enough evidence to compare tenant cohorts afterward.

## What must a production feature flag kill switch preserve?

Keep assignment and execution separate. A tenant can remain assigned to treatment while its next request takes the stable path because the switch is disabled. Rewriting that tenant's assignment to control would erase the distinction that the incident review needs.

An illustrative request might carry tenant key `tenant_17`, assigned cohort `treatment`, effective variant `control`, and flag revision `8`. Its reason is `disabled`. A preceding request from that same tenant might have used revision `7` and executed treatment. Those two records describe a transition without pretending that every worker changed behavior at the same instant.

The first decision criterion is therefore evidence continuity. Define a decision record that connects an opaque tenant key, experiment identifier, request identifier, assigned cohort, effective variant, release identifier, and observed revision. Record the reason for the decision as well. A control assignment and a disabled treatment assignment both run control, but they answer different questions during reconstruction. Keep the control change as a separate event: actor, requested state, prior revision, new revision, and reason. Join it to request decisions through the revision. Wall-clock proximity alone should not be the joining rule.

This has a privacy boundary. OWASP's logging guidance covers sensitive data exclusion, event attributes, access controls, and protection against log injection. For this experiment, use an opaque tenant key and keep account details, payment payloads, access tokens, and credentials out of decision events. Treat event fields as an allowlist. An opaque key can still be sensitive when it is linkable, so its mapping and retention need access controls too.

## Containment depends on where the gate sits

The second criterion is the boundary at which a change takes effect. Evaluate the switch before entering the experimental operation, then retain that decision for the operation. Do not evaluate it once for the request log and again for the actual branch: a change between those evaluations could make the evidence disagree with the execution.

A kill switch is prospective. Its limitation is that it can't undo a transfer, cancel an already accepted external operation, or repair persisted state merely by selecting the stable branch. For queued work, decide explicitly whether the gate controls enqueueing, execution, or both. Include the decision at the boundary you intend to investigate. Otherwise, a queue can continue draining treatment work after the web tier reports that treatment is disabled.

Choose the acceptable propagation interval before selecting the distribution mechanism. The trade-off with a cached snapshot is reduced dependency on the control service during requests in exchange for workers using an older revision until refresh. A remote evaluation on every request creates a different availability dependency. Neither choice supplies an instantaneous global cutoff by itself.

For a one-person SaaS, the revenue-per-hour lens favors a small, rehearsed control surface. Ship weekly, but make the experiment's stable path independently usable. Outsource the undifferentiated work of storage and event transport where that reduces operating effort; retain ownership of the decision schema, authorization boundary, and replay procedure. Those are what make containment explainable.

Budget observability by the evidence needed.

Repeating an entire financial payload on every evaluation adds exposure and investigation work without making the cohort comparison more defensible.

## A small control API and a decision snapshot

The following TypeScript example isolates two mechanics: a revision-checked disable operation and an immutable decision for each request. It is suitable as a focused Node.js service example with a Request/Response adapter. The adapter must derive `authorized` from authenticated operator identity and permission checks; it must never accept that value from the request body.

The state is deliberately process-local. Revision `7` is illustrative. This example demonstrates the control contract, not cross-worker distribution or durable auditing. It isn't suitable for coordinating multiple production workers; that requires shared authoritative state and an adoption policy.

```ts
type Cohort = "control" | "treatment";
type Snapshot = Readonly<{ enabled: boolean; revision: number }>;
type Decision = Readonly<{
  assigned: Cohort;
  effective: Cohort;
  revision: number;
  reason: "assigned" | "disabled";
}>;

let snapshot: Snapshot = { enabled: true, revision: 7 };

export function decide(assigned: Cohort): Decision {
  const observed = snapshot;
  return {
    assigned,
    effective: observed.enabled ? assigned : "control",
    revision: observed.revision,
    reason: observed.enabled ? "assigned" : "disabled",
  };
}

// Bind this handler to a dedicated operator-only endpoint.
export async function disable(
  request: Request,
  authorized: boolean,
): Promise<Response> {
  if (!authorized) return new Response(null, { status: 403 });
  if (request.method !== "POST") {
    return new Response(null, {
      status: 405,
      headers: { Allow: "POST" },
    });
  }

  let body: unknown;
  try {
    body = await request.json();
  } catch {
    return new Response(null, { status: 400 });
  }
  if (
    typeof body !== "object" || body === null ||
    !("expectedRevision" in body) ||
    !Number.isSafeInteger(body.expectedRevision)
  ) {
    return new Response(null, { status: 400 });
  }
  if (body.expectedRevision !== snapshot.revision) {
    return Response.json({ revision: snapshot.revision }, { status: 409 });
  }

  if (snapshot.enabled) {
    snapshot = { enabled: false, revision: snapshot.revision + 1 };
  }
  return Response.json(snapshot);
}
```

A disable request with `{ "expectedRevision": 7 }` returns the disabled snapshot at revision `8`. Repeating that stale request returns a conflict; the operator should read the current revision and reconcile intent. Sending the current revision while already disabled leaves it unchanged. Keep re-enabling as a separate, deliberate action.

At the business boundary, call `decide(assigned)` once. Use its `effective` field to choose the implementation, and attach that same decision to the request's evidence record. The code returns a decision; it does not claim that a financial operation succeeded. Record completion or failure separately under the same operation identifier.

A production implementation needs an authoritative store that commits the state change and its control event together, plus distribution to workers. Use a conditional update against the expected revision there. Include authenticated actor and reason in the durable event. The response should describe the committed control state; worker observations provide evidence of adoption. A process-local response cannot establish fleet-wide containment.

## Reconstruct the cohort comparison before shipping again

Start with decisions, then join outcomes. Partition the experiment by assigned cohort, effective variant, observed revision, and release. Keep tenant identity available under the relevant access controls so that one unusually active tenant does not silently stand in for an entire cohort. Define the observation window and comparison unit before drawing a conclusion.

That gives two distinct views. Assignment describes the original experiment grouping. Effective execution describes what happened around containment. Combining those into a single “treatment” series hides the period when treatment-assigned tenants were already receiving stable behavior.

Missing evidence matters. Don't fill gaps with assumptions.

An operation with no completion record is unresolved, not automatically successful. A worker still reporting revision `7` after the control plane commits revision `8` needs investigation against the chosen propagation policy. Avoid inferring adoption from silence, especially if decision events are sampled.

Rehearse the awkward transition. Keep one worker on the older snapshot while another observes the disabled state; verify that their request records preserve their different revisions. Then test a stale operator request, an unauthorized caller, and a request already in flight when the disable occurs. These checks exercise containment and reconstruction together. A test that merely asserts `enabled === false` misses the operational question.

Keep the next weekly release small enough to rerun that rehearsal.

## When deployment rollback is the better choice

Deployment rollback is the better primary response when the defect affects the stable path, the gate itself, or code outside the experimental boundary. If the release cannot evaluate the switch safely, changing flag state is insufficient. Preserve release identity in the evidence so the comparison can account for the replacement.

If harmful work has already crossed a financial side-effect boundary, pause at a safe processing boundary and reconcile accepted operations against completed effects. Repeatedly replaying uncertain work can make reconstruction harder; recovery needs an explicit decision about each unresolved operation.

Choose the kill switch for a contained experiment with a safe stable path. Choose rollback for a defective release. In either case, success means being able to explain which tenant requests ran which behavior and what happened to their operations afterward.

## Further reading

References:

- OWASP Logging Cheat Sheet — event content, sensitive data handling, and log protection: https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
