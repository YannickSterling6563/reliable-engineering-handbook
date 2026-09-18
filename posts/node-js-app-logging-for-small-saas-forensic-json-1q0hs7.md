# Node.js App Logging for Small SaaS: Forensic JSON Trails After Checkout Disputes

A small e-commerce SaaS should retain a compact trail of structured order transitions, then choose a logging service that can search those fields without turning routine traffic into investigation noise. TL;DR: keep the evidence needed to replay a customer's timeline, use a simple log product while that answers the question, and buy a full observability stack only when alerts, traces, or governance controls become requirements.

The deciding constraint is operator attention. For a solo founder shipping weekly, every hour spent tuning telemetry is an hour not spent shipping. Yet an order dispute still needs a defensible answer: what did checkout accept, what happened at payment, and what reached fulfillment?

That is the whole test.

## Which app logging service should a small Node.js SaaS use?

Start from the decision the incident review must support. A useful order event needs a stable event name, an ISO 8601 timestamp, an internal order ID, an internal customer reference, the attempted transition, its outcome, and the deployed application version. `trace_id` and `span_id` are useful when they already exist, but they only provide fields for manual correlation unless the chosen service has a distributed tracing query UI and span tree.

Do not log an email address merely because it is nearby. An internal customer reference limits how often personal data is copied and gives the application one place to resolve identity. This matters more when the logging service has no per-user deletion interface. The same review should ask about bulk export or subscriptions before logs become part of a GDPR workflow or downstream archive.

The signal rule is blunt: retain business state changes and unexpected outcomes; discard routine framework chatter. `payment.failed` can explain why fulfillment never started. Ten successful JSON parse messages cannot. More volume can make an incident harder to reconstruct because the meaningful transition is surrounded by events that lead to no operator action.

I would use a tiny vocabulary for the first release: three stages and three outcomes. That is nine combinations, small enough to test during every weekly deployment. It is also enough to distinguish a payment decline from a fulfillment handoff failure without serializing carts, request bodies, or payment metadata.

## A payment decline, line by line

The contract should belong to the application, not the logging vendor. This TypeScript module creates one bounded JSON event and rejects incomplete evidence before a transport sees it.

```ts
type OrderStage = "checkout" | "payment" | "fulfillment";
type Outcome = "started" | "succeeded" | "failed";

type OrderEvidence = {
  occurred_at: string;
  event: "order.transition";
  order_id: string;
  customer_ref: string;
  stage: OrderStage;
  outcome: Outcome;
  deployment: string;
  trace_id?: string;
  span_id?: string;
};

function makeOrderEvidence(
  input: Omit<OrderEvidence, "occurred_at" | "event">,
  now: Date = new Date(),
): OrderEvidence {
  if (!input.order_id || !input.customer_ref || !input.deployment) {
    throw new Error("order evidence requires stable identifiers");
  }

  return {
    occurred_at: now.toISOString(),
    event: "order.transition",
    ...input,
  };
}

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) return Number(retryAfter) * 1_000;
  return 250 * 2 ** attempt;
}

async function ingest(evidence: OrderEvidence, eventId: string): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  const baseUrl = process.env.INFRAI_BASE_URL;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");
  if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/logs/ingest`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": eventId,
      },
      body: JSON.stringify(evidence),
    });

    if (response.ok) return response.json();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`log ingestion failed (${response.status}): ${await response.text()}`);
    }

    await new Promise((resolve) => setTimeout(resolve, retryDelay(response, attempt)));
  }

  throw new Error("log ingestion exhausted its retry limit");
}

const result = await ingest(
  makeOrderEvidence({
    order_id: "ord_01JQ8N7K4R",
    customer_ref: "customer_1842",
    stage: "payment",
    outcome: "failed",
    deployment: "2026.09.18.1",
    trace_id: "4bf92f3577b34da6a3ce929d0e0e4736",
    span_id: "00f067aa0ba902b7",
  }),
  "order-ord_01JQ8N7K4R-payment-failed-01",
);

