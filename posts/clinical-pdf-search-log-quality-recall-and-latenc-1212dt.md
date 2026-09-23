# Clinical PDF Search: Log Quality, Recall, and Latency for Each Query

Log one structured event for every question, with recall@k beside end-to-end and stage latency. The deciding constraint is document freshness: a fast search over stale or badly split clinical PDFs is still a failed retrieval. **Treat chunk revision as part of the measurement**, not as incidental metadata.

TL;DR: keep a small judged set of relevant chunk IDs for evaluation queries, time each retrieval stage, and emit the query ID, corpus revision, chunking version, returned IDs, recall@k, and durations together. Use the same event shape online, where relevance labels may be absent. This keeps the dashboard honest without putting a second evaluation system on the critical path.

## How should we log retrieval quality, recall, and latency per query?

A useful event answers three questions: which corpus and chunking policy were searched, how much of the known relevant material appeared in the first k results, and where the request spent time. Leave any one out and the number becomes hard to act on.

Measure all three together.

For a folder of clinical PDFs, identify a chunk with a stable document ID, a document revision, and a chunk ID. A filename alone is weak evidence because a replaced PDF can keep its name. The revision also prevents a labeled query from being scored against a corpus different from the one its labels describe.

Recall@k is the fraction of labeled relevant chunks found among the first k returned chunks. If the judged relevant set is `{a, b, c}` and the first five results contain `{a, c}`, recall@5 is 2/3. It is not answer correctness. It measures a narrower failure boundary: did retrieval expose the expected evidence to the generation step?

That distinction matters.

Latency needs equally explicit boundaries. Record retrieval time separately from total request time. If the implementation has a distinct candidate search and reranking step, record both. Do not infer stage time by subtracting unrelated dashboard percentiles; calculate durations inside the same request.

## The smallest working TypeScript implementation

The implementation below uses a generic retriever and a replaceable event sink. It has no dependency on a search engine or telemetry product. The evaluation labels live outside the request handler, so production traffic can use the same function without pretending that every query has ground truth.

```ts
interface Hit {
  chunkId: string;
  documentId: string;
  documentRevision: string;
  score: number;
}

interface Retriever {
  search(query: string, limit: number): Promise<Hit[]>;
}

interface RetrievalEvent {
  queryId: string;
  corpusRevision: string;
  chunkingVersion: string;
  k: number;
  returnedChunkIds: string[];
  relevantChunkIds?: string[];
  recallAtK?: number;
  retrievalMs: number;
  totalMs: number;
  status: "ok" | "error";
}

type EventSink = (event: RetrievalEvent) => void;

function recallAtK(
  returnedIds: string[],
  relevantIds: string[],
  k: number,
): number | undefined {
  const relevant = new Set(relevantIds);
  if (relevant.size === 0) return undefined;

  const found = new Set(
    returnedIds.slice(0, k).filter((id) => relevant.has(id)),
  );
  return found.size / relevant.size;
}

async function retrieveWithMetrics(input: {
  queryId: string;
  query: string;
  k: number;
  corpusRevision: string;
  chunkingVersion: string;
  relevantChunkIds?: string[];
  retriever: Retriever;
  emit: EventSink;
}): Promise<Hit[]> {
  const totalStartedAt = Date.now();
  const retrievalStartedAt = Date.now();

  try {
    const hits = await input.retriever.search(input.query, input.k);
    const retrievalMs = Date.now() - retrievalStartedAt;
    const returnedChunkIds = hits.map((hit) => hit.chunkId);

    input.emit({
      queryId: input.queryId,
      corpusRevision: input.corpusRevision,
      chunkingVersion: input.chunkingVersion,
      k: input.k,
      returnedChunkIds,
      relevantChunkIds: input.relevantChunkIds,
      recallAtK: input.relevantChunkIds
        ? recallAtK(returnedChunkIds, input.relevantChunkIds, input.k)
        : undefined,
      retrievalMs,
      totalMs: Date.now() - totalStartedAt,
      status: "ok",
    });

    return hits;
  } catch (error) {
    input.emit({
      queryId: input.queryId,
      corpusRevision: input.corpusRevision,
      chunkingVersion: input.chunkingVersion,
      k: input.k,
      returnedChunkIds: [],
      relevantChunkIds: input.relevantChunkIds,
      retrievalMs: Date.now() - retrievalStartedAt,
      totalMs: Date.now() - totalStartedAt,
      status: "error",
    });
    throw error;
  }
}
```

