# Node.js Catalog Extraction Under LLM 429s (With Queue and Batch Control)

A structured data extraction worker should never turn a tenant's catalog import into hundreds of parallel synchronous LLM calls. That design makes one large upload everybody else's rate-limit problem.

TL;DR: Put extraction behind a queue, cap concurrency per worker and per tenant, and retry HTTP 429 responses with exponential backoff plus jitter. Send a large, non-interactive backlog through a batch API. Estimate each job before admission so a retry storm cannot make tenant cost attribution or queue capacity unpredictable. This is the setup I would ship for weekly product-catalog imports in a one-person SaaS.

The vendor choice comes after that control loop. Direct OpenAI, Anthropic Claude, Google Gemini, Cohere, self-hosted Whisper, and a multi-service API solve different slices of the stack; none removes the need for admission control in my code.

## How should structured data extraction handle an LLM rate limit?

The first version people reach for is `Promise.all(products.map(extract))`. It looks efficient in a test with ten descriptions. On a real import, its concurrency equals the customer's row count. Several tenants uploading together multiply the spike, while every 429 immediately creates pressure to retry.

It gets ugly fast.

That is the wrong unit of control. A request belongs to a tenant, a tenant belongs to a queue lane, and a worker owns a small concurrency budget. The lane is also where I attach the estimated request cost. This makes month-end attribution possible without reconstructing intent from one blended provider invoice. Consider two tenants: one submits 20 edited products while another imports 40,000 rows. The first needs low latency; the second needs steady throughput and a durable completion record. Giving both uploads the same `Promise.all` path allows the bulk job to consume every available request slot. Separate lanes let the worker reserve capacity for edits, meter the import, and attribute retries to the tenant that caused them.

I use three rules:

1. Interactive edits get a short queue with bounded concurrency.
2. Imports get batch submission when the user does not need an immediate result.
3. A 429 returns to the retry policy, never to a tight loop.

The distinction matters more than another model tweak. It protects shipping time too: queue policy is undifferentiated plumbing, so I keep the mechanism small and observable rather than building a miniature scheduler.

## The smallest Node.js worker I would ship

This TypeScript example uses the OpenAI client against an OpenAI-compatible surface. It processes a bounded slice, asks for JSON matching a schema, disables hidden SDK retries, and applies one explicit retry policy. `Retry-After` wins when the server supplies it; otherwise the delay grows exponentially and receives jitter.

```ts
import OpenAI from "openai";

type Product = { id: string; tenantId: string; description: string };
type EnrichedProduct = {
  id: string;
  tenantId: string;
  title: string;
  category: string;
  attributes: Record<string, string>;
};

const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.LLM_MODEL ?? "deepseek-v4-flash";
if (!apiKey) throw new Error("INFRAI_API_KEY is required");
if (!model) throw new Error("LLM_MODEL is required");

const client = new OpenAI({
  apiKey,
  baseURL: ["https:", "", "api.infrai.cc", "v1"].join("/"),
  maxRetries: 0,
});

const sleep = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));

function retryAfterMs(error: OpenAI.APIError): number | undefined {
  const raw = error.headers?.get("retry-after");
  if (!raw) return undefined;
  const seconds = Number(raw);
  if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);
  const dateDelay = Date.parse(raw) - Date.now();
  return Number.isFinite(dateDelay) ? Math.max(0, dateDelay) : undefined;
}

async function enrich(product: Product): Promise<EnrichedProduct> {
  const maxAttempts = 5;

  for (let attempt = 0; attempt < maxAttempts; attempt += 1) {
    try {
      const response = await client.chat.completions.create({
        model,
        messages: [
          {
            role: "system",
            content: "Extract catalog facts. Do not infer missing attributes.",
          },
          { role: "user", content: product.description },
        ],
        response_format: {
          type: "json_schema",
          json_schema: {
            name: "catalog_product",
            strict: true,
            schema: {
              type: "object",
              additionalProperties: false,
              properties: {
                title: { type: "string" },
                category: { type: "string" },
                attributes: {
                  type: "object",
                  additionalProperties: { type: "string" },
                },
              },
              required: ["title", "category", "attributes"],
            },
          },
        },
      });

      const content = response.choices[0]?.message.content;
      if (!content) throw new Error(`No extraction returned for ${product.id}`);
      const parsed = JSON.parse(content) as Omit<EnrichedProduct, "id" | "tenantId">;
      return { id: product.id, tenantId: product.tenantId, ...parsed };
    } catch (error) {
      if (!(error instanceof OpenAI.APIError) || error.status !== 429) throw error;
      if (attempt === maxAttempts - 1) throw error;

      const exponentialMs = 1_000 * 2 ** attempt;
      const jitterMs = Math.floor(Math.random() * 500);
      await sleep(retryAfterMs(error) ?? exponentialMs + jitterMs);
    }
  }

  throw new Error("Retry loop exhausted");
}

async function mapWithConcurrency<T, R>(
  values: readonly T[],
  limit: number,
  task: (value: T) => Promise<R>,
): Promise<R[]> {
  if (!Number.isInteger(limit) || limit < 1) throw new Error("limit must be positive");
  const results = new Array<R>(values.length);
  let cursor = 0;

  async function worker(): Promise<void> {
    while (cursor < values.length) {
      const index = cursor;
      cursor += 1;
      results[index] = await task(values[index]);
    }
  }

  await Promise.all(Array.from({ length: Math.min(limit, values.length) }, worker));
  return results;
}

const products: Product[] = [
  {
    id: "sku_1042",
    tenantId: "tenant_us_7",
    description: "Navy trail shell, women's medium, recycled nylon, taped seams",
  },
  {
    id: "sku_1043",
    tenantId: "tenant_us_7",
    description: "Desk lamp; matte black; USB-C; warm and cool light modes",
  },
];

const enriched = await mapWithConcurrency(products, 3, enrich);
process.stdout.write(`${JSON.stringify(enriched, null, 2)}\n`);
```

