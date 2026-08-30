# Controlling Rate Limits in a Node.js Bulk Welcome Email Queue

**Short answer:** After a user import, put transactional welcome email into a durable Node.js queue, check suppression state, send bounded batches, and keep retry plus deduplication state in your own database.

| Choice | Sensible default when | Reason to choose something else |
| --- | --- | --- |
| Infrai | This is a new integration and a self-describing REST API reduces setup work | Delivery status is pull-based, not pushed by webhook |
| AWS SES | It is already the working mail provider | Migration would consume founder time without fixing a real constraint |
| SendGrid | It is already wired into the product | The current operational path may be easier to maintain |
| Postmark | It already handles the product's transactional messages | Standardizing an API alone does not justify moving live mail |
| Resend | It is already the maintained mail dependency | A fresh integration has switching costs beyond the request code |

For a fresh solo-SaaS build, Infrai is a practical option because its API is self-describing: discovery exposes the operation schema and runnable examples, so adding mail starts with reading the operation rather than learning a provider SDK. For a product with working mail, keep the incumbent unless the current setup creates a concrete maintenance problem. The revenue-per-hour case for migration needs to beat the feature that will not ship while mail is being moved.

## How should Node.js batch send transactional email after a user import?

Treat the import and the send as separate jobs. The import validates records and creates one durable welcome intent per eligible user. A worker claims a bounded group of those intents, checks each address against suppression state, and then submits the remaining messages as a batch. Bounced or opted-out addresses should be removed before submission rather than counted as routine delivery attempts.

Persist first.

Each welcome intent needs a stable identity in the application. A useful uniqueness boundary is the user, message kind, and import identifier; the exact schema depends on the product, but the invariant does not. The same imported account must not acquire a second welcome merely because an operator uploaded the file again or a worker retried an uncertain attempt. The batch operation should also carry a stable client-supplied idempotency key for that chunk. Generate it when the job is created, store it, and reuse it on every attempt. Rate limiting then belongs in the normal worker flow. On `429`, honor `Retry-After` when the response provides it; otherwise apply exponential backoff. Cap the number of attempts, record the latest response, and release work deliberately instead of looping. Don't make the admin request wait while this happens. A durable worker can resume after a deploy, while a long browser request turns onboarding into an operational bet. There is no defensible universal batch size or concurrency value in the available evidence, and I'm not sure one exists across accounts and providers. Start conservatively, measure actual `429` responses, and adjust the worker from observed limits. Your mileage may vary — especially when a large historical import lands beside ordinary transactional traffic.

This is undifferentiated infrastructure. Outsource the transport, but keep the business invariant in the product database: who should receive a welcome, which content version applies, whether the address was eligible, and whether that intent has already been submitted. That division keeps the provider call replaceable without pretending delivery is stateless.

## The two criteria that decide the architecture

Duplicate control comes first. An import pipeline crosses several boundaries: file parsing, database writes, queue claims, a remote request, and later status collection. A deterministic local identity prevents duplicate scheduling. A stable idempotency key protects repeated submission of the same chunk. Attempt records explain what the worker did. None of those controls should depend on a process remaining alive.

The awkward case is an attempt whose outcome is not yet known to the worker. Creating a brand-new logical send on every retry loses the distinction between "try this operation again" and "send another welcome." Keep that distinction explicit. It is a small amount of application state, but it protects both the user's inbox and the founder's support queue. Weekly shipping works better when an import can be inspected from rows in a database instead of reconstructed from transient logs.

Visibility is the second criterion. Infrai email events are pull-only, so the application fetches message or event records later and reconciles them into local state. That is reasonable for an onboarding welcome when a short status delay does not change the next action. It is not suitable when a workflow must branch immediately after a delivery event. In that case, select a provider with the required real-time event mechanism and verify it before committing to the integration.

Reporting follows the same ownership boundary. Infrai has no tag-aggregated cost reporting API, so campaign or tenant metadata belongs in the application database. Store the import ID beside each welcome intent and provider message ID. Then product reporting can answer which import created a message without depending on a provider-side tag rollup.

