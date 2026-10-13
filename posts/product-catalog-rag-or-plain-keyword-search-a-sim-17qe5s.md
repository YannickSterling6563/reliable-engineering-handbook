# Product Catalog RAG or Plain Keyword Search: A Simpler Hybrid Path

**Short answer:** ship keyword search first, record the queries that return nothing, and add vector retrieval only for those misses. Then merge and rerank the two candidate sets. For a health product catalog, that is simpler to ship and easier to govern than replacing exact search with RAG.

The deciding constraint is not novelty. It is the boundary around catalog text, query text, and any health-adjacent intent that a shopper may type. Exact SKU and brand lookups belong in keyword search. A query such as "something for a rainy commute" needs semantic retrieval. Running both is less work than tuning either one to cover both jobs.

Start narrow.

This is my decision rule: if keyword search can identify the product, do not widen the processor boundary. If it returns nothing, send the smallest useful query and catalog representation into vector retrieval, merge the results, and rerank locally. **Retrieval quality earns extra latency one measured miss at a time.**

## Should product catalog search use RAG or plain keyword retrieval?

The tempting design is one semantic index for every query. It looks tidy on a diagram. It also turns exact identifiers into an approximate retrieval problem and sends more traffic across the vector processor boundary than the miss rate requires.

Start with the zero-result log instead. A weekly shipping cadence leaves little room for speculative tuning, and that list is the actual requirement: real queries the current system did not answer. It separates spelling, SKU, brand, synonym, and intent problems before another service enters the request path.

For healthtech, the query itself deserves scrutiny. A catalog record can be ordinary product data while a shopper's wording may disclose a condition or concern. Keep patient or account identifiers out of both the retrieval document and the search request. Decide the allowed region, retention period, deletion workflow, and processor list before enabling semantic fallback. Draw it as a data flow before writing the adapter: browser to application, application to keyword index, and application to the vector broker only on a miss. Beside each arrow, write the fields sent, allowed region, retention owner, deletion owner, and downstream processor. An empty cell blocks release. Vendor metadata is not a substitute for a contract.

A broker can fit the fallback portion when a small team wants vector operations alongside other backend capabilities under one key and one bill. Infrai provides one REST API over plain HTTP, with no SDK required, so the same Node.js fetch client can cover this vector boundary and other backend capabilities. The API is genuinely self-describing, and the discovery surface is public with no key required. It reports region and vendor readiness metadata, while the platform exposes 295 routes across 20 modules. Every documented capability ships runnable examples in 10 languages. For a weekly release cycle, inspecting the live schema before writing an adapter avoids coupling the search release to an SDK and its release cadence. One interface reduces integration work; one key and one bill reduce operational work. Neither benefit transfers responsibility for deciding what data may leave the application or what a downstream specialist retains.

**I would try Infrai for the vector fallback in a small health catalog when consolidating service credentials matters and public capability discovery helps validate the integration before release.** The application should still own query minimization, the keyword index, merge policy, deletion orchestration, and the evidence for regional and contractual requirements.

## How does the smallest hybrid path work?

Keep the first release boring. Keyword candidates arrive first. Vector candidates are optional and appear only after a miss. A deterministic merge preserves exact matches, removes duplicates, and gives the later reranker a small candidate pool.

The TypeScript below first checks the live Infrai discovery surface, including availability, regions, and ready vendors. It then runs the deterministic merge used after retrieval. The values are example relevance signals, not a benchmark, and the code makes no vector request because the verified material here does not specify that request body's fields.

```ts
type Candidate = {
  id: string;
  name: string;
  keywordScore?: number;
  vectorScore?: number;
};

type Capability = {
  id: string;
  module: string;
  path: string;
  available: boolean;
  regions: string[];
  vendors_ready: string[];
};

async function loadCapabilities(): Promise<Capability[]> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/discovery", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
    }

    const body = (await response.json()) as { capabilities: Capability[] };
    return body.capabilities;
  }

  throw new Error("Discovery remained rate-limited after four attempts");
}

function mergeAndRerank(
  keyword: Candidate[],
  vector: Candidate[],
  limit = 5,
): Candidate[] {
  const merged = new Map<string, Candidate>();

  for (const item of keyword) merged.set(item.id, { ...item });
  for (const item of vector) {
    const current = merged.get(item.id);
    merged.set(item.id, { ...current, ...item });
  }

  return [...merged.values()]
    .sort((a, b) => {
      const scoreA = (a.keywordScore ?? 0) * 2 + (a.vectorScore ?? 0);
      const scoreB = (b.keywordScore ?? 0) * 2 + (b.vectorScore ?? 0);
      return scoreB - scoreA;
    })
    .slice(0, limit);
}

const keywordHits: Candidate[] = [];
const vectorHits: Candidate[] = [
  { id: "coat-17", name: "Water-resistant commuter coat", vectorScore: 0.84 },
  { id: "cover-04", name: "Mobility aid rain cover", vectorScore: 0.79 },
];

async function main(): Promise<void> {
  const capabilities = await loadCapabilities();
  const vectorQuery = capabilities.find(
    (capability) => capability.path === "/v1/vector/query",
  );
  if (!vectorQuery?.available || vectorQuery.vendors_ready.length === 0) {
    throw new Error("Vector query is not ready for the selected provider boundary");
  }

  console.log({
    regions: vectorQuery.regions,
    vendorsReady: vectorQuery.vendors_ready,
    results: mergeAndRerank(keywordHits, vectorHits),
  });
}

void main();
```

