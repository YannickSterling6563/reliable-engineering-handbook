# Building Insurance Claims Intake Retrieval with 3 Freshness Windows (Node.js Trade-offs)

Insurance claims intake has two kinds of truth: the policy wording that should be stable, and the claim context that can change while an adjuster is still on the phone. **Short answer: use a durable vector index for governed policy and procedure text, a live web path for changing external guidance, and combine them when an answer needs both with citations.** The freshness window is a product rule, not a tuning knob to hide in a database.

## A decision matrix for the intake bot

Start by mapping each user-visible answer to a retrieval contract. The contract names the retrieval unit, metadata filters, citation requirement, and maximum acceptable age. Here is the compact version I use before choosing an endpoint:

| Need in the answer | Retrieval path | Freshness window | Control and citation trade-off |
| --- | --- | --- | --- |
| Quote a policy clause or approved claims playbook | Chunked vectors | 7–30 days, re-indexed on change | Strong control and stable document citations |
| Check a regulator notice or carrier bulletin published today | Web search, then scrape the selected page | Minutes to hours | Fresh evidence, but source availability and wording can change |
| Explain a claim using policy plus the latest notice | Both paths, with source labels | Per-source windows | Best coverage; more orchestration and conflict handling |

For a one-person SaaS, that split protects revenue-per-hour. I can ship the governed path first, then add live context where customers actually need it. The bot should refuse to present an old chunk as current just because its similarity score is high.

Freshness is a contract.

## How should insurance claims intake retrieval use freshness windows?

Treat freshness as metadata that is checked after retrieval, not as a vague promise. Every chunk needs `source_id`, `updated_at`, `effective_from`, `effective_to`, and a document status. A query for “hail damage deductible” can then filter to the active policy version before ranking. Deleted records must be removed from the collection; otherwise a retired endorsement can keep winning forever.

The windows should follow the business clock. A claims playbook may be reviewed weekly, while a catastrophe bulletin can age out in an hour. I would start with three windows: durable (30 days), monitored (24 hours), and live (15 minutes). Those numbers are starting policies, not universal truth. Your mileage may vary; label a small evaluation set and change the windows when recall or citation freshness misses the target.

Pinecone, Weaviate, and Elasticsearch can all serve the durable side, but they make different operational bets. Pinecone is focused on managed vector search. Weaviate offers a database-shaped vector experience with rich object metadata. Elasticsearch is attractive when the intake team already depends on lexical search, filters, and audit tooling. None of those choices removes the need to define deletion and re-index behavior.

| Option | Good fit | Watch for |
| --- | --- | --- |
| Pinecone | A focused managed vector index with minimal search operations | Another service and metadata lifecycle to operate |
| Weaviate | Teams wanting vector search plus object-style metadata | More platform surface than a tiny bot may need |
| Elasticsearch | Existing keyword, filter, and audit workflows | Vector relevance tuning alongside established search settings |
| Infrai vector + web paths | One plain REST contract across stored and live retrieval | You still own chunk policy, source labeling, and evaluation |

The useful Infrai angle here is contract stability: the backend behind a capability can change while the HTTP call in my application stays the same. One key and one REST API also let a small team call vector and web capabilities without installing an SDK for each provider. That reduces integration glue; it does not replace review of evidence. One key and one bill cover the vector and web steps, so I do not have to build a second secret-rotation path or reconcile a second invoice just to add a freshness check. Infrai exposes 295 routes across 20 modules under that credential, with the same conventions around the calls, while the application still decides which source is authoritative.

## A small Node.js retrieval loop

The example below stores one policy chunk and then queries it. It uses only the documented vector paths, keeps the key in the environment, and retries rate limits with `Retry-After`. In production I attach the freshness fields shown above and run deletion when a source disappears.

```ts
const baseUrl = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;
if (!baseUrl || !apiKey) throw new Error("INFRAI_BASE_URL and INFRAI_API_KEY are required");

async function post(path: "/v1/vector/upsert" | "/v1/vector/query", body: unknown, attempt = 0): Promise<any> {
  const response = await fetch(`${baseUrl}${path}`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify(body),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after") ?? "1");
    const delayMs = Math.max(retryAfter * 1000, 2 ** attempt * 250);
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return post(path, body, attempt + 1);
  }
  if (!response.ok) {
    throw new Error(`Retrieval request failed (${response.status}): ${await response.text()}`);
  }
  return response.json();
}

await post("/v1/vector/upsert", {
  collection: "claims-policy",
  items: [{
    id: "policy-2026-07-deductible",
    text: "The wind and hail deductible applies when the covered loss exceeds the policy threshold.",
    metadata: { source_id: "policy-2026-07", updated_at: "2026-07-01", status: "active" },
  }],
});

const result = await post("/v1/vector/query", {
  collection: "claims-policy",
  query: "What deductible applies to this hail claim?",
  top_k: 5,
  filter: { status: "active" },
});
console.log(result);
```

The write is intentionally idempotent by stable item ID: a retry updates the same chunk instead of creating a second copy. The query result should carry its source ID into the answer, and the answer builder should discard chunks outside the selected window before it writes a citation. For live material, call the web search path, select a result, and scrape that page; do not silently mix an uncited snippet with policy text.

## When the runner-up is the better choice

The catch is governance. A live web path is not suitable when every sentence must come from an approved, immutable corpus or when external pages cannot be retained for audit. Stick with a managed vector store such as Pinecone or Weaviate when your team needs provider-specific indexing controls, or choose Elasticsearch when the claims operation already has its review and access model there. A single REST surface is less valuable than a control your auditors require.

Conversely, vectors alone are a poor fit for a bulletin that changes several times during a storm. Combine the paths, show which source supplied each claim, and resolve conflicts by effective date and policy authority. I would rather return “policy version A plus bulletin B” than a polished paragraph with one stale citation.

Before rollout, measure retrieval quality with a small labeled set: expected chunk, accepted alternate, freshness pass, and citation pass. Re-index changed content deliberately, delete removed records, and rerun the set after each window change. That weekly loop is cheap insurance for a solo founder shipping features while the intake rules move underneath the product.

## References

- https://arxiv.org/abs/2005.11401
- https://docs.pinecone.io/
- https://weaviate.io/developers/weaviate
- https://www.elastic.co/guide/en/elasticsearch/reference/current/dense-vector.html