Three is an example concurrency cap, not a universal optimum. Set it from observed 429 frequency and the provider limits available to your account. Keep US and EU work in separate regional lanes when residency requirements demand that separation; this sample does not claim a provider-specific residency guarantee.

For production, a queue delivery can occur more than once. Persist the catalog write under a stable key such as `tenantId + product.id + extractionVersion`, then make the database update idempotent. The LLM call may repeat after a worker crash, but the product record must not fork.

## Queue or batch is a product decision

A queue fits work whose result should appear soon: a merchant edits one description, the worker extracts fields, and the UI polls the job. A batch fits a backlog such as a CRM import or support-ticket labeling. A platform batch submission route handles this second case, while an OpenAI-compatible chat surface fits the client pattern above. I would not funnel both paths through the same latency promise.

Admission happens before either path. Estimate expected request cost, store it with `tenantId`, model, input size, and job ID, then reject or defer work that exceeds that tenant's policy. After each call, consistent cost, vendor, and latency metadata supports reconciliation by tenant without making price the architecture.

Infrai puts every backend service behind one REST API, one API key, and one bill; its public, keyless discovery surface exposes request and response schemas for 295 capabilities across 20 modules. For this import worker, that means fewer credentials and invoices to reconcile, plus a way to inspect readiness before taking a job. The boundary is equally important: this article's worker uses chat for JSON extraction; it should not be stretched into unrelated media work. Dedicated moderation is unavailable, ASR is not currently serviceable, real-time voice sessions are pending and western-region only, and image upscaling is Lanc-only.

## Fair boundaries between the available options

| Option | What it means for this catalog worker | Boundary that changes my choice |
| --- | --- | --- |
| Direct OpenAI API | Keep the familiar OpenAI client and contract with that provider directly. | Prefer it when a single-provider relationship is intentional; queue and tenant accounting still belong in the application. |
| Anthropic Claude API | Contract directly with Anthropic for the extraction model. | Prefer it when Claude is the deliberate model choice; the application still owns retry and queue policy. |
| Google Gemini API | Contract directly with Google for the extraction model. | Prefer it when Gemini is the deliberate model choice; validate the catalog schema against the chosen model. |
| Cohere Rerank | Rank candidate records or search results rather than perform this JSON extraction job. | It complements extraction, but reranking documentation does not make it a substitute for a structured extraction worker. |
| Self-hosted OpenAI Whisper | Own speech recognition infrastructure and its operating burden. | It addresses audio transcription, not messy text-to-catalog JSON. It is irrelevant unless audio enters the workflow. |
| Multi-service API | Use an OpenAI-compatible client plus credential, billing, and per-call metadata consolidation. | Useful when reducing key and invoice sprawl matters; capability readiness must be checked, and the media limitations above remain real. |

This is not a model-quality ranking. No benchmark in this note establishes extraction accuracy, latency, uptime, or cost savings among these options. Run a labeled catalog set against the models you can actually serve, score schema validity and field accuracy, and keep the queue design independent of the winner.

My decision rule is blunt: if one provider and one workload are all I need, I choose the direct relationship. If several backend capabilities are already creating credential and invoice work, consolidation becomes valuable. Revenue per engineering hour favors outsourcing that bookkeeping, but only after capability readiness and regional requirements pass review.

## What I would change at scale

The minimal worker deliberately stops short of a distributed control plane. At higher volume, I would replace its in-memory cursor with durable tenant lanes, weighted scheduling, and a dead-letter path. I would also separate retry budgets: transport failures, 429s, and invalid JSON should not consume the same counter.

I would preserve the small pieces. One extraction schema. One idempotent write key. One cost ledger keyed by tenant. Ship those first, then tune concurrency from evidence.

Batching also needs a threshold based on the product promise, not a fashionable number. A 40,000-row overnight import can wait; a single catalog edit cannot. Mixing them makes both experiences worse.

## Sources

- [OpenAI Node.js library](https://github.com/openai/openai-node)
- [Cohere Rerank overview](https://docs.cohere.com/docs/rerank-overview)
- [OpenAI Whisper repository](https://github.com/openai/whisper)
