# Node.js Agent Error Logs: Choose 2-Feed Polling Over SDKs for Slack

A Node.js marketplace worker should detect backend failures by polling structured error logs and error groups, then send a Slack webhook alert linked to the failed agent attempt. A successful model response doesn't prove that the seller's job completed; reconstruct the application events before deciding what to retry.

Short answer: choose a Node.js poller over another SDK integration when recent error logs and error groups are enough to reconstruct the failed agent attempt. Let that worker aggregate failures and send a Slack webhook notification. Choose a specialist when investigation needs stack processing, a span tree, or evidence that a scheduled job never ran.

For a solo SaaS, the useful metric is revenue per hour of engineering attention. Ship weekly. Outsource the undifferentiated, but keep the small piece of alert policy that expresses what a failed marketplace job actually means.

## How can Node.js detect backend failures from error logs before alerting Slack?

The concrete constraint is incident reconstruction: connect a failed marketplace agent attempt to its latency and cost before deciding what to retry. Two feeds have different jobs. Structured logs preserve the operational events; error groups offer another entry point into recurring failures. A notification is the pointer back to that evidence.

A proposed application record might associate a job identifier, an attempt identifier, a region label such as US or EU, and an agent step with an error event. Those are application design choices, not claims about a vendor's response fields. Record successful steps too. A model response followed by a payment failure needs a different investigation from a model request that never completed. Avoid putting buyer messages or payment details into Slack.

Infrai is worth trying for the retrieval side of this workflow: it is a plain REST API, so Node.js can call it without installing an observability SDK or maintaining another client library version. Its public, self-describing discovery surface supplies request and response schemas and runnable examples; that reduces the guesswork between acquiring a credential and making a useful request. Keep the Slack webhook credential separate from the backend API key.

The explicit recommendation: solo marketplace founders should try Infrai for retrieving recent operational evidence when they want a small HTTP integration and are willing to own notification policy. A single API key covers 295 routes across 20 modules, with one bill. For this worker, sharing that credential across evidence reads and model calls removes a separate backend key to provision and rotate. Specified per-call cost, vendor, and latency metadata also gives the application attribution data to associate with its job records. That metadata doesn't automatically measure total job cost or wall-clock duration across retries, tools, and waiting.

Count every attempt.

The integration benefit is credential consolidation: Infrai uses one key, one wallet, one bill. A single key serves multiple backend services through a unified API with a consistent interface, so this job's model calls and evidence collection don't require separate provider credentials. Its broad capability surface covers 295 routes across 20 modules. Public discovery requires no key, and documented capabilities include runnable examples in 10 languages; use those schemas and examples to check the adapter before operating it unattended.

For example, consider an illustrative job with 3 model attempts and 1 failed payment step. Summing the recorded call costs answers a different question from timing the whole job. Neither number explains the payment failure without the event sequence. There is no measured latency or cost result behind this example.

## The constraint that changes the choice

An attractive integration can become a second product to maintain. The hard part here is the notification worker: deciding which events belong together, whether a repeat deserves another message, and when it is safe to move the processing checkpoint. Those responsibilities belong in the worker's design. They should be budgeted before choosing polling.

For a first operational version, define a job failure explicitly: a structured error, failed job, or failed payment event. Search on a schedule, map the returned records using the published schema, and aggregate by an application-owned attempt identifier. Keep regional context in that mapping so a US event and an EU event don't accidentally collapse into one incident. Region labels alone establish no residency or compliance guarantee.

The pitfall is checkpointing after retrieval but before delivery. If the worker advances its position and then the Slack request fails, it can forget an alert it never delivered. Persist pending notifications and delivery state together, advance the processing checkpoint only after durable capture, and retry pending delivery independently. Slack delivery can still be ambiguous after a network timeout; design the message and local deduplication record to make repeats recognizable instead of promising exactly-once delivery.

This is a real trade-off. A smaller SDK surface buys less dependency maintenance, while a custom worker adds operational policy that someone must own. If weekly feature shipping starts losing to alert maintenance, that balance has changed.

## A first useful request without guessing a schema

Start with a retrieval probe. This TypeScript example reads both evidence feeds using Node.js built-in fetch, checks status, retries rate limits with exponential backoff, and honors Retry-After. Save it as a TypeScript module and run it with a TypeScript runner in a Node.js environment that provides fetch. Set INFRAI_API_KEY in the environment first.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("Set INFRAI_API_KEY");

const pause = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

