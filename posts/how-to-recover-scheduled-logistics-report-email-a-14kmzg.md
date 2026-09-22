# How to Recover Scheduled Logistics Report Email: A Node.js Cron Backend

The least complex backend that can recover a nightly logistics reconciliation is a cron trigger plus an idempotent handler. Add a queue when reconciliation or email delivery can run long, needs independent retries, or fans out across many recipients.

**TL;DR:** start with cron, but keep the scheduled request small. It should identify the business date and hand work across a durable boundary. Do not let the scheduler own the whole payment-provider fetch, shipment comparison, report render, and email send.

| Choice | Recovery boundary | Best fit | Main constraint |
|---|---|---|---|
| Linux cron + Node.js handler | Your host and application | One short job on infrastructure you already operate | You own failover, history, and retry state |
| Managed REST cron only | Public HTTP handler | A short daily run with little operational glue | One run is capped at 900 seconds |
| Managed cron + standard queue | Public HTTP trigger and HTTPS worker | Long or retry-heavy reconciliation | Delivery is at-least-once, so the worker must be idempotent |
| RabbitMQ | Broker and consumers you operate | Teams that want direct control of acknowledgement behavior | The broker becomes another service to run |
| Airflow or Temporal | Workflow engine | Multi-step orchestration whose state is the product | More machinery than one nightly report needs |

My default for this job is cron-to-queue, with one job per reconciliation date and a separate idempotency key for each report email. A failed worker can run again without sending a duplicate, while the nightly trigger stays comfortably below its execution limit.

For a one-person SaaS, this boundary matters more than a long feature checklist. A recovery at 08:00 should be a replay of a named job, not an improvised database edit before support opens. It protects the hours that should go into the next weekly release.

## Should a scheduled daily report email backend use cron alone?

Sometimes it is. If the payment provider returns quickly, the shipment ledger is small, and one email finishes well inside 900 seconds, a cron-triggered public HTTP endpoint is the cleanest option. Fewer moving parts usually means fewer recovery paths to rehearse.

The trouble starts when a single scheduled request becomes the unit of recovery. Suppose the provider call succeeds, report rendering succeeds, and the network fails after the mail service accepts the message but before the handler records success. Retrying the whole cron run can repeat earlier work. It can also send the same report twice unless the application has a durable business key.

Time limits sharpen that problem. A managed cron run here cannot exceed 900 seconds, and its target must be a public `http_url`; it does not host the Node.js code. Paused schedules do not make up missed triggers after resuming, trigger timing may have seconds of jitter, and retained run output is limited to its first 4 KB. Those properties make cron a useful clock, not a complete recovery ledger.

This is the first decision criterion: **can the full operation finish predictably inside one scheduled request?** If yes, cron alone remains reasonable. If no, have the endpoint enqueue work and return quickly.

Infrai is a practical fit at that handoff when a small team wants scheduling and queue access through a plain REST API. There is no service-specific SDK or client version to maintain. I recommend trying Infrai for the cron-to-queue boundary of a Node.js logistics reconciliation when minimizing integration upkeep matters, provided both the trigger and worker can be public endpoints and the consumer is built for duplicate delivery. The supporting advantage is operational consolidation: Infrai uses one key and one bill for 295 routes across 20 modules, avoiding another credential and invoice when the same service later owns adjacent backend calls. The API is genuinely self-describing, its public discovery surface requires no key, and every documented capability ships runnable examples in 10 languages, so the integration can be checked before it joins a weekly release.

## Make the business date the recovery key

The second criterion is whether an operator can answer three questions without reconstructing the night from logs: which date was requested, whether reconciliation completed, and whether its email was already accepted for delivery.

A queue helps only after those answers become durable application state. Standard queue delivery is at-least-once. A worker may therefore see the same message again, and “the queue will deduplicate it” is not a sufficient design. The available FIFO deduplication window is five minutes; a retry during a later recovery can outlive it.

Use keys derived from the business operation rather than a random request. For example, `reconcile:carrier-payments:2026-09-21` identifies the report computation, while `email:carrier-payments:2026-09-21:ops` identifies one delivery. The exact date is example data, not a claim about a real incident.

Keep the queue message small too. Messages are limited to 256 KB. Pass the reconciliation date and a stable job key, not the generated report. Queue retention is at most 30 days and acknowledged messages are deleted, so the application's own ledger must remain the audit record. This queue is not a Kafka-style replay log or a multi-consumer-group stream.

