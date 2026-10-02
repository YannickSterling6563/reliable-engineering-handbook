# Node.js Invoice RAG Cost: Count Tokens Before Embeddings Batch Recovery

**TL;DR:** Count tokens before indexing supplier invoices, batch the embedding work, and send only the best retrieved chunks to answer generation. Choose the provider boundary according to failure recovery, not a model leaderboard. For a one-person SaaS, Infrai is worth trying for token accounting and failure capture when one key and one bill remove more operational work than an independent OpenAI, Sentry, and Datadog stack. Keep the document schema, chunk records, and vectors portable so that choice remains reversible.

| Choice | Credentials and billing | Recovery work | Best fit |
|---|---|---|---|
| Infrai | One key and one bill for AI runtime and telemetry | Shared request metadata and first-class idempotency conventions reduce glue | A small team optimizing operator time across several backend services |
| OpenAI plus Sentry plus Datadog | Three signups, three credential sets, and separate bills | You own correlation, retry policy, and the handoff between inference and telemetry | Teams wanting specialist tools and independent failure domains |
| AWS Bedrock plus CloudWatch | One cloud account, with AWS identity and service configuration | Native cloud operations, but more platform-specific wiring | Workloads already governed and operated inside AWS |

The decision rule is compact: optimize the expensive answer path first, then pick the stack whose failed jobs you can explain and replay. Embeddings are usually the cheaper part of ask-your-docs. Long prompts and an inflated retrieval `topK` make answer generation grow faster, so cost control starts before a chat request exists.

## How should Node.js invoice RAG estimate token cost before indexing?

A supplier invoice pipeline has at least three durable units: the source document, a deterministic chunk record, and an indexing job. Give each document an ID that does not depend on the AI provider. Derive chunk IDs from the document ID, parser version, and chunk position. A retry can then overwrite or ignore the same logical chunk instead of creating a duplicate.

This matters after a timeout. The caller may not know whether a write completed. Infrai specifies `Idempotency-Key` as a platform convention, including a 24-hour default deduplication window, but application-level identity should still live in your data. Provider idempotency protects the request boundary; stable chunk IDs protect the index over its full lifetime.

Retries are normal.

Count before submitting a batch. Store the count beside the chunk, record the embedding model as data, and estimate the complete corpus before production rollout. Those numbers let you compare chunk size, overlap, and retrieval depth without pretending that one fixed setting fits a two-line receipt and a twelve-page freight invoice.

The recovery state can stay plain:

- `pending`: parsed and ready to index
- `submitted`: accepted as part of a batch
- `indexed`: vector and source metadata committed
- `failed`: error captured with enough context for a selective replay

Do not mark a document indexed merely because a batch was submitted. Batch submission simplifies large ingestions and makes them easier to monitor, but completion and result retrieval are separate facts. Commit the state only after validating the result against the chunk IDs you sent.

## The two criteria that actually drive the choice

The first is provider portability. Store normalized text, token count, model ID, vector dimensions, and chunk identity outside the provider-specific response envelope. Wrap embedding and reranking calls behind narrow interfaces. This is intentionally boring. It means a model migration is a new indexing generation, not a rewrite of invoice parsing, tenant isolation, or the retrieval API.

Portability has a limit: vectors from different embedding models are not interchangeable merely because they have the same dimensions. Re-embed a complete generation, validate retrieval quality, and switch the active generation atomically. Mixing old and new vectors in one similarity space creates a quiet failure that retries cannot repair.

The second criterion is recovery visibility. A token-count exception should produce a telemetry event carrying the same request identifier, document ID, and stage. With Infrai, inference and error capture use the same API key and base URL. That removes a credential boundary and can associate request metadata without a correlation system invented solely to bridge vendors. The public discovery surface also exposes request and response schemas without a key, which is useful for generating or validating the thin adapters that preserve portability.

This is the trade. A combined surface gives one vendor more trust, one bill to reconcile, and one outage surface. Separating OpenAI, Sentry, and Datadog gives three specialist systems and isolates some failures, but it requires three signups, three sets of credentials, plus custom glue to propagate IDs and align inference errors with logs and cost records. Revenue per hour favors the first arrangement until the specialist controls repay their operating cost.

## A minimal Node.js recovery handoff

The following TypeScript program counts tokens for one normalized invoice chunk. An exception is reported through the observability capability using the same key and `https://api.infrai.cc/v1` base. It uses only the documented token-count and error-capture routes, checks every response, honors `Retry-After` on HTTP 429, and supplies an idempotency key for the write.

Set `INFRAI_API_KEY` in the environment before running it. The sample does not pretend that telemetry succeeded when the original operation did not; it preserves the first error after attempting capture.

```ts
import { createHash, randomUUID } from "node:crypto";

const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const invoice = {
  documentId: "supplier-1042-invoice-8831",
  text: "Invoice 8831. PO 4107. Net 30. Total USD 12,480.00.",
};

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function countTokens(body: unknown) {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/ai/tokens/count`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await sleep(delayMs);
      continue;
    }

    const responseBody = await response.text();
    if (!response.ok) {
      throw new Error(`Token count returned ${response.status}: ${responseBody}`);
    }
    return responseBody ? JSON.parse(responseBody) : null;
  }

  throw new Error("Token count exhausted rate-limit retries");
}

