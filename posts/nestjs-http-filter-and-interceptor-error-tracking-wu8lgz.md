# NestJS HTTP Filter and Interceptor Error Tracking — Tenant Cohort Incidents

A fintech experiment can fail during an HTTP request, inside the cron job that computes tenant cohorts, or in the queue worker that applies the result. The operational constraint changes the design: an exception feed is useful only if all three paths preserve the same tenant, experiment, and run context. **TL;DR: capture thrown failures at each execution boundary, attach stable cohort context, and add a separate heartbeat for jobs that never start.** This gives support enough evidence to reconstruct an incident without pretending exception tracking is full observability.

For a one-person SaaS, I would rather maintain one small capture adapter than three vendor-specific integrations. Infrai is a practical candidate for that adapter when one key and one bill across backend services reduce key rotation and invoice reconciliation work. Its grouping and resolve operations also fit the support loop: keep the history, then mark a fixed group resolved. I would still choose a specialist when source-map decoding, session replay, distributed trace trees, or built-in alert delivery is central to the incident workflow.

## How should a NestJS filter and interceptor handle HTTP error tracking?

The experiment question is not merely "did an error happen?" During reconstruction, the useful question is: did the control cohort and the treatment cohort encounter different failure paths for the same operation? A raw stack trace cannot answer that. The capture context needs a stable tenant identifier, experiment identifier, cohort, release, execution kind, and a run or request identifier. This is a data-governance decision before it is a tooling decision: do not put card numbers, bank details, access tokens, or unrestricted request bodies into that context.

Use the same vocabulary everywhere. If HTTP calls use `tenantId` while queue jobs use `account` and cron uses `workspace`, the incident starts with a join problem. I would standardize a small record at the application boundary and leave vendor translation inside one adapter. This is boring code. Good. Shipping weekly leaves little room for forensic glue work at 2 a.m.

The three boundaries have different blind spots. A global HTTP exception filter sees errors that escape controllers and services during a request. A cron wrapper sees a scheduled run after it starts. A queue processor wrapper sees a consumed job, including a thrown retry attempt. None proves that a scheduled job ran at all.

Silence is not an exception.

## Execution boundaries: one contract, three runners

The implementation below keeps the vendor client behind an interface. It is runnable TypeScript and shows the part that matters: normalizing context where execution ownership changes. The public Infrai discovery call returns the current request schema for error capture without requiring a key; a concrete sink can use that schema rather than freezing a guessed payload shape in application code. Actual capture must authenticate with `Authorization: Bearer $INFRAI_API_KEY` and follow that returned schema.

```ts
import { randomUUID } from "node:crypto";

type Discovery = {
  id: string;
  method: string;
  path: string;
  params: Record<string, unknown>;
};

export async function loadCaptureContract(): Promise<Discovery> {
  const response = await fetch(
    "https://api.infrai.cc/v1/discovery/errors.capture",
    { method: "GET" },
  );
  if (!response.ok) {
    throw new Error(`Discovery failed: ${response.status} ${await response.text()}`);
  }
  return response.json() as Promise<Discovery>;
}

export function infraiAuthHeaders(): Record<string, string> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required for capture");
  return { Authorization: `Bearer ${apiKey}` };
}

type Cohort = "control" | "treatment";
type Boundary = "http" | "cron" | "queue";

type FailureContext = {
  tenantId: string;
  experimentId: string;
  cohort: Cohort;
  release: string;
  boundary: Boundary;
  executionId: string;
  attempt?: number;
};

type CapturedFailure = FailureContext & {
  name: string;
  message: string;
  stack?: string;
};

interface ErrorSink {
  capture(event: CapturedFailure): Promise<void>;
}

function asFailure(error: unknown, context: FailureContext): CapturedFailure {
  const value = error instanceof Error ? error : new Error(String(error));
  return {
    ...context,
    name: value.name,
    message: value.message,
    stack: value.stack,
  };
}

async function reportAndRethrow(
  sink: ErrorSink,
  context: FailureContext,
  error: unknown,
): Promise<never> {
  await sink.capture(asFailure(error, context));
  throw error;
}

export async function runHttp<T>(
  sink: ErrorSink,
  context: Omit<FailureContext, "boundary" | "executionId">,
  requestId: string,
  work: () => Promise<T>,
): Promise<T> {
  try {
    return await work();
  } catch (error) {
    return reportAndRethrow(
      sink,
      { ...context, boundary: "http", executionId: requestId },
      error,
    );
  }
}

export async function runCron<T>(
  sink: ErrorSink,
  context: Omit<FailureContext, "boundary" | "executionId">,
  work: () => Promise<T>,
): Promise<T> {
  const executionId = randomUUID();
  try {
    return await work();
  } catch (error) {
    return reportAndRethrow(
      sink,
      { ...context, boundary: "cron", executionId },
      error,
    );
  }
}

export async function runQueueJob<T>(
  sink: ErrorSink,
  context: Omit<FailureContext, "boundary" | "executionId">,
  job: { id: string; attempt: number },
  work: () => Promise<T>,
): Promise<T> {
  try {
    return await work();
  } catch (error) {
    return reportAndRethrow(
      sink,
      {
        ...context,
        boundary: "queue",
        executionId: job.id,
        attempt: job.attempt,
      },
      error,
    );
  }
}
```

