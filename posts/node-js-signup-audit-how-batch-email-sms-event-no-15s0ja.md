# Node.js Signup Audit: How Batch Email SMS Event Notifications Isolate Partial Failure

Short answer: treat every signup verification recipient as a small state machine, not as one row in a “bulk sent” job. Record the enqueue decision, each channel attempt, the provider correlation ID, and every accepted status transition. When a queue looks stuck, reconcile only nonterminal recipients and retry only when the evidence says another attempt is safe.

That design fixes the dangerous ambiguity in mixed email and SMS batches: a completed worker does not prove delivery, while a timed-out worker does not prove failure. For a one-person e-commerce SaaS, the practical target is smaller. Ship a narrow ledger that can answer who was attempted, what happened, and why the next action was allowed. Outsource transport, but keep that evidence.

Evidence first.

## How should batch event notifications handle partial email and SMS failure?

There are at least three clocks in play. The application enqueues a verification notice. A worker hands it to a transport. Later, the transport may report a recipient-level outcome. Collapsing those events into a single `sent` boolean loses the boundary between local execution and remote acceptance.

Partial failure is normal at that boundary. One email address may be accepted while an SMS attempt for another recipient times out. A process can also lose its response after the remote side accepted the request. Retrying the whole batch then risks duplicate verification messages. Marking the whole batch complete hides the unresolved recipient. Neither choice produces credible compliance evidence.

Use explicit states instead:

| State | Meaning | Automatic action |
| --- | --- | --- |
| `queued` | An application decision exists, but no attempt has started | Claim for delivery |
| `submitting` | This attempt has a stable idempotency key | Wait briefly, then reconcile |
| `accepted` | The transport accepted responsibility | Poll or await a signed status event |
| `delivered` | Recipient-level delivery evidence arrived | Stop |
| `retryable` | A classified temporary outcome arrived | Retry with backoff |
| `failed` | A permanent outcome or policy limit was reached | Stop and retain evidence |

“Accepted” deserves its own row. SMTP defines successful transfer in terms of a server accepting responsibility for a message; that is different from proving that a person received or read it [1]. SMS transports have the same architectural distinction even when their status vocabulary differs. Normalize external labels at the adapter boundary, but preserve the original label and payload hash for audit.

The verification link itself should be single-purpose, expire according to the account risk policy, and become unusable after successful verification. Do not put the token, phone number, email address, or rendered message body in routine logs. Evidence needs identifiers and decisions, not reusable secrets.

## Build the smallest useful recipient ledger

Start with one durable record per intended recipient and channel. A batch is only a grouping key. It never owns the delivery truth.

The following TypeScript keeps the transport generic and makes the decision boundary visible. The repository methods must be backed by transactional storage; an in-memory map would not protect state across worker restarts.

```ts
type Channel = "email" | "sms";
type DeliveryState =
  | "queued"
  | "submitting"
  | "accepted"
  | "delivered"
  | "retryable"
  | "failed";

type RecipientDelivery = {
  id: string;
  batchId: string;
  signupId: string;
  channel: Channel;
  destinationRef: string;
  templateVersion: string;
  state: DeliveryState;
  attempt: number;
  idempotencyKey: string;
  providerMessageId?: string;
  nextCheckAt?: Date;
  updatedAt: Date;
};

type SubmitResult = {
  providerMessageId: string;
  providerStatus: string;
};

interface DeliveryTransport {
  submit(input: {
    destinationRef: string;
    templateVersion: string;
    idempotencyKey: string;
    variables: Readonly<Record<string, string>>;
  }): Promise<SubmitResult>;
  getStatus(providerMessageId: string): Promise<string>;
}

interface DeliveryRepository {
  claimQueued(id: string): Promise<RecipientDelivery | undefined>;
  recordAccepted(
    id: string,
    providerMessageId: string,
    providerStatus: string,
    nextCheckAt: Date,
  ): Promise<void>;
  recordUncertain(id: string, reason: string, nextCheckAt: Date): Promise<void>;
}

async function submitRecipient(
  id: string,
  repo: DeliveryRepository,
  transports: Record<Channel, DeliveryTransport>,
  variables: Readonly<Record<string, string>>,
): Promise<void> {
  const row = await repo.claimQueued(id);
  if (!row) return;

  try {
    const result = await transports[row.channel].submit({
      destinationRef: row.destinationRef,
      templateVersion: row.templateVersion,
      idempotencyKey: row.idempotencyKey,
      variables,
    });

    await repo.recordAccepted(
      row.id,
      result.providerMessageId,
      result.providerStatus,
      new Date(Date.now() + 60_000),
    );
  } catch (error) {
    const reason = error instanceof Error ? error.name : "UnknownError";
    await repo.recordUncertain(row.id, reason, new Date(Date.now() + 60_000));
  }
}
```

The key detail is `submitting`. `claimQueued` should atomically move one row into that state and increment its attempt count before any network call. A timeout stays uncertain; it is not immediately rewritten as “failed.” The stable idempotency key should identify the logical recipient delivery, not a worker run. A transport that supports idempotent submission can then deduplicate repeated calls. If it cannot, reconciliation must precede resubmission.

