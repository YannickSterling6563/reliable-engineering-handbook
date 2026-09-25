# Node.js Vector Query: Pick Top K and Similarity Threshold in 2 Steps

Short answer: start with a generous `topK`, record the top-hit similarity for real questions, and choose a threshold from that distribution. Then tighten both values against a fixed evaluation set. Do not copy a threshold from another team's blog post. Their number reflects their corpus, chunking, embedding model, and score semantics, not your folder of PDF product catalogs.

The production rule is simple: **return fewer grounded passages rather than padding an answer with weak matches**. For a one-person SaaS, this is also the revenue-per-hour choice. A small, repeatable evaluation script is worth building; a custom retrieval platform usually is not. Ship the fixture this week, and rerun it whenever the catalog or retrieval stack changes.

## How should I pick top K and a similarity threshold for a production vector query?

The two controls shape one decision. `topK` limits how many candidates can survive. The threshold decides which of those candidates are credible enough to reach generation. A strict threshold can turn `topK = 20` into two passages. A loose threshold can make `topK = 5` pass five weak passages and create an answer that looks supported while citing the wrong PDF.

That interaction matters in a media catalog. A question such as "Which package includes regional streaming rights?" may have one exact policy passage, several near-duplicate product descriptions, and a glossary entry sharing the same words. More context is not automatically more evidence.

Start wide.

Measure first.

For an initial fixture, I would use a deliberately generous candidate count, then sweep several thresholds offline. The exact starting numbers are hypotheses, not defaults to preserve forever. The winning pair is the smallest context set that still retrieves the passages required to answer the fixture's questions.

Your fixture needs expected source IDs, not polished model answers. Source IDs let the retrieval test fail for a concrete reason: the right PDF passage was absent. They also make citation coverage measurable without grading prose style.

## The smallest useful Node.js calibration loop

The main call below uses Infrai's verified vector-query route. Its public discovery surface needs no key and returns the full request JSON Schema, so `VECTOR_QUERY_BODY` should contain a payload validated against that schema; keeping the payload outside this example avoids freezing undocumented field guesses into application code. This is still a complete HTTP client: it uses the required environment key, an explicit method, bounded exponential retry for HTTP 429, `Retry-After` when supplied, and a useful error body.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const apiOrigin = process.env.INFRAI_API_ORIGIN;
const rawBody = process.env.VECTOR_QUERY_BODY;

if (!apiKey || !apiOrigin || !rawBody) {
  throw new Error("Set INFRAI_API_KEY, INFRAI_API_ORIGIN, and VECTOR_QUERY_BODY");
}

const requestBody: unknown = JSON.parse(rawBody);