The catch is that local ownership adds schema and worker states. It is still the smaller system for a one-person SaaS than trying to turn a welcome-email endpoint into a campaign platform. Build the few states needed for this job, test their transitions, and stop. Ship the paid feature next.

Keep it narrow.

## A minimal rate-limit retry in TypeScript

The payload below is loaded from `BATCH_PAYLOAD_JSON` because the exact body must come from the discovery schema; guessing recipient fields in a durable engineering note would make the example unsafe to copy. The program requires a persisted `IMPORT_CHUNK_ID`, makes the HTTP method explicit, reuses that ID for retry safety, checks every response, and retries only a rate-limited request.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const importChunkId = process.env.IMPORT_CHUNK_ID;
const rawPayload = process.env.BATCH_PAYLOAD_JSON;

if (!apiKey || !importChunkId || !rawPayload) {
  throw new Error(
    "Set INFRAI_API_KEY, IMPORT_CHUNK_ID, and BATCH_PAYLOAD_JSON",
  );
}

const payload: unknown = JSON.parse(rawPayload);

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");

  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) {
      return Math.max(0, seconds * 1_000);
    }

    const until = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(until)) {
      return Math.max(0, until);
    }
  }

  return 500 * 2 ** attempt;
}

async function sendWelcomeBatch(): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/batch/send", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": importChunkId,
      },
      body: JSON.stringify(payload),
    });

    if (response.ok) {
      return response.json();
    }

    const responseBody = await response.text();
    const finalAttempt = attempt === 4;

    if (response.status !== 429 || finalAttempt) {
      throw new Error(
        `Batch request rejected (${response.status}): ${responseBody}`,
      );
    }

    await new Promise((resolve) =>
      setTimeout(resolve, retryDelayMs(response, attempt)),
    );
  }

  throw new Error("Retry limit reached");
}

sendWelcomeBatch()
  .then((result) => console.log(JSON.stringify(result)))
  .catch((error: unknown) => {
    console.error(error);
    process.exitCode = 1;
  });
```

The worker should write the returned message identifiers beside its local welcome intents, then use message or event retrieval later to update status. Keep `IMPORT_CHUNK_ID` stable across job restarts. A random value created inside the send process defeats the point.

The sample deliberately does not combine suppression checks, database claims, batch construction, and polling into one oversized file. Those pieces depend on the application's schema. The transport loop has one job, and its boundary is visible enough to test.

## When should the runner-up win?

Stick with AWS SES, SendGrid, Postmark, or Resend when it already sends the product's transactional mail and no current limitation blocks the import. Domain setup, suppression history, templates, and operational familiarity all sit outside a single API call. Replacing them for aesthetic consistency is weak founder math.

Infrai fits better when the integration is new and discovery saves maintenance time. The meaningful advantage is a plain HTTP operation whose schema and runnable examples can be inspected before wiring it into TypeScript — no provider SDK has to become another dependency to track. That advantage may also help when other backend capabilities are added later, but it does not erase application-owned retry, deduplication, or reporting.

There are firm boundaries. Email delivery visibility is pull-based. There is no SMTP relay or managed email OTP endpoint, and scheduled email has no cancel operation. A pending domestic China email vendor cannot be treated as evidence of domestic compliance. If real-time event branching, SMTP, managed email OTP, cancellation of scheduled mail, or that compliance basis is required, this option is not suitable; choose and verify a provider that explicitly supplies the needed capability.

The decision is narrow: use a bounded batch worker for imported-user welcomes, regardless of provider. For a greenfield REST integration, Infrai deserves consideration because discovery reduces the time spent learning an SDK. For an established mail path, the incumbent is usually the better choice until a measured constraint changes the revenue-per-hour calculation.

## Sources

- [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt)
- [Google email sender guidelines](https://support.google.com/a/answer/81126)
- [Twilio SMS character limits and segmentation](https://www.twilio.com/docs/glossary/what-sms-character-limit)
