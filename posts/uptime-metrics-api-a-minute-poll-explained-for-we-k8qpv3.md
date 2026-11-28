# Uptime Metrics API: A Minute Poll Explained for Webhook and Email Alerts

Poll a metrics API every minute, treat missing evidence as an unknown state, and send an incident through a webhook relay that can deliver email and Slack messages. For an e-commerce AI agent loop, attach the deployed release to latency, failures, and cost so the alert supports a rollback decision. Keep the rollback itself under explicit control.

**Short answer:** The smallest useful Node.js uptime monitor needs a deadline, fresh measurements, consecutive-failure detection, and notification retries. A successful metrics request alone doesn't prove that the agent is completing useful work.

## How should an uptime metrics API poll distinguish release failures from missing evidence?

Consider a shopping assistant that calls tools to assemble a cart. Its process can answer a health request while the agent loop fails to finish. The release decision needs evidence about completed work: failed loops, loop latency, and the cost of attempted work during a defined window. These are proposed measurement fields, not a universal metrics API schema.

The constraint that changes the design is rollback safety. If the metrics endpoint becomes unreachable, the monitor has lost visibility; it hasn't established that the new release caused a customer-facing failure. Classify that observation as unknown and notify the operator. If fresh measurements for the expected release cross a declared limit, classify the observation as bad and include that release in the incident. That distinction prevents a monitoring outage from becoming an automatic deployment decision. Consider a proposed test fixture where the metrics request succeeds but the payload identifies the previous release, then a second response reports a new window for the expected release with failed loops: the first observation must reset confirmation to unknown, while the second can start a bad streak, because combining their evidence would attribute an old deployment's behavior to the new one. Test that boundary explicitly. Attribution comes first.

Keep releases reversible. A green metrics request doesn't settle the rollback decision.

An application rollback also needs a separate check that the previous code can read the current data and that an agent retry cannot repeat a purchase. The poller below supplies evidence; deployment and transaction controls must enforce those requirements.

For a one-person SaaS, use a revenue-per-hour lens: the monitoring work earns its place when it helps decide whether to leave a weekly release running. Ship weekly, preserve a known rollback target, and outsource the undifferentiated transport of notifications behind a small interface. Owning another mail delivery service is a separate operating commitment.

## The minimum implementation starts with a measurement contract

The example uses TypeScript in a Node.js environment with global fetch. Its metrics endpoint must return JSON matching the Snapshot type. Put any existing API's translation in an adapter before this boundary. The endpoint reports one completed 60-second window, including failed attempts in its cost total; p95 is the loop-duration percentile computed by the producer. A zero-completion window is unknown because it cannot establish customer-path health. A separate synthetic transaction can supply evidence when traffic is quiet.

The limits below are illustrative policy choices: two consecutive bad observations, a 2,500 ms latency ceiling, and a 500,000 micro-USD cost ceiling per window. They are neither measured results nor provider prices. Replace them with a service objective and a workload-aware budget before deployment. A fixed total-cost ceiling is especially crude when order volume changes.

The webhook is an application-owned relay contract. It accepts an incident key, state, release, and summary, then routes them to configured email and Slack destinations. This JSON isn't a native payload for either destination. Implement and test those destination adapters separately, keeping their credentials out of the metrics response.

```ts
import { setTimeout as delay } from "node:timers/promises";
import { randomUUID } from "node:crypto";

type Snapshot = {
  release: string;
  windowEndMs: number;
  completedLoops: number;
  failedLoops: number;
  p95LoopMs: number;
  costMicroUsd: number;
};
type State = "healthy" | "bad" | "unknown";
type Observation = { state: State; summary: string };
type Notice = Observation & { key: string; release: string };

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}
const metricsUrl = required("METRICS_URL");
const relayUrl = required("ALERT_WEBHOOK_URL");
const release = required("EXPECTED_RELEASE");

function valid(value: unknown): value is Snapshot {
  if (typeof value !== "object" || value === null) return false;
  const s = value as Record<string, unknown>;
  const counts = [s.completedLoops, s.failedLoops];
  return typeof s.release === "string" &&
    [s.windowEndMs, s.p95LoopMs, s.costMicroUsd].every(
      n => typeof n === "number" && Number.isFinite(n) && n >= 0
    ) && counts.every(
      n => typeof n === "number" && Number.isSafeInteger(n) && n >= 0
    );
}

let lastWindow = -1;
async function observe(): Promise<Observation> {
  try {
    const response = await fetch(metricsUrl, {
      signal: AbortSignal.timeout(5_000),
      cache: "no-store"
    });
    if (!response.ok) throw new Error("metrics unavailable");
    const s: unknown = await response.json();
    if (!valid(s)) throw new Error("invalid metrics");
    const age = Date.now() - s.windowEndMs;
    if (s.release !== release || age > 120_000 || age < -5_000 ||
        s.windowEndMs <= lastWindow) {
      throw new Error("wrong release or nonfresh window");
    }
    lastWindow = s.windowEndMs;
    const summary = `release=${release}; failed=${s.failedLoops}; ` +
      `completed=${s.completedLoops}; p95_ms=${s.p95LoopMs}; ` +
      `cost_micro_usd=${s.costMicroUsd}`;
    if (s.failedLoops > 0 || s.p95LoopMs > 2_500 ||
        s.costMicroUsd > 500_000) return { state: "bad", summary };
    return s.completedLoops > 0
      ? { state: "healthy", summary }
      : { state: "unknown", summary: "no completed loops" };
  } catch {
    return { state: "unknown", summary: "metrics evidence unavailable" };
  }
}

async function notify(notice: Notice): Promise<void> {
  const response = await fetch(relayUrl, {
    method: "POST",
    headers: { "content-type": "application/json" },
    body: JSON.stringify(notice),
    signal: AbortSignal.timeout(8_000)
  });
  if (!response.ok) throw new Error("relay rejected notification");
}

async function main(): Promise<void> {
  let candidate: State = "healthy";
  let streak = 0;
  let announced: State = "healthy";
  let pending: Notice | undefined;
  for (;;) {
    const started = performance.now();
    const observation = await observe();
    streak = observation.state === candidate ? streak + 1 : 1;
    candidate = observation.state;
    if (!pending && streak >= 2 && candidate !== announced) {
      pending = { ...observation, release, key: randomUUID() };
    }
    if (pending) {
      try {
        await notify(pending);
        announced = pending.state;
        pending = undefined;
      } catch {
        console.error("Notification pending; retry on next poll");
      }
    }
    await delay(Math.max(0, 60_000 - (performance.now() - started)));
  }
}
void main().catch(() => { process.exitCode = 1; });
```