process.stdout.write(`${JSON.stringify(result)}\n`);
```

Run it with the TypeScript toolchain already used by the application, with `INFRAI_API_KEY` and the documented v1 base URL in `INFRAI_BASE_URL`. Before changing the event shape, inspect the public discovery schema rather than deriving fields from descriptive prose. The example makes at most four attempts. It honors a numeric `Retry-After` on HTTP 429; without that header, the first pause is 250 ms and doubles after each rejection. Every response is checked, and the write carries a stable idempotency key.

The idempotency detail is not decorative. A timeout after the server accepts an event leaves the client uncertain, and a blind retry can create duplicate evidence. Stable keys make the retry safe when the service supports them. If it does not, deduplicate before ingestion or accept that counts are unsuitable for decisions.

One common mistake is a generic `context` object that absorbs exception text, addresses, headers, and whole request bodies. It feels convenient during the first integration. Six releases later, field names drift and deletion scope becomes unclear. Keep the required record narrow. Put exceptional detail behind an explicit retention decision.

## Five options under the same signal test

Search quality and operating surface matter more here than a long feature checklist. These products are real alternatives, but they ask for different amounts of ownership.

| Service | Good fit | Boundary to examine |
|---|---|---|
| Better Stack | A dedicated logging workflow with JavaScript ingestion guidance and incident-management features nearby | Confirm current region, retention, and notification behavior for the account |
| Axiom | Structured event ingestion where flexible querying is a central part of incident work | Its query and dataset concepts are additional operating surface |
| Datadog | Logs that must connect to mature alerting, tracing, and broader observability workflows | Configuration breadth can be excessive for one narrow evidence trail |
| Grafana Cloud | Teams already using Grafana and willing to model logs around Loki concepts | Labels and the wider stack demand more telemetry design up front |
| Unified REST option | Centralized structured JSON logs, searchable fields, and a basic dashboard alongside other backend capabilities | No notification routing or distributed trace UI; no per-user log deletion or bulk export/subscription API |

Infrai offers **one key, one bill, and one REST API** for 295 routes across 20 modules, which fits when simple log search is one of several undifferentiated backend jobs a small product needs. It is pure HTTP, with no SDK to install, so any language or runtime can use the same contract instead of accumulating separate credentials and integrations. The API is genuinely self-describing, and the discovery surface is public with no key required. That consistency is useful to a weekly shipping cadence, but it does not turn the logging capability into a full observability suite.

Better Stack is the stronger shortlist entry when a dedicated logging and incident workflow is the priority. Axiom is attractive when event querying is the main craft the operator wants to invest in. Datadog earns its larger surface when logs must join alerts and distributed traces. Grafana Cloud makes more sense when Loki and Grafana are already part of the operating model.

Pick for the next real incident, not an imagined platform team. For this application, the practical test is whether an operator can search one order ID, read the transitions in sequence, and decide what to do without filtering pages of unrelated output.

The limitation is decisive: this unified option is not suitable when built-in alert routing, a distributed trace UI, exact per-user deletion, or a bulk export pipeline is required. Choose Datadog for a broad cross-signal program, Better Stack for a more dedicated logging and incident workflow, or Grafana Cloud when the team already operates around Grafana and Loki. I would trade the convenience of one integration for those controls because they determine whether the incident workflow can operate at all. Avoiding an SDK is secondary.

## Three exit conditions for simple logging

It stops when response depends on an active control loop. A searchable dashboard cannot wake anyone. With no threshold alerting or email, SMS, or webhook notification routing, query polling and delivery become application-owned work. That can be acceptable for a low-volume system, but it is another scheduled process to operate and verify.

No event, no warning.

A silent fulfillment reconciliation job illustrates the gap. If the task never starts, it emits no failure log. A heartbeat monitor such as Healthchecks can detect the missing run. Logs preserve what happened; a heartbeat proves expected work happened; notification routing brings a person into the loop.

The boundary is equally clear for richer debugging. Log fields named `trace_id` and `span_id` do not provide trace queries or a span tree. Source-map decoding, crash symbolication, Electron minidump parsing, and Session Replay are separate requirements. If checkout diagnosis routinely needs browser replay or cross-service causality, shortlist a product built for those workflows instead of stretching log search past its job.

Governance can force the same move. A service without per-user deletion is unsuitable when vendor-side erasure must be precise. Missing bulk export or subscription interfaces constrain archival and downstream processing. Retention and cold storage should count as controls only when the product exposes a documented configuration path.

At higher order volume, I would preserve the event contract and replace the plumbing around it. OpenTelemetry's logs data model is a sensible normalization boundary once several services or languages emit evidence. Sampling could retain every failed transition while reducing repetitive successes, but only after actual incident reviews show which success events are necessary. The release marker matters here: a record from `2026.09.18.1` must remain distinguishable after the transport changes, or the migration has damaged the evidence. I prefer that boring continuity over a clever query that only works in one dashboard.

I would also separate recent operational search from governed archival. That requires supported export or subscription behavior; it cannot be assumed. If cross-service causality becomes a daily question, add real tracing. Manual correlation is fine for an occasional order dispute and poor as a permanent operating model.

The final decision remains narrow: choose simple centralized logging when structured events can reconstruct the customer incident, its data controls match the application's obligations, and separate heartbeat and alerting tools are acceptable. Choose the broader platform when any of those conditions fail. Outsource the undifferentiated work, but keep ownership of the evidence contract.

## References

- [OpenTelemetry, Logs signal concepts](https://opentelemetry.io/docs/concepts/signals/logs/)
- [Better Stack, JavaScript logging documentation](https://betterstack.com/docs/logs/javascript/)
- [Axiom, Send logs from JavaScript](https://axiom.co/docs/send-data/javascript)
- [Datadog, Log Management documentation](https://docs.datadoghq.com/logs/)
- [Grafana Loki documentation](https://grafana.com/docs/loki/latest/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