One detail matters here: an empty relevance set returns `undefined`, not zero. Zero would claim retrieval missed known evidence. In fact, there was no scorable judgment. Keep those states separate.

Missing labels are not misses.

The catch path emits latency and status, then rethrows. Failed requests belong in the latency distribution and error count, but not in recall. That is a deliberate trade: one event schema serves both evaluation and operations while the fields retain their actual meaning.

## How do chunking and freshness change the score?

Chunk IDs are part of the evaluation contract. Change chunk boundaries and old relevant IDs may disappear, even if the underlying PDF says the same thing. Version the chunking policy, rebuild the judged set for that version, and compare like with like. Otherwise a rollout can look like a recall collapse when the labels are merely orphaned.

Freshness needs a similar guard. Pin each evaluation run to a corpus revision. For online events, log the active revision and the revisions of returned documents. A question about a newly replaced clinical policy should not receive quiet credit for retrieving an older chunk whose wording happens to overlap.

Stale evidence fails fast.

This is where the revenue-per-hour lens helps. Do not begin with a large annotation platform. Start with a small set of high-value questions, but do not present its size as a universal benchmark. Cover the PDF changes that would create support work or unsafe answers. Ship the instrumentation this week; expand judgments when a real failure exposes a missing category. Outsource storage, charting, and alert delivery to whichever generic systems are already maintained. The labeled queries and chunk identity rules are the differentiated work.

A compact evaluation record can look like this:

```ts
const judgedQuery = {
  queryId: "contraindications-001",
  query: "Which contraindications are listed for this procedure?",
  corpusRevision: "clinical-pdfs-2026-09-24T08:00:00Z",
  chunkingVersion: "section-aware-v2",
  relevantChunkIds: ["policy-17:r4:contraindications:0"],
};
```

The text is realistic; the identifier is illustrative. Avoid logging patient text or document contents merely to make debugging convenient. Query identifiers, chunk identifiers, revisions, counts, and durations are enough for the metric path shown here. Any decision to retain query text belongs in a separate data-governance review.

## Reading the log without fooling yourself

Aggregate only compatible events. Group recall by corpus revision, chunking version, and k before comparing a release. Group latency by status as well as stage. Then inspect distributions rather than relying on one average, because a small set of slow requests can be hidden by a typical value.

**Recall and latency form a constraint pair, not a single ranking.** Raising k may expose more labeled chunks while increasing downstream work. Smaller chunks may improve targeting while scattering one clinical statement across boundaries. Larger chunks preserve more surrounding context but can dilute the match. The log tells you where a candidate policy landed; it does not choose the acceptable trade-off.

Use a release gate that names both requirements. For example, require no recall regression on the version-matched judged set and enforce a latency budget at the retrieval and total-request boundaries. The actual thresholds must come from the service objective and dataset, not from a copied benchmark.

Short event payloads help. They are easier to sample, retain, and inspect, and they avoid turning observability into a shadow copy of the clinical corpus.

Keep the payload boring.

## What I would change at scale

The first upgrade would be stage spans with a shared query ID: query preparation, candidate retrieval, reranking, and response assembly. Keep the event above as the summary. Spans explain a slow request; the summary supports release comparisons.

Next, move evaluation off the user request path. Replay the judged queries against a pinned index revision, store one immutable result per run, and compare a candidate chunking version with the current version before promotion. Online requests should still emit timing and identity fields, but they should not wait for labels or batch scoring.

Finally, test the instrumentation itself. A fixture with three relevant IDs and two returned matches should produce 2/3. Duplicate hits should count once. An empty judged set should remain unscored. A thrown search must emit an error event and still propagate the error. These are small tests, and they protect the metric from becoming polished fiction.

The trade-off is extra cardinality. Query IDs and chunk IDs are valuable for diagnosis but expensive in metric labels. Keep them in structured events or traces; reserve low-cardinality fields such as status, chunking version, and corpus revision for aggregate metric dimensions. This division lets a one-person team ship weekly without giving up the evidence needed to debug a bad retrieval release.

## References

- https://arxiv.org/abs/2005.11401