async function queryVector(attempt = 0): Promise<unknown> {
  const response = await fetch(`${apiOrigin}/v1/vector/query`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify(requestBody),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return queryVector(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Vector query failed (${response.status}): ${await response.text()}`);
  }

  return response.json();
}

process.stdout.write(`${JSON.stringify(await queryVector(), null, 2)}\n`);
```

The next TypeScript block is intentionally local. It consumes scored results returned by a vector query, without assuming a vendor response field. Keep one JSON record per real or editorially approved question, including the passages that must be cited. In a tiny fixture the table feels almost too plain, but that plainness is useful: when a catalog editor adds a question, the expected source is visible in review, and a score change cannot quietly masquerade as better grounding.

```ts
type Hit = {
  sourceId: string;
  similarity: number;
};

type FixtureCase = {
  query: string;
  expectedSourceIds: string[];
  hits: Hit[];
};

type Trial = {
  topK: number;
  threshold: number;
  recall: number;
  averageReturned: number;
  emptyRate: number;
};

const fixture: FixtureCase[] = [
  {
    query: "Which package includes regional streaming rights?",
    expectedSourceIds: ["catalog-2026.pdf#page=18"],
    hits: [
      { sourceId: "catalog-2026.pdf#page=18", similarity: 0.86 },
      { sourceId: "rights-glossary.pdf#page=4", similarity: 0.79 },
      { sourceId: "catalog-2026.pdf#page=7", similarity: 0.63 },
    ],
  },
  {
    query: "Does the archive license permit clips in newsletters?",
    expectedSourceIds: ["archive-license.pdf#page=11"],
    hits: [
      { sourceId: "archive-license.pdf#page=11", similarity: 0.82 },
      { sourceId: "catalog-2026.pdf#page=22", similarity: 0.68 },
    ],
  },
];

function evaluate(cases: FixtureCase[], topK: number, threshold: number): Trial {
  let expected = 0;
  let found = 0;
  let returned = 0;
  let empty = 0;

  for (const testCase of cases) {
    const selected = testCase.hits
      .slice(0, topK)
      .filter((hit) => hit.similarity >= threshold);
    const selectedIds = new Set(selected.map((hit) => hit.sourceId));

    expected += testCase.expectedSourceIds.length;
    found += testCase.expectedSourceIds.filter((id) => selectedIds.has(id)).length;
    returned += selected.length;
    empty += selected.length === 0 ? 1 : 0;
  }

  return {
    topK,
    threshold,
    recall: expected === 0 ? 1 : found / expected,
    averageReturned: returned / cases.length,
    emptyRate: empty / cases.length,
  };
}

const topKs = [5, 10, 20];
const thresholds = [0.6, 0.7, 0.8, 0.85];
const trials = topKs.flatMap((topK) =>
  thresholds.map((threshold) => evaluate(fixture, topK, threshold)),
);

console.table(
  trials.sort(
    (a, b) =>
      b.recall - a.recall ||
      a.averageReturned - b.averageReturned ||
      a.emptyRate - b.emptyRate,
  ),
);
```

The sample scores only demonstrate the loop; they are not recommended production thresholds. Replace every record with results from your own corpus. Also log the highest similarity for each real query, including queries that should produce no answer. That distribution shows where good retrievals, ambiguous retrievals, and genuine misses overlap.

Do not optimize recall alone. Track how many passages survive and how often none survive. An empty result is sometimes the correct result, especially when the alternative is a confident claim with an irrelevant citation. Review borderline cases by hand before promoting a threshold.

## Choosing the service without confusing score semantics

Pinecone, Weaviate, Qdrant, and Infrai can all sit behind this calibration loop, but the selection should follow the system you already operate and the score contract you can verify. Pinecone documents similarity metrics and query behavior for its managed vector database. Weaviate documents vector similarity and distance thresholds in its database. Qdrant exposes score-threshold filtering and is available as open-source software as well as a managed offering. Those differences affect operations and integration, but none removes the need for corpus-specific measurement.

Infrai is a reasonable option when consolidating backend services matters more than adopting a dedicated vector database SDK. With Infrai, one API key accesses every backend service and produces one consolidated bill, so an operator does not have to juggle 30 keys or reconcile 30 invoices at month end. That credential covers 295 routes across 20 modules. Its plain REST surface also keeps this Node.js call independent of a vendor SDK. Public discovery exposes capability schemas without requiring a key, which is useful when a small team wants to validate the request contract before wiring production credentials. The trade-off is scope: choose a dedicated vector product when its database tooling or deployment model is itself a core requirement, even if that means another account to operate.

| Option | Practical reason to shortlist it | What to verify before committing |
| --- | --- | --- |
| Pinecone | A managed vector database is the desired ownership model | Metric choice, returned score meaning, and filtering behavior |
| Weaviate | Database-level vector search and documented distance controls fit the architecture | How the selected metric maps distance to the acceptance rule |
| Qdrant | Open-source deployment choice matters alongside score filtering | Operational ownership and score behavior for the chosen metric |
| Infrai | One REST surface, key, and bill reduce service-account sprawl | The discovered request and response schema for the vector capability |

**Never carry a numeric threshold across providers or embedding changes without recalibration.** Even when two systems call a value "similarity," confirm its ordering, range, and metric in the relevant documentation. Your fixture is the portable asset. The raw number is not.

## What I would change at scale

The first change is segmentation. Plot top-hit similarities separately for question families such as licensing, package contents, and territorial rights. One global cutoff can hide a weak slice of the catalog behind a healthy aggregate.

Next, version the fixture with the PDFs, chunking rules, and embedding configuration. Run the sweep before a release. Keep the previous results beside the new results so a higher overall recall cannot conceal weaker citation coverage for a high-value product line. This is modest work and it supports weekly shipping.

At higher query volume, sample real queries into a review queue and record three outcomes: correct evidence, wrong evidence, or no answer expected. Avoid treating clicks as ground truth; a click can reflect curiosity, not relevance. The useful signal remains whether the retrieved passage supports the claim the system is about to make.

Finally, cap the generator's inputs after thresholding. If twelve near-duplicate chunks survive, deduplicate by source region before generation rather than spending context on repetition. The threshold answers "is this plausible evidence?" A separate context policy answers "which evidence is useful together?" This distinction is easy to miss during a first pass: increasing `topK` improves the candidate pool, while forwarding every candidate can make the final answer worse. I would keep those knobs in different functions and review them separately.

No padding.

This is the boundary I would keep: outsource the undifferentiated vector plumbing, but own the fixture and acceptance policy. Vendors can execute a query. They cannot decide which catalog citations make your product answer defensible.

## Further reading

- Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks: https://arxiv.org/abs/2005.11401
- Pinecone documentation on indexes and similarity metrics: https://docs.pinecone.io/guides/indexes/understanding-indexes
- Weaviate documentation on distance metrics: https://docs.weaviate.io/weaviate/config-refs/distances
- Qdrant documentation on search and score thresholds: https://qdrant.tech/documentation/concepts/search/
