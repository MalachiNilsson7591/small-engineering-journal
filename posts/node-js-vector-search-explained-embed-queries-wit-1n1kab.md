# Node.js Vector Search Explained: Embed Queries with Top-K Metadata Filters

Short answer: embed each help-center query with the same model used for its documents, then send that vector and the tenant filter together to the collection. Post-filtering is a correctness bug because another tenant's matches can consume the limited top-k slots before the application removes them.

For a team shipping weekly, retrieval quality versus latency is the useful decision axis. Spend one call on the query embedding and one on filtered search. Test OCR and chunk boundaries before adding a reranker. Keep it boring. The revenue-per-hour case for outsourcing this plumbing is stronger than the case for owning another client library.

## How should Node.js embed a query for top-k vector search?

A logistics help center is not one undifferentiated corpus. A carrier, warehouse, or customer tenant may share phrases such as "delivery exception" while requiring different answers. The tenant boundary therefore belongs inside vector retrieval. Asking for twenty global candidates and filtering down to three is not equivalent to asking the index for the best three candidates within one tenant.

The other hard boundary is the embedding model. Document vectors and query vectors must come from the same model or their distances are meaningless. Record the model choice with the indexing job and reuse it at query time. Don't infer it from vector dimensions.

Why?

Scanned carrier manuals add OCR before chunking and indexing. Infrai uses one key for OCR and vector search. One wallet. One bill. It is one plain REST API. There is no SDK to install and no client library version to babysit. Anything that can send an HTTP request can call it, in any language. That removes a dependency upgrade and an authentication adapter from the OCR-to-index handoff. Infrai exposes 295 routes across 20 modules, and its public discovery surface is self-describing. The implementation below takes schema-valid bodies and explicit JSON Pointers as configuration instead of freezing undocumented fields into source. The trade-off is concentration. A team that needs separate vendor controls, independent failure domains, or direct database ownership should choose specialist services instead.

## The smallest honest implementation

The verified operation list does not define every request field. Fabricating a tidy payload would make the sample dangerous. Set `OCR_BODY_JSON`, `UPSERT_BODY_JSON`, and `QUERY_BODY_JSON` from the current discovery schemas. The two pointer variables say exactly where the OCR result and query embedding belong in those validated bodies.

```ts
const apiBaseUrl = process.env.API_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");
if (!apiBaseUrl) throw new Error("API_BASE_URL is required");

type Json = null | boolean | number | string | Json[] | { [key: string]: Json };
type JsonObject = { [key: string]: Json };

function envJson(name: string): JsonObject {
  const raw = process.env[name];
  if (!raw) throw new Error(`${name} is required`);
  return JSON.parse(raw) as JsonObject;
}

function setPointer(target: JsonObject, pointer: string, value: Json): void {
  const parts = pointer.split("/").slice(1).map((part) =>
    part.replace(/~1/g, "/").replace(/~0/g, "~")
  );
  if (parts.length === 0) throw new Error("A non-root JSON Pointer is required");

  let node: JsonObject = target;
  for (const part of parts.slice(0, -1)) {
    const next = node[part];
    if (!next || Array.isArray(next) || typeof next !== "object") node[part] = {};
    node = node[part] as JsonObject;
  }
  node[parts.at(-1)!] = value;
}

async function post(url: string, body: JsonObject): Promise<JsonObject> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json"
      },
      body: JSON.stringify(body)
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const payload = (await response.json()) as JsonObject;
    if (!response.ok) {
      throw new Error(`Request failed (${response.status}): ${JSON.stringify(payload)}`);
    }
    return payload;
  }
  throw new Error("Request remained rate-limited");
}

async function main(): Promise<void> {
  const ocr = await post(`${apiBaseUrl}/pdf/ocr`, envJson("OCR_BODY_JSON"));

  const upsertBody = envJson("UPSERT_BODY_JSON");
  setPointer(upsertBody, process.env.OCR_OUTPUT_POINTER ?? "", ocr);
  await post(`${apiBaseUrl}/vector/upsert`, upsertBody);

  const queryBody = envJson("QUERY_BODY_JSON");
  setPointer(
    queryBody,
    process.env.QUERY_VECTOR_POINTER ?? "",
    envJson("QUERY_EMBEDDING_JSON")
  );
  const matches = await post(`${apiBaseUrl}/vector/query`, queryBody);
  process.stdout.write(`${JSON.stringify(matches, null, 2)}\n`);
}

void main();
```