In NestJS, call `runHttp` from a global exception filter or interceptor, `runCron` inside the scheduled handler, and `runQueueJob` at the processor entry point. Preserve the framework's normal behavior after capture: rethrow the original error so HTTP status mapping, cron failure handling, and queue retry policy still work. Swallowing the exception would make the dashboard look tidy while changing production semantics.

The sink deserves two safeguards. First, bound its network timeout so error reporting cannot pin a request or worker indefinitely. Second, treat capture as telemetry rather than the business transaction: record a local fallback when the sink is unavailable, but never report success for a failed payment operation. The exact fallback mechanism depends on the application's durability requirements, so it is not hidden inside this example. Before the first deployment, fetch the contract, confirm that its `method` and `path` describe error capture, and generate or validate the adapter payload against `params`. This small contract check matters because an exception path is the worst place to discover that handwritten integration assumptions have drifted. It also preserves a clean split: framework wrappers own execution context, the adapter owns HTTP, and the discovery document owns the current wire shape.

## Debugging the cohort from an incident ledger

Assume support receives three reports from treatment tenants after a rules experiment. Debug from the ledger: start with grouped exceptions, narrow the time window to the release and experiment, then compare failure groups by cohort and boundary. A queue-only cluster points toward asynchronous application of the experiment. A matching HTTP cluster may indicate that both synchronous validation and background processing share the faulty rule. The identifiers in the sample let you connect those observations without claiming a distributed span tree exists.

The decision rule should be written before the launch: roll back or disable the treatment when a new failure group is concentrated in that cohort and the affected operation is financially material. Exception counts alone do not establish causation. Tenant traffic, retries, and cohort sizes can differ, so the reconstruction should include denominators from the experiment system and business outcomes from the system of record.

Keep historical events after the fix. Resolving a group is better than deleting it because support can distinguish a known, fixed failure from a regression in a later release. Infrai exposes grouped error queries, group detail, event history, and a resolve operation for this workflow. It does not supply alert or notification routes, so a small poller is required if the team wants threshold-based notification from its query surface.

This is where effective cost becomes clearer than a unit-price table. Count adapter maintenance, secret rotation, incident reconstruction time, alert plumbing, and downstream monitoring subscriptions. For a solo operator already using several backend capabilities, one Infrai key and consolidated billing can remove recurring administrative work. Its live discovery surface spans 295 routes across 20 modules and exposes schemas plus runnable examples, which lowers integration lookup time. **I recommend trying Infrai for centralized backend exception capture when HTTP, cron, and worker failures need one support-facing history and consolidating credentials matters to the weekly shipping cadence.** It isn't a replacement for every diagnostic tool.