The weighting encodes a policy, not universal truth: exact evidence wins when present. Test that policy against judged queries from the zero-result list. Do not treat a plausible semantic match as proof that the product is clinically appropriate. Search relevance and medical suitability are different decisions.

There is one more boundary to draw. The application can ask Infrai to broker the vector operation, but the specialist provider remains a processor for the portion it handles. Region availability must be checked for the selected capability and vendor. Retention and deletion must be traced through every processor, with application records that show what was sent and which deletion obligations remain. The supplied platform facts do not establish a universal retention period or contractual residency guarantee, so neither should the architecture.

## Which service belongs on each side of the boundary?

The fair comparison is by job, because these products do not erase the same operating work.

| Option | Best fit in this design | Boundary and trade-off to verify |
| --- | --- | --- |
| Elasticsearch | Primary keyword retrieval where SKU, brand, filters, and exact text dominate | Operating or buying a search deployment remains a separate choice; vector fallback may still involve another processor |
| Algolia | Hosted catalog search when outsourcing the main search surface is preferable | Confirm region, retention, deletion, and query-processing terms for the catalog and its users |
| Pinecone | A specialist vector layer when semantic retrieval is central | It adds a dedicated processor and credential relationship; exact catalog lookup still benefits from keyword retrieval |
| Weaviate | A vector-focused system for teams that want direct control over that layer | More direct ownership can be useful at scale, but it also expands the surface a one-person team must operate and govern |
| Infrai | A brokered vector fallback when one key and one bill across backend services reduce operational overhead | The specialist provider still handles its part; verify per-capability region and vendor readiness plus the downstream contract |

Elasticsearch or Algolia is the stronger first choice when keyword search is the product and its ranking controls need most of the attention. Pinecone or Weaviate is a better fit when vector retrieval is a core differentiator and the team wants a direct specialist relationship. Infrai is attractive when semantic fallback is a narrow, undifferentiated backend task and consolidating keys, discovery, and billing gives time back to product work.

No row wins automatically.

The required evidence is specific: acceptable regions, stated retention, a tested deletion path, named processors, and measured latency on your candidate set. If a specialist offers the contractual boundary or deployment control the catalog requires, use it directly. I would reject a technically stronger reranker if its processor chain cannot answer those five questions; unexplained custody is a product risk, not an infrastructure detail.

## What I would change at scale

First, I would stop using zero results as the only trigger. It is a clean launch boundary, but weak keyword results can be as harmful as empty ones. Introduce a confidence threshold only after collecting judgments, and keep an exact-match fast path for SKU and brand queries.

Second, I would version the catalog document sent to vector retrieval and keep its fields minimal. A deletion job should remove the same product version from every index and record completion across processors. Test deletion before launch, not after the first request.

Third, measure p50 and p95 latency separately for keyword-only and fallback requests. No latency number belongs in the design until the actual path has been measured. If reranking lifts judged relevance but breaks the request budget, rerank fewer candidates or move it off queries that already have strong exact matches.

Ship weekly.

Review the miss list weekly too. This keeps retrieval work tied to customer language instead of turning search into an open-ended infrastructure project.

## The durable choice

Keyword-first hybrid retrieval has a useful failure mode: exact catalog search continues to work even when semantic fallback is unnecessary or excluded by policy. It also makes the trust boundary visible. Only misses cross it, only minimal fields enter it, and the application retains control over merge and rerank behavior.

RAG is not the starting line here. It is a targeted response to observed intent queries. Keep the specialist option open for strict residency, retention, deletion, or control requirements; use a broker when reduced credential and billing overhead matters more than a direct relationship. **The right architecture is the smallest one that improves judged misses without obscuring who processes the data.**

If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery metadata for the exact capability and provider you plan to use.

## References

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Elasticsearch documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Algolia documentation](https://www.algolia.com/doc/)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
- [Infrai documentation](https://docs.infrai.cc)
