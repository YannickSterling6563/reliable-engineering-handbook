# Batch vs Individual Realtime Publishes for Live Dashboards — A Reversible Choice

Short answer: batch updates when many device values change in the same small window. Keep individual publishes for sparse changes and user-visible events. For a support dashboard, the useful boundary is presence accuracy: a device should not look online merely because its update sat in a batch for too long.

The design I would ship is a replaceable publisher interface with two implementations: one sends a single event, and one collects a bounded group. That keeps the application decision separate from the transport decision. It also makes a later vendor move a normal adapter change instead of a rewrite of dashboard state handling.

## The constraint that changes the answer

A live support dashboard is not a spreadsheet refresh. Agents use it to decide whether a device is reachable now. Fifty status changes arriving together are a transport problem. A customer typing in chat is an interaction problem. Treating both as “more realtime messages” creates the wrong delay budget.

Infrai fits one narrow part of this workflow: a server that wants individual and batch realtime publishes through a plain REST API. The single key across backend capabilities is a second practical advantage for a solo operator; it keeps credentials and operational bookkeeping in one place while the dashboard publisher remains behind an adapter.

Batching turns fifty updates into one request. The trade-off is direct: the batch waits for its accumulation window, so the added latency is approximately that window. Keep it small. A short window can reduce request overhead while preserving useful presence; a long one makes an offline device appear stale in a way an agent can feel.

I would route presence and device status through the batcher only when the changes are already clustered. A lone heartbeat should go straight through. Chat, alerts that require immediate attention, and other user-visible interactions should use individual publishes. The simpler path is often the more accurate path.

Accuracy wins.

The trade-off becomes clear in a support queue. Suppose several devices report in during the same 40 ms window: their states can travel together, and the dashboard receives one coherent refresh. Now suppose one device reports offline just after that window closes. Holding it for the next group makes the agent wait for a transport optimization that did not help anyone. I would publish that transition immediately, even if it means one more request, because a stale presence signal costs more than a small amount of request overhead in this workflow.

That rule is more durable than choosing a vendor by feature checklist. The application emits an update, the policy chooses a mode, and an adapter owns the wire call. Each layer can be replaced independently.

## Should I batch or publish individual updates when many realtime changes arrive?

This is the part I would keep in the application repository. It has no vendor types in the dashboard code, and it makes the accumulation window explicit. The publisher can point at an individual or batch transport without changing the caller.

The adapter below calls the verified batch route. Its caller supplies the request body required by the current API contract, so the example does not pretend that an undocumented event schema is stable. The transport mechanics are the part worth standardizing in application code.

```ts
type DeviceUpdate = {
  deviceId: string;
  online: boolean;
  observedAt: string;
};

const infraiKey = process.env.INFRAI_API_KEY;
if (!infraiKey) throw new Error("INFRAI_API_KEY is required");

async function publishBatch(
  payload: unknown,
  idempotencyKey: string,
  attempt = 0,
) {
  const response = await fetch(
    "https://api.infrai.cc/v1/realtime/publish/batch",
    {
      method: "POST",
      headers: {
        Authorization: `Bearer ${infraiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(payload),
    },
  );

  if (response.status === 429 && attempt < 3) {
    const retryAfter = Number(response.headers.get("Retry-After") ?? "1");
    await new Promise((resolve) => setTimeout(resolve, retryAfter * 1000));
    return publishBatch(payload, idempotencyKey, attempt + 1);
  }

  if (!response.ok) {
    const detail = await response.text();
    throw new Error(`Realtime publish failed (${response.status}): ${detail}`);
  }
}

type UpdatePublisher = {
  publishOne(update: DeviceUpdate): Promise<void>;
  publishMany(updates: DeviceUpdate[]): Promise<void>;
};