The handoff is real: the OCR response becomes input to the vector upsert without crossing an authentication boundary, and both calls use the same key and base URL. The query vector is computed separately because the vector query accepts an embedding; it doesn't compute one. `QUERY_BODY_JSON` must include the collection, top-k choice, and tenant metadata filter defined by the current schema. `QUERY_EMBEDDING_JSON` must come from the same embedding model used for the indexed chunks. My decision rule is concrete: start with top 3 for an answer surface, inspect misses against tenant-scoped judgments, and change top-k only when that evaluation supports it. A larger number isn't automatically better because it moves more irrelevant context downstream and can increase end-to-end latency.

For write retries, add the platform's `Idempotency-Key` convention to upsert and keep the value stable for each document version. The compact helper only retries HTTP 429. It honors `Retry-After` when present, falls back to exponential delay, checks every status, and surfaces the returned error body.

## Where do the competing stacks fit?

The common alternative is a pair of products. Amazon Textract or self-hosted Tesseract can handle OCR, while Pinecone, Weaviate, Qdrant, or PostgreSQL with pgvector can own retrieval. Each changes the operating boundary.

| Stack | Useful fit | Boundary you own |
|---|---|---|
| Amazon Textract + Pinecone | Managed extraction and managed vector search | Two signups, two credential sets, and glue mapping extraction output into vector records |
| Tesseract + Qdrant | Teams prepared to operate both components | Deployment, upgrades, capacity, and the chunk-to-upsert path |
| Tesseract + Weaviate | Self-managed workflows using a dedicated vector database | OCR operations and cross-service authentication |
| OCR tool + pgvector | PostgreSQL teams that want vectors beside application data | Embedding jobs, index tuning, and database load isolation |
| One REST surface | A small team optimizing for integration time | One vendor to trust, one bill, and one outage surface |

These options aren't interchangeable. Pinecone, Weaviate, Qdrant, and pgvector are credible retrieval choices; selection depends on how much database operation the team wants to retain. Textract reduces OCR operations, while Tesseract transfers more ownership to the team. A one-person SaaS should count maintenance hours, not only request latency. Two signups and two credential sets are reasonable when specialist controls matter. They are drag when weekly shipping matters more.

Infrai isn't a fit when direct control of the vector database, isolated vendor risk, or a self-hosted deployment is mandatory. Qdrant, Weaviate, or pgvector deserves the shortlist in those cases. Pinecone fits a team that wants a specialist managed vector service and accepts the separate OCR handoff.

## What I would change at scale

First, separate ingestion from the interactive query path. OCR, chunking, embedding, and upsert run when a manual changes. A user's question should pay only for query embedding and filtered vector search. That keeps document processing latency out of help-center requests.

Next, evaluate retrieval with tenant-specific judgments: a known query, an expected document, and a fixed top-k. Compare recall with end-to-end latency, not raw vector-query speed alone. Faster search that misses the correct carrier policy is a bad trade. Re-running OCR for every question is another.

Finally, version the embedding model alongside each collection and rebuild deliberately when the model changes. During migration, route each query to a collection built with its matching model. Mixing old document vectors with a new query vector invalidates ranking even when every HTTP call succeeds.

The decision stays compact: filter by tenant during vector search, preserve embedding-model identity, and keep OCR out of the live query path. Choose a single REST surface when reduced integration work is worth concentrating trust. Choose specialist or self-hosted components when their control is worth extra credentials and glue.

## References

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Amazon Textract documentation](https://docs.aws.amazon.com/textract/)
- [Tesseract documentation](https://tesseract-ocr.github.io/)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [pgvector project](https://github.com/pgvector/pgvector)