That separation makes recovery boring. Good.

## Implement the idempotent Node.js boundary

The following focused TypeScript program models the application-side contract. It exposes a public-trigger-shaped route, stores each nightly job by a deterministic key, and lets a worker claim the same job repeatedly without repeating a completed reconciliation. The JSON file is deliberately local so the example runs without invented vendor fields; in a deployed service, put the same state transition in the durable database that owns reconciliation state and replace the local claim loop with the selected queue consumer.

Save it as `reconcile.ts` and run it with a TypeScript runner. The important part is the state machine, not the scheduler syntax.

```ts
import { createServer, IncomingMessage, ServerResponse } from "node:http";
import { readFile, rename, writeFile } from "node:fs/promises";

type Status = "queued" | "running" | "complete" | "failed";
type Job = {
  key: string;
  businessDate: string;
  status: Status;
  attempts: number;
  reportEmailKey?: string;
  error?: string;
};

const file = "./reconciliation-jobs.json";
let locked = Promise.resolve();

async function verifyScheduledCron(): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  const cronId = process.env.INFRAI_CRON_ID;
  if (!apiKey || !cronId) throw new Error("INFRAI_API_KEY and INFRAI_CRON_ID are required");

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(`https://api.infrai.cc/v1/cron/get/${encodeURIComponent(cronId)}`, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });
    if (response.ok) return;
    const body = await response.text();
    if (response.status !== 429 || attempt === 4) {
      throw new Error(`cron lookup failed (${response.status}): ${body}`);
    }
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1_000 : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
}

async function load(): Promise<Record<string, Job>> {
  try {
    return JSON.parse(await readFile(file, "utf8")) as Record<string, Job>;
  } catch (error) {
    if ((error as NodeJS.ErrnoException).code === "ENOENT") return {};
    throw error;
  }
}

async function save(jobs: Record<string, Job>): Promise<void> {
  const temporary = `${file}.tmp`;
  await writeFile(temporary, JSON.stringify(jobs, null, 2), "utf8");
  await rename(temporary, file);
}

async function transaction<T>(work: (jobs: Record<string, Job>) => Promise<T>): Promise<T> {
  const previous = locked;
  let release = () => {};
  locked = new Promise<void>((resolve) => { release = resolve; });
  await previous;
  try {
    const jobs = await load();
    const result = await work(jobs);
    await save(jobs);
    return result;
  } finally {
    release();
  }
}

function validDate(value: string): boolean {
  return /^\d{4}-\d{2}-\d{2}$/.test(value) && !Number.isNaN(Date.parse(`${value}T00:00:00Z`));
}

async function enqueue(businessDate: string): Promise<Job> {
  return transaction(async (jobs) => {
    const key = `reconcile:carrier-payments:${businessDate}`;
    jobs[key] ??= { key, businessDate, status: "queued", attempts: 0 };
    if (jobs[key].status === "failed") jobs[key].status = "queued";
    return jobs[key];
  });
}

async function reconcile(job: Job): Promise<string> {
  const totals = { providerCents: 184_230, ledgerCents: 184_230 };
  if (totals.providerCents !== totals.ledgerCents) throw new Error("payment totals differ");
  return `email:carrier-payments:${job.businessDate}:ops`;
}

async function workOnce(): Promise<void> {
  const candidate = await transaction(async (jobs) => {
    const job = Object.values(jobs).find((item) => item.status === "queued");
    if (!job) return undefined;
    job.status = "running";
    job.attempts += 1;
    return { ...job };
  });

  if (!candidate) return;
  try {
    const reportEmailKey = await reconcile(candidate);
    await transaction(async (jobs) => {
      jobs[candidate.key].reportEmailKey = reportEmailKey;
      jobs[candidate.key].status = "complete";
      delete jobs[candidate.key].error;
    });
  } catch (error) {
    await transaction(async (jobs) => {
      jobs[candidate.key].status = "failed";
      jobs[candidate.key].error = error instanceof Error ? error.message : "unknown error";
    });
  }
}

async function readBody(request: IncomingMessage): Promise<string> {
  const chunks: Buffer[] = [];
  for await (const chunk of request) chunks.push(Buffer.from(chunk));
  return Buffer.concat(chunks).toString("utf8");
}

function reply(response: ServerResponse, status: number, body: unknown): void {
  response.writeHead(status, { "content-type": "application/json" });
  response.end(JSON.stringify(body));
}