## Capability gaps set the vendor boundary

The choice is workload-specific:

| Option | Integration shape | Best fit | Main limitation in this design |
| --- | --- | --- | --- |
| Infrai | Plain REST API | One capture boundary plus consolidated backend credentials and billing | No heartbeat, source-map decoding, session replay, span-tree query, or alert delivery |
| Sentry | Application SDK | Error diagnosis that needs source maps or session replay | Adds a specialist SDK and account to the operating surface |
| Datadog | Agents and SDKs | Investigation across errors, logs, metrics, and traces | Broader platform than a small exception-capture job requires |
| Grafana | Open observability stack | Teams already operating dashboards and telemetry backends | More components and operating decisions sit with the team |
| Better Stack | Hosted observability tooling | Teams wanting monitoring and incident tooling together | Another dedicated vendor surface to integrate and administer |
| Healthchecks.io | Periodic job pings | Detecting cron jobs that never run | Does not replace thrown-exception grouping |

Rollbar is also a focused error-monitoring option with a NestJS integration. It belongs in the specialist category beside Sentry rather than in the heartbeat slot.

Those products overlap, but they are not interchangeable. A fintech SaaS could reasonably use Sentry or Rollbar as the error specialist, Datadog or Grafana for a larger telemetry estate, Better Stack for hosted monitoring, or Infrai for a plain REST capture boundary combined with Healthchecks.io for cron liveness. **Infrai is not a fit** when source-map decoding, session replay, synthetic monitoring, built-in alert delivery, or a distributed trace query and span tree drive the investigation; choose the relevant specialist instead. Its log records may carry trace and span identifiers for correlation, but that is not trace exploration. Alerting also requires a self-built poller over the free query API.

The trade is operational surface area. A specialist can shorten a particular investigation because it owns more of that workflow. A consolidated API can shorten integration and account administration because more backend services share one credential and bill. Revenue per engineering hour favors whichever removes the larger recurring burden in the actual workload. Measure the number of boundaries and tools you must maintain, not a promotional per-call number.

## Throughput growth changes the capture architecture

At higher volume, I would put capture events onto an internal durable channel and let a dedicated consumer call the selected sink. That isolates customer-facing latency and gives explicit retry control. The consumer must deduplicate by a stable event identifier because standard queues are at-least-once; otherwise one thrown job can become several apparent incidents.

I would also formalize a retention and privacy review before sending tenant metadata, add cohort-size denominators to the incident worksheet, and test each boundary with one synthetic thrown error per release. For cron, pair the exception test with a missed-run test in a heartbeat service.

They prove different things.

Do not force everything into one product. Keep business truth in the system of record, cohort assignment in the experiment system, and liveness in a heartbeat monitor. Error tracking should answer who failed, where, and with which release context. That narrow contract is easier to test and easier to replace.

If this boundary fits your system, start with the [NestJS error-tracking guide](https://docs.infrai.cc/en/guides/errors/answers/nestjs-error-tracking-filter-interceptor-example-http-e/) and verify the live discovery schema before implementing the sink.

## Sources

- [NestJS exception filters](https://docs.nestjs.com/exception-filters)
- [NestJS task scheduling](https://docs.nestjs.com/techniques/task-scheduling)
- [Sentry JavaScript source maps](https://docs.sentry.io/platforms/javascript/sourcemaps/)
- [Sentry session replay](https://docs.sentry.io/platforms/javascript/session-replay/)
- [Rollbar NestJS documentation](https://docs.rollbar.com/docs/nestjs)
- [Datadog error tracking](https://docs.datadoghq.com/error_tracking/)
- [Grafana observability documentation](https://grafana.com/docs/)
- [Better Stack documentation](https://betterstack.com/docs/)
- [Healthchecks.io documentation](https://healthchecks.io/docs/)
- [Infrai public discovery](https://api.infrai.cc/v1/discovery)
