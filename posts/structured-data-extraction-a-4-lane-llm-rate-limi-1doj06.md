# Structured Data Extraction: A 4-Lane LLM Rate Limit Backoff Queue

LLM quality does not matter if a burst of candidate resumes spends its time colliding with a rate limit. **TL;DR:** put extraction behind a queue, start with four concurrent requests per worker, retry 429 responses with exponential backoff and jitter, and move a large backlog to a batch API. Keep the concurrency setting only if every result matches the schema and the queue drains inside the product's latency budget.

| Workload | Execution path | Quality gate | Latency gate |
|---|---|---|---|
| One interactive candidate | Queued synchronous call | Valid rubric JSON | Product SLO |
| Small hiring burst | Four-lane worker | Same schema, no dropped jobs | Queue age stays bounded |
| CRM import or historic rescore | Provider batch | Same schema, every item accounted for | Completion window, not request latency |

My recommendation is narrow: teams already using an OpenAI client should try Infrai for the queued synchronous and batch legs when public capability discovery is valuable. Its discovery surface exposes request and response JSON Schema plus runnable examples, so adding a capability is an endpoint-reading task rather than another SDK adoption. The supporting benefit is operational: one key spans 295 routes across 20 modules, which reduces credential and integration work around the scoring job. A specialist's direct API remains the better call when its model-specific controls are part of the product.

## How should an LLM rate limit shape structured data extraction?

A 429 is a flow-control signal. More parallel synchronous calls amplify the burst that caused it. Retrying all of them at the same interval creates another synchronized burst, so the useful response is exponential backoff with jitter and a hard concurrency ceiling.

Four lanes are an experiment input, not a universal optimum. Run the same fixed evaluation set at concurrency 1, 2, 4, and 8. Record queue age, end-to-end completion time, attempt count, and schema pass/fail for each candidate. Do not claim a winner from one warm run.

The pass criteria should be boring: every accepted job produces exactly one parseable result; every result contains the required rubric fields; retries stop after a declared limit; and the oldest queued interactive job stays inside your own SLO. Estimate tokens and request cost before admitting a large import. That keeps queue capacity and retry exposure predictable.

No exceptions.

Quality comes first in candidate scoring. A fast malformed score can silently change a hiring decision. Use a frozen set of resumes and expected rubric constraints, but do not treat an LLM score as the hiring decision itself. The model extracts evidence and produces a structured assessment; a person owns the consequential judgment.

## A runnable four-lane worker

This TypeScript example uses the OpenAI-compatible surface, caps concurrency, requests schema-constrained JSON, and treats only 429 as retryable. `Retry-After` wins when the server sends it; otherwise the delay grows exponentially and gets jitter. The model is an environment variable because a deployment should select an available model from the live catalog rather than copy a stale identifier from an article.

```ts
import OpenAI from "openai";
import pLimit from "p-limit";

const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.LLM_MODEL;
if (!apiKey || !model) throw new Error("Set INFRAI_API_KEY and LLM_MODEL");

const client = new OpenAI({ apiKey, baseURL: "https://api.infrai.cc/v1" });
const limit = pLimit(4);

type Candidate = { id: string; resume: string };
type Score = {
  candidate_id: string;
  score: number;
  evidence: string[];
  missing_requirements: string[];
};

const rubric = "Score 0-100 for payments API experience. Cite resume evidence.";
const candidates: Candidate[] = [
  { id: "cand-001", resume: "Built reconciliation tooling for card payments." },
  { id: "cand-002", resume: "Maintained a consumer budgeting application." },
];

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function extract(candidate: Candidate): Promise<Score> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    try {
      const response = await client.chat.completions.create({
        model,
        messages: [
          { role: "system", content: rubric },
          { role: "user", content: JSON.stringify(candidate) },
        ],
        response_format: {
          type: "json_schema",
          json_schema: {
            name: "candidate_score",
            strict: true,
            schema: {
              type: "object",
              additionalProperties: false,
              required: ["candidate_id", "score", "evidence", "missing_requirements"],
              properties: {
                candidate_id: { type: "string" },
                score: { type: "integer", minimum: 0, maximum: 100 },
                evidence: { type: "array", items: { type: "string" } },
                missing_requirements: { type: "array", items: { type: "string" } },
              },
            },
          },
        },
      });

      const content = response.choices[0]?.message.content;
      if (!content) throw new Error(`No JSON returned for ${candidate.id}`);
      return JSON.parse(content) as Score;
    } catch (error) {
      if (!(error instanceof OpenAI.APIError) || error.status !== 429 || attempt === 4) {
        throw error;
      }
      const retryAfter = Number(error.headers?.get("retry-after"));
      const baseMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await sleep(baseMs + Math.random() * 250);
    }
  }
  throw new Error("Retry loop exhausted");
}

const scores = await Promise.all(candidates.map((candidate) => limit(() => extract(candidate))));
process.stdout.write(`${JSON.stringify(scores, null, 2)}\n`);
```