Template rendering belongs before enqueue. Mustache variables are HTML-escaped by default, while triple braces or `&` produce unescaped output [2]. For a verification URL, define the template contract, validate every required variable, and test the rendered link. Store a template version and a digest of the rendered content rather than the secret-bearing content itself. This ties an attempt to an artifact without turning the audit table into a credential store.

## Reconcile uncertainty instead of replaying the batch

Polling is a repair loop, not the primary send loop. Select due rows in `accepted` or stale `submitting`, lock a limited set, and ask the appropriate adapter for recipient status. Apply a monotonic transition so a late “accepted” observation cannot overwrite `delivered`.

```ts
const terminalStates = new Set<DeliveryState>(["delivered", "failed"]);

type NormalizedObservation = {
  state: "accepted" | "delivered" | "retryable" | "failed";
  occurredAt: Date;
  rawStatus: string;
};

const allowedTransitions: Record<DeliveryState, ReadonlySet<DeliveryState>> = {
  queued: new Set(["submitting"]),
  submitting: new Set(["accepted", "retryable", "failed"]),
  accepted: new Set(["delivered", "retryable", "failed"]),
  delivered: new Set(),
  retryable: new Set(["submitting", "failed"]),
  failed: new Set(),
};

function canApply(current: DeliveryState, observed: DeliveryState): boolean {
  if (terminalStates.has(current)) return false;
  return allowedTransitions[current].has(observed);
}

function nextPollDelayMs(pollCount: number): number {
  const cappedExponent = Math.min(pollCount, 6);
  const base = 30_000 * 2 ** cappedExponent;
  const jitter = Math.floor(Math.random() * 10_000);
  return Math.min(base + jitter, 30 * 60_000);
}
```

Keep the transition and its evidence in one database transaction. The evidence row can include the recipient-delivery ID, normalized state, raw external status, provider message ID, observation time, ingestion time, source (`submit`, `poll`, or `webhook`), and a digest of the raw payload. Retention and access controls should follow the applicable policy; no universal duration fits every jurisdiction and business.

This also makes “stuck” measurable. Define it as a state plus an age, never as a vague batch condition: for example, `submitting` past its reconciliation deadline or `accepted` past its next check. Those thresholds are operating choices. Derive them from the transport contract and the signup experience, then put them in configuration so the evidence explains which policy was active.

Retries need a budget per logical delivery. Classify explicit permanent outcomes as `failed`. For temporary outcomes, schedule a new attempt with backoff and jitter, preserving the prior attempt records. Rate limits and transport outages should slow the queue globally; a malformed destination should stop only that recipient. This is where per-recipient state pays for itself.

## Test the evidence, not just the happy path

A worker unit test that asserts “submit was called” proves very little. The valuable tests cut the process between durable writes and network responses.

Run these cases before deployment:

1. The transport accepts a request, but the worker loses the response. Reconciliation must avoid an unexamined duplicate.
2. Email is delivered while SMS returns a permanent recipient error. The batch summary must show both outcomes without rolling either back.
3. The same status event arrives twice and then an older event arrives late. The ledger must remain terminal and retain both observations.
4. A worker crashes after claiming a row. A lease or stale-state scan must make that row eligible for reconciliation.
5. A template variable is absent. Enqueue must fail before any delivery attempt exists.

Use synthetic destinations supplied for testing by the chosen transport, and keep production tokens out of fixtures. In staging, assert the ledger fields and state progression rather than assuming that one API response represents the whole lifecycle.

For operations, one dashboard is enough at first: counts by channel and state, age of the oldest nonterminal row, retries per recipient, and transition lag. Alert on age and growth, not on a single transient error. A queue depth of 500 might be healthy during a campaign; one verification link in `submitting` for hours is not.

Email operations also extend beyond queue mechanics. Current sender guidance includes authentication, complaint handling, valid forward and reverse DNS, and easy unsubscribe requirements for promotional traffic [3]. A signup verification message is transactional, but domain reputation and accurate classification still matter. Keep marketing consent and verification purpose as separate application decisions so a delivery retry cannot silently become a marketing send.

## What I would change at scale

I would keep the recipient ledger and replace only its pressure points. Polling can give way to authenticated webhooks plus a slower reconciliation sweep. Database row claiming can move to partitioned queues. Raw evidence can move to access-controlled archival storage while the hot table retains normalized transitions.

I would not begin there. Every extra component consumes the same hours that could ship a checkout improvement. The revenue-per-hour test is blunt: add infrastructure when the current ledger cannot meet a measured latency, throughput, or retention requirement. Ship the auditable state machine this week; outsource undifferentiated transport; measure before splitting services.

The trade-off is more rows and explicit policy in exchange for localized retries and defensible evidence. That is a good bargain for account verification because duplicate messages erode trust, missing messages block revenue, and a batch-level success flag explains neither.

## Further reading

- [1] RFC 5321, Simple Mail Transfer Protocol: https://www.rfc-editor.org/rfc/rfc5321
- [2] Mustache template syntax manual: https://mustache.github.io/mustache.5.html
- [3] Yahoo sender best practices and requirements: https://senders.yahooinc.com/best-practices/
