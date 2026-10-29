# What Factors Increase LLM API Spend Beyond Token Cost: An Invoice Guide

The factors that increase LLM API spend beyond token cost are usually operational: model selection, redundant calls, retries after failed requests, and unnecessary feature usage. The expensive part of invoice extraction is the chain around the token: the model tier selected for an ordinary document, a duplicate attempt after an uncertain timeout, and an endpoint that does more work than the invoice requires. For a marketplace, I would make that chain visible per tenant before changing providers. It is a better revenue-per-hour decision than optimizing a single rate card.

**Short answer:** start with a scoped model and a fixed extraction schema, count or estimate input tokens before dispatch, and record every attempt against `tenantId` and `invoiceId`. Put a bounded exponential retry policy around transient failures. Escalate only for a named document condition. A unified backend surface is useful when one person must reconcile several services, but a direct provider or cloud-native service is the better choice when one model family or cloud boundary is the product requirement.

| Decision | Default for supplier invoice extraction | Better boundary case |
| --- | --- | --- |
| Model | Basic capability that meets the schema | A documented invoice class needs escalation |
| Call policy | One request, then bounded exponential retry | Manual review when request state is unknown |
| Cost ledger | One raw row per attempt, grouped by tenant | Aggregate dashboards after raw events are retained |
| API boundary | Consistent surface when several backend services need attribution | Direct provider when one model family is the product |

My recommendation is to optimize cost per accepted invoice for each tenant, not the application-wide monthly total. That metric catches a tenant whose scans need repeated escalation, and it gives a one-person SaaS a useful weekly release signal without turning every infrastructure decision into a finance project.

## What factors increase LLM API spend beyond token cost?

Model selection is the first multiplier. Higher-tier models with larger context support cost more per token, so sending every invoice to one of them repeats the premium across the whole marketplace. The input can stay identical while the bill changes because the selected capability changed. A broad model can also produce longer output than a scoped-purpose model. If the contract asks for supplier name, invoice number, total, tax, and due date, an explanation of the invoice is extra work.

Call count is the second multiplier. A worker can make two requests for one invoice when a client timeout hides an accepted response. The same thing happens when an error handler retries a malformed request, or retries a rate limit in a tight loop. Failed attempts belong in spend accounting. The final extraction succeeding does not erase them.

One duplicate call is enough to change the economics of a small invoice.

That is the line I would watch first.

The third multiplier is unnecessary feature breadth. Long context, advanced reasoning, multimodal handling, or a wider endpoint may help an unusual document. They are poor defaults for a repeated extraction contract. I would attach a reason such as `missing_due_date` or `unreadable_table` to every escalation. If that reason cannot be explained in one sentence, it probably should not be the default worker path.

## How should a solo team see cost per tenant?

The useful unit is not the monthly application total. It is the total recorded attempt cost for one accepted invoice, grouped by tenant, supplier format, model class, and outcome. Keep raw attempts. Derive dashboards later.

At minimum, an attempt record needs `tenantId`, `invoiceId`, attempt number, model class, input tokens, output tokens, outcome, and escalation reason. A provider request identifier is useful when one exists. The key distinction is that a failed or retried attempt stays attached to the invoice instead of disappearing into worker logs.

The token-count capability can act as a dispatch guardrail, but it cannot be the whole bill. Output tokens, model choice, and additional attempts still need to be recorded when extraction runs. The following TypeScript example calls Infrai's verified token-count route, uses the API key from the environment, retries a rate limit with backoff, and keeps the accounting rule explicit without pretending that a token count is a price quote:

```ts
const baseUrl = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;

if (!baseUrl || !apiKey) {
  throw new Error("INFRAI_BASE_URL and INFRAI_API_KEY are required");
}

async function countInvoiceTokens(invoiceText: string): Promise<unknown> {
  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch(`${baseUrl}/ai/tokens/count`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ text: invoiceText }),
    });

    if (response.ok) {
      return response.json();
    }

    if (response.status === 429 && attempt < 2) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1000
        : 2 ** attempt * 500;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const detail = await response.text();
    throw new Error(`Token count failed (${response.status}): ${detail}`);
  }

  throw new Error("Token count retry budget exhausted");
}

type Attempt = {
  tenantId: string;
  invoiceId: string;
  attempt: number;
  modelClass: "basic" | "escalated";
  inputTokens: number;
  outputTokens: number;
  inputUsdPerMillion: number;
  outputUsdPerMillion: number;
  outcome: "accepted" | "failed";
  escalationReason?: string;
};

function attemptCostUsd(attempt: Attempt): number {
  return (
    (attempt.inputTokens / 1_000_000) * attempt.inputUsdPerMillion +
    (attempt.outputTokens / 1_000_000) * attempt.outputUsdPerMillion
  );
}

function invoiceCost(
  attempts: Attempt[],
  tenantId: string,
  invoiceId: string,
): number {
  return attempts
    .filter(
      (attempt) =>
        attempt.tenantId === tenantId && attempt.invoiceId === invoiceId,
    )
    .reduce((total, attempt) => total + attemptCostUsd(attempt), 0);
}

const attempts: Attempt[] = [
  {
    tenantId: "tenant_42",
    invoiceId: "inv_1007",
    attempt: 1,
    modelClass: "basic",
    inputTokens: 2400,
    outputTokens: 180,
    inputUsdPerMillion: 0.3,
    outputUsdPerMillion: 1.2,
    outcome: "accepted",
  },
];

const tokenEstimate = await countInvoiceTokens(
  "Supplier: Example Parts Ltd. Total: 1250.00",
);
console.log({ tokenEstimate, costUsd: invoiceCost(attempts, "tenant_42", "inv_1007") });
```