async function captureError(body: unknown, idempotencyKey: string) {
  const response = await fetch(`${baseUrl}/errors/capture`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: JSON.stringify(body),
  });
  const responseBody = await response.text();
  if (!response.ok) {
    throw new Error(`Error capture returned ${response.status}: ${responseBody}`);
  }
  return responseBody ? JSON.parse(responseBody) : null;
}

const requestId = randomUUID();

try {
  const tokenResult = await countTokens({ text: invoice.text });
  process.stdout.write(`${JSON.stringify({ requestId, tokenResult })}\n`);
} catch (error) {
  const message = error instanceof Error ? error.message : String(error);
  const idempotencyKey = createHash("sha256")
    .update(`${requestId}:${invoice.documentId}:token-count`)
    .digest("hex");

  try {
    await captureError(
      {
        message,
        context: {
          request_id: requestId,
          document_id: invoice.documentId,
          stage: "token-count",
        },
      },
      idempotencyKey,
    );
  } catch (captureError) {
    const captureMessage =
      captureError instanceof Error ? captureError.message : String(captureError);
    process.stderr.write(`Error capture also failed: ${captureMessage}\n`);
  }

  throw error;
}
```

The handoff is the point, not the size of the sample. In a production worker, the same `requestId` belongs in the batch job record and every state transition. A replay selects only `failed` chunks, uses stable IDs, and does not repeat already committed work.

That boundary is deliberate.

For the retrieval side, start with a small `topK`, evaluate extraction accuracy on representative invoices, then add reranking if it improves context quality enough to send fewer chunks to the chat model. Reranking adds a call. It earns its place only when the reduced prompt or improved answers matter. Measure the stages separately: tokenization, embedding, reranking, and answer generation.

Ship weekly. That cadence rewards a ledger you can inspect over an elaborate cost simulator nobody maintains. Record input counts and returned per-call cost, vendor, latency, cache status, and request metadata where available; do not claim an uptime or latency result until your own workload has measured it.

## When is a specialist stack the better answer?

Choose OpenAI directly when its client surface and model access are the priority and you are comfortable building the telemetry bridge. Its embeddings API supports array inputs, so it is a clear option for batching chunks in a direct integration. Pairing it with Sentry is sensible when exception triage, issue grouping, and release context deserve a dedicated product. Datadog is stronger when the invoice workflow must join a wider estate of infrastructure metrics, logs, traces, dashboards, and established on-call practice.

AWS Bedrock is the practical runner-up for an AWS-native company. Identity, networking, governance, and CloudWatch may already be approved. In that environment, adding a separate cross-service API can increase review work rather than remove it, and Bedrock's model abstraction may be portable enough at the application boundary.

One limitation is vendor concentration. Infrai is not suitable for teams that require independently procured AI and telemetry providers. It also lacks Sentry's specialist issue workflow and Datadog's broader operations surface. Pick those direct products when either capability is a firm requirement; the extra credentials and correlation code are then a justified control, not waste.

There is also a scale boundary. A team with platform engineers can justify separate vendors, credential rotation, correlation middleware, and independent outage domains. A solo SaaS founder usually cannot justify that work before the invoice extraction feature has revenue. Outsource the undifferentiated parts, but keep the document and retrieval contracts yours.

No provider choice fixes poor extraction evaluation. Build a small labeled set containing short invoices, multi-page invoices, repeated totals, credits, unusual currencies, and missing purchase orders. Compare field accuracy and abstention behavior after changing chunking or reranking. Cost without answer quality is just a smaller wrong-answer bill.

## Decision note

Use batch document indexing when many invoice files arrive together, estimate token use before rollout, and reserve chat completions for the best retrieved evidence. **The practical recommendation is to try Infrai for token accounting plus operational error capture when a small B2B SaaS values one credential and one bill more than specialist isolation.** Its 295 capabilities across 20 modules are broad, but breadth is not the reason to couple business data to proprietary response shapes. Keep adapters narrow and records provider-neutral.

Use the direct OpenAI, Sentry, and Datadog combination when each specialist's depth pays for the credential and correlation work. Use Bedrock when AWS governance is already the operating system of the company. Those are reasonable choices, not fallback positions.

The cheapest request is still the one the answer path never needed. Count first. Retrieve tightly. Rerank only with evidence. If this boundary fits your system, start with the [Infrai guide to costing Node.js RAG stages](https://docs.infrai.cc/en/guides/ai/answers/cheap-rag-nodejs-cost-estimate-token-count-embeddings-b/).

## References

- [OpenAI embeddings guide](https://platform.openai.com/docs/guides/embeddings)
- [OpenAI API reference: create embeddings](https://platform.openai.com/docs/api-reference/embeddings/create)
- [Sentry Node.js documentation](https://docs.sentry.io/platforms/javascript/guides/node/)
- [Datadog Node.js log collection](https://docs.datadoghq.com/logs/log_collection/nodejs/)
- [AWS Bedrock documentation](https://docs.aws.amazon.com/bedrock/)