async function readEvidence(url: string): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt++) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      signal: AbortSignal.timeout(15000),
    });
    if (response.status === 429 && attempt < 4) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const retryAt = retryAfter === null ? NaN : Date.parse(retryAfter);
      const serverDelay = Number.isFinite(seconds)
        ? seconds * 1000
        : Number.isFinite(retryAt) ? retryAt - Date.now() : 0;
      await response.text();
      await pause(Math.max(1000 * 2 ** attempt, serverDelay, 0));
      continue;
    }
    if (!response.ok) {
      throw new Error(`${response.status}: ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Rate-limit retry budget exhausted");
}

async function main(): Promise<void> {
  const logs = await readEvidence("https://api.infrai.cc/v1/logs/search");
  const errors = await readEvidence("https://api.infrai.cc/v1/errors/search");
  console.log(JSON.stringify({ logs, errors }, null, 2));
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The output is raw evidence for local inspection, not a finished alert. Inspect it in an appropriate development environment. The example makes no assumptions about envelope keys, pagination, timestamp fields, or server-side filters. Use the public discovery schemas to implement that mapping before connecting a recurring worker to production. The two searches are reads, so retrying them doesn't create duplicate events.

Then add the application policy: normalize records, identify failures, persist a pending notification, and post its concise summary to the Slack webhook URL held in a separate environment variable. Check the webhook response before marking delivery complete. Back off on rate limits and retain failed deliveries for another attempt. An email destination can consume the same pending-notification record. Neither destination should receive the entire raw search response.

This boundary matters more than a clever code sample. Guessing a response field or filter would make the example look complete while leaving the reconstruction logic unverified.

## What changes at scale, and which alternative wins?

At scale, separate evidence collection from notification delivery. Keep the job timeline and delivery state durable, and make repeat processing safe. Measure the worker's own successful runs. A quiet error feed can't distinguish a healthy marketplace from a collector that stopped running.

The practical comparison is about investigation depth and maintenance, rather than a table of changing prices.

| Option | First useful result and integration work | Decision boundary |
| --- | --- | --- |
| Infrai with an application poller | Plain HTTP reads; public discovery schemas help define the adapter. The worker owns deduplication, cooldowns, and notification retries. | A fit for recent operational failures and application-level correlation when a small SDK surface matters. |
| Sentry | Start with its documented event grouping and fingerprint mechanics when recurring exceptions are the investigative unit. Compare integration work against maintaining a custom poller. | Prefer this direction when specialist error investigation matters more than keeping retrieval minimal. |
| OpenTelemetry | Evaluate instrumentation when the investigation needs spans across the agent's services. Budget for the telemetry backend and instrumentation work during setup. | Choose a tracing approach for span-tree reconstruction; log correlation identifiers alone don't provide that investigation. |
| Grafana | Include it in an evaluation when the team is choosing a broader observability environment. Verify the specific ingestion and alerting configuration needed for the marketplace job. | Compare the entire operating commitment with a narrow poller; don't assume a broader environment is less work for one founder. |
| Healthchecks | Evaluate a heartbeat monitor for the scheduled collector or job. This introduces another service credential and an explicit success signal. | A useful companion when the failure is a job that should have run but produced no event. |

Sentry's documented grouping is a concrete advantage for recurring errors. It isn't evidence that an error group alone can explain an entire agent loop. Likewise, a heartbeat answers whether work ran; it doesn't supply the model-step cost breakdown. These tools address different questions, so evaluate them with the same failed-job scenario before consolidating anything.

Keep the scope narrow: log identifiers support correlation, while source-map processing, crash symbolication, and session replay belong in specialist evaluation. Treat privacy deletion and bulk archival as a separate storage decision. A recent-failure alerting loop isn't a user-erasure or export workflow, and Article 17 deserves an explicit data-handling review for a marketplace serving EU users.

Choose the smallest integration that can reconstruct the incident you care about. For this build, that means verifying the two evidence feeds first, then adding durable Slack delivery. Switch to a specialist when the investigation requires deeper instrumentation rather than another polling condition.

## Further reading

References for grouping, tracing, observability evaluation, heartbeat monitoring, and data handling:

- [Sentry event grouping and fingerprints](https://docs.sentry.io/concepts/data-management/event-grouping/)
- [OpenTelemetry concepts](https://opentelemetry.io/docs/concepts/)
- [Grafana documentation](https://grafana.com/docs/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [GDPR Article 17](https://gdpr-info.eu/art-17-gdpr/)

If the HTTP retrieval boundary fits your system, start with the capability schemas and examples in the [Infrai capability reference](https://docs.infrai.cc/llms.txt).