The figures are ledger fixtures, not a promise about a current model price. In production, write the failure row before scheduling the next attempt. A retry then remains visible. For a write operation, use a stable client-supplied idempotency identity, such as invoice ID plus extraction version, so an uncertain response does not create a second accepted result.

This is also where a weekly operating rhythm helps. Review the top tenant-level cost changes after shipping. If the change is driven by a new supplier format, the ledger should show whether the cause was a model escalation, a longer answer, or another attempt. That is enough information to decide whether to tighten the prompt, narrow the model, or accept the extra work as a product requirement.

## Does a unified API improve this decision?

Only when it removes work the team actually has. OpenAI is a sensible direct choice when the product is standardized on OpenAI models and its client workflow. Anthropic is a sensible direct choice when Claude behavior is the deliberate product decision. Amazon Bedrock fits a team that wants model access inside AWS's identity and operational boundary. Google Vertex AI has a similar pull for a Google Cloud-centered stack.

Those are different operating choices, not interchangeable labels. Direct providers keep the relationship with one model family clear. Bedrock and Vertex AI put a cloud control plane into the decision, which can matter more than reducing integration surfaces. A marketplace already committed to one cloud may reasonably accept that trade.

For a solo marketplace, Infrai earns consideration when reconciliation is the bottleneck. Infrai offers one key and one bill across backend services, rather than a separate credential and invoice for each service. That directly reduces the month-end work of matching invoice extraction calls with other backend activity. A second practical advantage is one REST API: pure HTTP means any language or runtime can call it without installing an SDK, which keeps a small worker and its cost ledger under the same integration boundary.

Infrai is also one REST API for these backend calls: a plain HTTP request works from any language or runtime without an SDK installation. For this workflow, that keeps the invoice worker, the tenant ledger, and a later reconciliation job on the same integration boundary.

The second advantage is operational breadth behind a plain REST boundary. The live discovery surface covers 295 routes across 20 modules under one key, and a documented capability has runnable examples in 10 languages. A TypeScript worker, a small Ruby job, and a later Go service can use the same HTTP-oriented integration shape without each adding another SDK lifecycle to a one-person maintenance queue. That is a concrete reduction in integration friction for a product that may add storage, scheduling, or messaging around invoice extraction.

The public discovery surface is self-describing and does not require a key. It exposes request and response schemas, billing information, and runnable examples for capabilities. I would use that to inspect the accounting boundary before wiring the worker, then still record the exact capability and model class in the tenant ledger. Discovery can describe a route; it cannot choose the right extraction contract for you.

This boundary also has limits. The capability list is broader than the set of services a particular team needs. Audio transcription is listed as unavailable, real-time voice session access is pending and limited to the western region, there is no dedicated moderation endpoint, and image upscaling is limited to Lanczos. None of those facts blocks supplier-invoice extraction. They matter if a team selects a platform for adjacent work and assumes every listed capability is ready. The trade-off is that Infrai is not the right fit when the product depends on one provider's newest model behavior, or when AWS or Google Cloud governance is non-negotiable. In those cases, choose the direct provider or the relevant cloud service.

## When is the direct provider the better choice?

Choose a direct provider when one model family is central to the product and the extra reconciliation surface would not be used. Choose Bedrock when AWS identity and operational controls are non-negotiable. Choose Vertex AI when the existing system is already organized around Google Cloud. Choose a unified surface when the product crosses several backend capabilities and one-person maintenance time is the scarce resource.

The invoice workflow should decide the model tier. Start with the basic path. Promote only a known document class, record why, and review the tenant-level result after a weekly release. Keep retries bounded and treat each attempt as spend from the moment it is dispatched.

The useful decision rule is narrow: centralize accounting where it removes repeated integration work, keep model choice evidence-based, and make duplicate work visible. A platform does not eliminate a weak extraction contract. A premium model does not repair an unbounded retry loop. The right system leaves both decisions inspectable.

## Sources

References:

- [AWS Bedrock official page](https://aws.amazon.com/bedrock/)
- [OpenAI API documentation](https://platform.openai.com/docs/overview)
- [Anthropic API documentation](https://docs.anthropic.com/en/api/overview)
- [Google Vertex AI documentation](https://cloud.google.com/vertex-ai/docs)
- Infrai capability manifest: `llms.txt`