Configure METRICS_URL, ALERT_WEBHOOK_URL, and EXPECTED_RELEASE in the runtime environment. Use authenticated transport appropriate to your deployment; this minimal example assumes access control is enforced at the network boundary. Don't expose either endpoint publicly without adding authentication.

The loop schedules the next attempt relative to the start of the previous one. Requests cannot overlap inside this process. Timers aren't a hard real-time guarantee: a blocked event loop, a stopped process, or work exceeding the interval delays the next poll. The two request deadlines bound ordinary network waits, while a separate supervisor must detect a dead monitor.

## The trap is confusing an alert with delivered evidence

Two bad observations mean two distinct accepted windows here. Re-reading the same cached response doesn't count as corroboration. The freshness check allows a 120-second age and a 5-second future-clock tolerance, both explicit example assumptions that require synchronized clocks and a compatible publishing cadence. Deployments with slower aggregation need a different contract.

There is a trade-off. Waiting for confirmation suppresses isolated failures but postpones notification. A one-minute poll can also miss a short incident between observations. This is a release check with coarse evidence, not a promise to detect every failed cart operation. Keep request-level error monitoring where a single failed transaction matters.

Notification failure has its own state. The example retains the same pending key and retries on the next iteration; the relay must deduplicate that key. A successful HTTP response means the relay accepted the event under its contract. It doesn't establish that a person read an email or that every destination accepted a message. Require durable acceptance at the relay and inspect delivery results there.

Acceptance isn't delivery. The relay still has to account for each configured destination.

One pending transition deliberately blocks newer transitions until acceptance. A delayed bad notice may arrive after the service recovers; a subsequent poll can then announce recovery after its confirmation streak. This keeps the implementation small, but it is a reason to display observation time and delivery time separately in an operational relay. If delivery stays broken, the process logs a retry notice without dumping response bodies that might contain secrets. Those logs need an independent path to an operator.

## Test the decision before shipping the weekly release

Exercise the boundary with scripted responses: two fresh bad windows for the expected release should trigger one incident; two healthy windows should trigger recovery. Repeat a window, return malformed JSON, change the release identifier, and stall the endpoint beyond its deadline. Each should produce unknown evidence, without authorizing an automatic rollback. Test a quiet window separately so an idle storefront isn't silently counted as healthy.

Then make the relay accept a request but drop its response. The retry must carry the same key, and destination delivery must not multiply. Restart the monitor during an incident too: its state is currently in memory, so the confirmation history and pending notice disappear. That is an explicit boundary of this example, not a claim of durable delivery.

Keep rollback evidence free of cart contents, customer identifiers, and prompt text. Aggregate failures and latency by bounded dimensions such as operation and environment. A customer ID or arbitrary prompt as a metric label creates unbounded series; the instrumentation guidance in References explains the cardinality concern. Release attribution can live in a bounded deployment record instead of retaining a new metric label value forever.

## What I would change at scale

First, persist transitions in an outbox and make the relay's acceptance durable. Add a lease if multiple monitors share the same job; otherwise they can generate separate incident keys for the same transition. Record the observation time, window, and release with each notice.

Watch the watcher. An external heartbeat check should cover the monitor itself.

Next, replace the fixed cost ceiling with a budget that accounts for workload and unsuccessful attempts. Keep latency distributions in the producer; averaging several reported p95 values doesn't produce a combined p95. Add a synthetic cart flow that stops before a purchase, with explicit cleanup and no customer data. These additions cost operating time, so adopt them when a specific gap in release evidence justifies the work.

The decision rule stays small: fresh business failures justify investigating the release and checking rollback compatibility; missing evidence justifies restoring visibility. Keep a human confirmation step until the deployment system can enforce transaction and data safety. That is enough structure to ship weekly without turning every alert into a risky release action.

## References

The instrumentation reference supports the guidance on bounded metric labels. The remaining references document the runtime primitives used in the example. The snapshot fields, policy thresholds, relay contract, and release decision procedure above are proposed design choices.

## Further reading

- https://prometheus.io/docs/practices/instrumentation/
- https://nodejs.org/api/globals.html#fetch
- https://nodejs.org/api/globals.html#static-method-abortsignaltimeoutdelay
- https://nodejs.org/api/timers.html#timerspromisessettimeoutdelay-value-options
- https://nodejs.org/api/crypto.html#cryptorandomuuidoptions