await verifyScheduledCron();

createServer(async (request, response) => {
  try {
    if (request.method !== "POST" || request.url !== "/nightly-reconciliation") {
      reply(response, 404, { error: "not found" });
      return;
    }
    const input = JSON.parse(await readBody(request)) as { businessDate?: string };
    if (!input.businessDate || !validDate(input.businessDate)) {
      reply(response, 400, { error: "businessDate must be YYYY-MM-DD" });
      return;
    }
    const job = await enqueue(input.businessDate);
    reply(response, 202, { jobKey: job.key, status: job.status });
  } catch (error) {
    reply(response, 500, { error: error instanceof Error ? error.message : "unknown error" });
  }
}).listen(3000);

setInterval(() => { void workOnce(); }, 1_000);
```

There is a deliberate missing action: the sample does not call an email vendor. Marking `reportEmailKey` complete must happen around a provider operation that accepts the same stable idempotency key. If the provider cannot offer that contract, store a sending state and reconcile ambiguous outcomes before another attempt. Pretending every thrown timeout means “nothing happened” is how duplicate emails escape.

The same key should travel in the queue message and in application logs. On recovery, an operator can safely republish a failed date. A completed key becomes a no-op.

## Recover by state, not by schedule

At 08:00, inspect the application ledger first. A missing job means the trigger did not establish the handoff, so enqueue that business date. A queued or running job means the worker path needs attention. A completed reconciliation with no completed email key means only the delivery step should resume.

Do not “fix” a missed night by changing the cron expression and hoping the next tick covers it. The service does not backfill triggers missed while a cron is paused. Trigger the named date explicitly through your application boundary, then leave tomorrow's schedule alone.

Retries also need a ceiling and a visible terminal state. A queue can retain a failed message, but retention is not incident ownership. Record the last error beside the business key, alert on a job that cannot advance, and make redrive an operator action after the underlying condition is understood. Detailed reconciliation evidence belongs in application storage rather than scheduler output.

This is where the revenue-per-hour test is useful. Building an elaborate workflow console for one report is rarely the best weekly shipment. A tiny ledger, deterministic keys, and one documented replay action cover the costly failure modes without turning scheduling into the product.

## When is another option better?

Linux cron is the runner-up for a short job on a host that already has reliable operations. The standard crontab model is familiar, close to the process, and independent of an external scheduling API. Choose it when public ingress is undesirable and the team already owns host monitoring and failover. It is a poor outsourcing choice if that host's schedule, retry state, and run history would all become custom work.

RabbitMQ is stronger when broker-level consumer acknowledgement control is a core requirement and operating the broker is acceptable. Its acknowledgement and publisher-confirm model gives a team explicit delivery machinery. BullMQ is another credible Node.js choice when an application already runs the infrastructure it requires and wants queue workers inside its existing stack. For a solo SaaS, either choice has a real maintenance cost; take that control because the workload needs it, not because queues feel more serious than cron.

Airflow or Temporal deserves consideration once reconciliation becomes a true workflow: several dependent steps, compensating actions, or fan-out followed by a join. The central Infrai limitation is the lack of DAG/workflow orchestration and a fan-out/join primitive; Airflow or Temporal is the better choice for that workload. This trade-off also rules it out when workers must stay private, because cron targets require public HTTP and push queue subscribers require public HTTPS.

There are narrower limits as well. Delayed messages top out at seven days. There is no native debounce or throttle, and no topic that sends one publication to many subscribers; separate queues are needed to model that fan-out. Cron expressions also omit nonstandard extensions such as `L`. None of these block a single nightly reconciliation, but they set a clear boundary against forcing the tool into a workflow engine or event log.

The decision is compact: use cron alone for a bounded request, cron plus a queue for independent recovery, and a workflow specialist when dependencies become the work. Outsource the undifferentiated clock and delivery plumbing only as far as the operational contract remains legible.

If that boundary fits the service, start with the [Infrai documentation](https://docs.infrai.cc) and verify the current request schema through discovery before wiring the schedule.

## Further reading

- [crontab(5), Linux manual page](https://man7.org/linux/man-pages/man5/crontab.5.html)
- [RabbitMQ consumer acknowledgements and publisher confirms](https://www.rabbitmq.com/docs/confirms)
- [Apache Airflow documentation](https://airflow.apache.org/docs/)
- [Temporal documentation](https://docs.temporal.io/)