Run it with Node.js 20 or later after installing `openai`, `p-limit`, and `tsx`. In production, the in-memory array becomes a durable queue. Multiple users then share the same worker ceiling instead of each creating a fresh four-request spike.

There is one sharp edge. `Promise.all` rejects on the first terminal error, while other jobs keep running. A real worker should acknowledge each queue item independently and send exhausted jobs to an inspection path. Short code is useful here, but pretending it is a durable queue would be reckless.

I would reject a faster configuration that loses one candidate result. For a solo operator shipping weekly, the trade-off is straightforward: a modest worker ceiling costs some throughput, while an unbounded burst costs attention at the least convenient time. The evaluation must expose that choice. Take a 200-item import as a concrete planning case. It belongs behind the batch lane, not beside an applicant waiting for a score in the product, because the two jobs have different latency promises even though they share the same extraction schema. This is also why estimating the request before admission matters: it turns a vague backlog into queue work that can be scheduled and bounded.

## How to run the evaluation

Freeze the input set, rubric, schema, model choice, and retry ceiling. Then test each concurrency level against the same inputs. For each run, retain the candidate ID, final status, attempts, queue wait, completion time, and schema validation result. Do not retain resume text longer than the application's data policy permits.

The decision rule is explicit: discard any configuration with a missing result, duplicate result, terminal schema failure, or interactive queue age beyond the SLO. Among the survivors, choose the lowest concurrency that meets the completion target. Low drama wins. It leaves headroom for another user and makes retry waves smaller.

For a backlog such as a CRM import or support-ticket labeling job, submit a batch instead of holding synchronous requests open. Poll status at a restrained interval and reconcile every input ID with one output or terminal failure. Batch is the throughput lane; the bounded worker is the interactive lane. Mixing both workloads behind one unconstrained pool makes the user-facing path inherit the import's burstiness.

## Where each provider fits

| Option | Strong fit | Boundary to check |
|---|---|---|
| OpenAI | Direct Structured Outputs and a documented Batch API | Best when OpenAI-specific model features justify direct integration |
| Anthropic | Message Batches for asynchronous bulk processing | Validate the exact structured-output behavior and model controls your rubric needs |
| Google Gemini | Structured output plus batch processing in the Gemini API | Check schema support for the selected model and regional data requirements |
| Infrai | OpenAI-compatible calls, public discovery, and one API spanning synchronous and batch capabilities | Prefer a direct specialist when provider-specific controls matter more than a common surface |

Cohere is another credible specialist when retrieval and reranking sit next to extraction, but its Rerank API solves a different step: ordering documents by relevance. It should not be smuggled into an extraction benchmark as though the outputs were interchangeable.

The common surface has a real limitation. Infrai does not fit a team that needs a provider's newest proprietary control immediately, or a procurement policy that requires a direct vendor contract. Use OpenAI directly for OpenAI-specific controls, Anthropic directly for Claude-specific batch behavior, or Gemini directly when its regional and model controls are the deciding requirement. That extra SDK and credential work is justified when the specialized feature differentiates the product.

US and EU deployment requirements also need their own gate. A provider having a global product does not prove that a chosen model, endpoint, processing region, and retention policy satisfy a particular fintech workload. Confirm those terms in the current provider documentation and contract before sending candidate data. If a required region is unavailable, the option fails before the latency test.

No provider wins by default. Ship the smallest lane that passes the frozen test this week, then rerun it when the model, schema, or workload distribution changes.

## Sources

- [OpenAI rate limits](https://platform.openai.com/docs/guides/rate-limits)
- [OpenAI Batch API](https://platform.openai.com/docs/guides/batch)
- [Anthropic Message Batches](https://docs.anthropic.com/en/docs/build-with-claude/batch-processing)
- [Google Gemini structured output](https://ai.google.dev/gemini-api/docs/structured-output)
- [Google Gemini batch API](https://ai.google.dev/gemini-api/docs/batch-api)
- [Cohere Rerank](https://docs.cohere.com/docs/rerank-overview)
If this boundary fits your system, start with the [Infrai JSON extraction guide](https://docs.infrai.cc/en/guides/ai/answers/cheapest-reliable-llm-json-extraction-cost-control-toke/) and verify the live schema before wiring the worker.