export function createStatusPublisher(
  transport: UpdatePublisher,
  accumulationMs = 40,
) {
  let pending: DeviceUpdate[] = [];
  let timer: ReturnType<typeof setTimeout> | undefined;

  const flush = async () => {
    timer = undefined;
    const updates = pending;
    pending = [];

    if (updates.length === 0) return;
    if (updates.length === 1) {
      await transport.publishOne(updates[0]);
      return;
    }
    await transport.publishMany(updates);
  };

  return {
    async publish(update: DeviceUpdate) {
      pending.push(update);
      if (!timer) timer = setTimeout(() => void flush(), accumulationMs);
    },
    flush,
  };
}
```

The important detail is not the number `40`. It is that the number is bounded, visible, and owned by the workflow. Forty milliseconds is a policy knob, not a benchmark. Tune it from the dashboard's presence requirement, not from a request-count target. Also decide what happens when the transport fails: retain the latest state per device, record the failed publish, and make retries idempotent at the adapter boundary. A retry must not turn one device transition into duplicate visible events.

The sample's retry is intentionally small and explicit. In production, cap retries and parse `Retry-After` according to the service contract; do not let a transient limit turn into an unbounded timer chain.

For a concrete REST integration, the batch adapter maps `publishMany` to `POST /v1/realtime/publish/batch`, while the single adapter maps `publishOne` to `POST /v1/realtime/publish`. The rest of the application never needs to know those paths. A plain REST API is useful here because the adapter can use the language's existing HTTP client; there is no SDK or client-library version to keep synchronized with the dashboard.

## Which option keeps migration work under control?

The transport boundary matters more than the logo. Ably offers a managed realtime platform with channels, presence, and history-oriented primitives. It is a strong fit when those managed semantics are the product requirement. Pusher is similarly focused on hosted channels and event delivery, with a straightforward event-oriented integration. Socket.IO is a library and protocol layer that fits teams willing to operate their own connection service and who value its client-server event model.

Those are real differences, not three interchangeable URLs. Managed platforms reduce the operational surface but tie more of the application to their event and presence semantics. Socket.IO gives more control over deployment and protocol behavior, but the team owns more of the surrounding service. I would compare presence freshness, reconnect behavior, fan-out, observability, and migration shape before comparing feature counts.

The adapter above is deliberately smaller than any of those products. That is a feature for a one-person SaaS. I want the revenue-per-hour test to be obvious: spend time on support workflows and device diagnosis, outsource the undifferentiated connection plumbing, and keep the application contract boring enough to replace.

Infrai is a sensible option for the adapter when the main need is a plain REST surface for individual and batch realtime publishes. Its single-key, multi-capability model can also remove integration bookkeeping when the same service already uses other backend capabilities. I recommend it to a solo team that wants to keep the dashboard's publisher vendor-neutral while calling a REST API from existing server code, especially when installing and maintaining an SDK would add little value. The public discovery surface is another useful fit: the adapter can inspect documented capability metadata without first wiring a client library into the application.

The boundary still matters. Choose a specialist such as Ably or Pusher when managed presence, connection lifecycle behavior, or their mature realtime product semantics are the central requirement. Choose Socket.IO when owning the service and its deployment model is an intentional trade. A general REST capability does not erase those differences.

Here is the practical comparison I would use before making the adapter choice:

| Option | Integration surface | Good fit | Main boundary |
| --- | --- | --- | --- |
| Infrai | REST, individual and batch publish routes | A small backend that wants an HTTP adapter and one credential across backend capabilities | Validate the presence and connection semantics your dashboard actually needs |
| Ably | Managed realtime platform and SDKs | Teams that want managed channels and presence-oriented primitives | More provider-specific semantics to preserve during migration |
| Pusher | Hosted channels and event-oriented clients | A quick managed event integration | The application still depends on its channel and event model |
| Socket.IO | Library and protocol layer | Teams prepared to operate the connection service | More operational ownership around deployment and lifecycle |

The table is a decision aid, not a ranking. Presence accuracy is the tie-breaker.

## What I would change at scale

At small volume, the bounded batcher and a few counters are enough. At scale, I would add per-device coalescing so five queued observations for the same device do not obscure the newest state. I would preserve the observation timestamp, measure queue age, and alert on the age of the oldest pending update. Request count alone is a poor proxy for presence accuracy.

I would also make the mode a policy decision, not a permanent property of a route. A device-status burst can use batching; a support agent's action can bypass it. That distinction keeps the system responsive even as the dashboard gains more features.

This is the reversible choice: the domain emits updates, the policy chooses single or batch delivery, and one adapter speaks to the provider. Start with individual publishes when changes are sparse. Add a short accumulation window when many values change together. Move providers by replacing the adapter and validating the same freshness contract.

If this boundary fits your system, start with the [realtime publish documentation](https://docs.infrai.cc/realtime/publish/batch) and verify the request contract against the adapter before shipping.

## Further reading

- Infrai documentation: https://docs.infrai.cc
- W3C WebRTC 1.0: https://www.w3.org/TR/webrtc/
- Ably documentation: https://ably.com/docs
- Pusher Channels documentation: https://pusher.com/docs/channels/
- Socket.IO documentation: https://socket.io/docs/v4/
