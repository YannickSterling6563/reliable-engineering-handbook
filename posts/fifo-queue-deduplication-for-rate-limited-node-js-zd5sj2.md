# FIFO Queue Deduplication for Rate-Limited Node.js Jobs with Idempotency Keys

Short answer: use a five-minute admission deduplication window to collapse repeated enqueue requests, but use a durable idempotency record at execution time as the alternative to an exactly-once claim. Keep the gaming renewal reminder's business deadline in the job, release due work into a FIFO queue, and acknowledge a delivery only after the reminder outcome is durably recorded.

This splits three concerns that are easy to muddle: delay until a business deadline, rate-limit the downstream call, and prevent the same renewal reminder from taking effect twice. A single queue flag cannot safely do all three. The practical target is at-least-once delivery with an idempotent side effect, plus enough durable state to recover after a process exits at an awkward moment.

That's the choice I would ship for a weekly release cadence. The queue is plumbing; renewal policy is the product. Time spent pretending a distributed workflow is exactly once has poor revenue per hour.

## How should a FIFO queue deduplicate rate-limited Node.js jobs?

Start with the business identity, not a hash of the whole payload. For this example, a useful identity is the game account, subscription, reminder kind, and renewal deadline. Two HTTP requests that differ only in tracing metadata should still describe one reminder. A corrected deadline, however, is a new scheduling decision and needs a new key or an explicit replacement operation.

The five-minute window belongs at admission. An atomic insert-if-absent on the deduplication key rejects a burst of repeats for 300 seconds. It reduces queue noise, but it does not prove that a reminder executes once: a retry can arrive after the window, a worker can lose its lease, and an acknowledgement can disappear after the effect has already committed. Deduplication is traffic hygiene. Idempotency protects the business action.

Keep a longer-lived execution record keyed by the same business identity. Its states can be `processing` with a lease and `completed` with a stored outcome. A worker that sees `completed` acknowledges without sending again. A worker that sees an active lease leaves the delivery available for a later attempt. When an expired lease is reacquired, processing can continue. The retention period must cover the longest interval in which the system might redeliver that reminder; that period is a product and operations decision, so I won't invent a universal number.

FIFO needs one qualification. Strict global FIFO means a rate-limited or repeatedly failing head job blocks every later job. For renewal reminders, ordering usually matters within one subscription or account, not across the entire game. Partitioning by that key preserves the useful ordering rule while allowing unrelated reminders to move. If the business truly requires one global order, accept the head-of-line delay and make it visible in the service-level objective.

## The constraint that changes the design

The deadline is a business time, not a sleep duration. Store an absolute `dueAt` timestamp in UTC, compare it with a trusted server-side clock, and leave the record durable while it waits. A Node.js timer is fine for waking a loop; it is not the source of truth. Deployments, crashes, and long pauses must not erase scheduled work.

The dangerous boundary is between changing application data and publishing a queue message. If those are separate commits, the database can contain a renewal deadline while the queue never receives its reminder, or the queue can receive a reminder for a transaction that later rolls back. The Transactional Outbox pattern addresses that gap by writing the business change and an outbox record in the same database transaction, then letting a relay publish the outbox record. The relay can publish more than once, which is why the consumer still needs idempotency.

This matters more than clever queue selection. A solo operator can recover from a slow backlog with a clear ledger. Recovering a reminder that never existed in any durable scan is guesswork.

Rate limiting sits after eligibility and before the external effect. A shared token bucket or equivalent limiter must cover all workers, not one counter per Node.js process. When no token is available, reschedule the delivery for the next eligible instant without marking the business action complete. Preserve the original enqueue sequence as a tie-breaker for jobs with the same due time.

Short version: persist first.

## Smallest working TypeScript shape

The following code is intentionally an orchestration boundary, not a vendor SDK tutorial. The queue, ledger, and limiter interfaces make the correctness decisions visible. Their implementations need atomic operations in durable shared storage.

```ts
type ReminderJob = {
  id: string;
  accountId: string;
  subscriptionId: string;
  renewalDeadline: string;
  dueAt: string;
  sequence: number;
};

type Delivery = {
  receipt: string;
  job: ReminderJob;
};

type LeaseResult = "acquired" | "busy" | "completed";

interface FifoQueue {
  reserveDue(now: Date): Promise<Delivery | null>;
  acknowledge(receipt: string): Promise<void>;
  retryAt(receipt: string, when: Date): Promise<void>;
  deadLetter(receipt: string, reason: string): Promise<void>;
}

interface IdempotencyLedger {
  acquire(key: string, leaseUntil: Date): Promise<LeaseResult>;
  complete(key: string, outcome: { reminderId: string }): Promise<void>;
  abandon(key: string): Promise<void>;
}

interface SharedRateLimiter {
  take(key: string, now: Date): Promise<{ allowed: true } | { allowed: false; retryAt: Date }>;
}

interface ReminderSender {
  send(job: ReminderJob, idempotencyKey: string): Promise<{ reminderId: string }>;
}

const keyFor = (job: ReminderJob): string =>
  [job.accountId, job.subscriptionId, "renewal", job.renewalDeadline].join(":");

async function processOne(
  queue: FifoQueue,
  ledger: IdempotencyLedger,
  limiter: SharedRateLimiter,
  sender: ReminderSender,
  now = new Date(),
): Promise<void> {
  const delivery = await queue.reserveDue(now);
  if (!delivery) return;

  const { job, receipt } = delivery;
  const key = keyFor(job);
  const leaseUntil = new Date(now.getTime() + 60_000);
  const lease = await ledger.acquire(key, leaseUntil);

  if (lease === "completed") {
    await queue.acknowledge(receipt);
    return;
  }

  if (lease === "busy") {
    await queue.retryAt(receipt, leaseUntil);
    return;
  }

  const permit = await limiter.take(job.accountId, now);
  if (!permit.allowed) {
    await ledger.abandon(key);
    await queue.retryAt(receipt, permit.retryAt);
    return;
  }

  try {
    const outcome = await sender.send(job, key);
    await ledger.complete(key, outcome);
    await queue.acknowledge(receipt);
  } catch (error) {
    await ledger.abandon(key);
    const reason = error instanceof Error ? error.message : "unknown send failure";
    await queue.deadLetter(receipt, reason);
  }
}
```

There is a deliberate order in the success path: send with the idempotency key, durably mark the key complete, then acknowledge. If the worker stops after `complete` but before `acknowledge`, redelivery finds `completed` and exits cleanly. If it stops during `send`, the expired processing lease permits recovery; the sender must honor the same idempotency key, or the effect and ledger need to share a transactional boundary. Without one of those guarantees, no queue-side setting can rule out a duplicate effect.

The example dead-letters an unknown send failure to keep its retry policy honest. A real implementation should classify errors: a rate-limit response such as 429 gets a bounded delayed retry that respects the server's retry guidance, while a permanent validation error goes to review. Don't spin immediately. Also avoid copying raw error text into customer-visible logs; attach a stable reason code and trace identifier in the queue metadata.

Admission is a separate, small transaction. Compute `keyFor(job)`, atomically claim that key with a 300-second expiry, and enqueue only when the claim succeeds. If enqueueing and claiming cannot share a transaction, put the request into an outbox first. This is the place where a compact implementation often grows one extra table, and that table earns its keep during recovery.

## Operational recovery before optimization

Recovery begins with observable states. Track the oldest due job, due backlog by partition, admission duplicates, active leases, expired leases, retry age, completed idempotency hits, and dead-letter count. The oldest due age is especially useful because queue depth alone can look healthy while one account partition is stuck behind a poisoned job.

Crashes happen.

Dead letters are evidence, not a disposal system. RabbitMQ documents dead-letter exchanges as a way to republish messages after conditions such as rejection or expiration. Whatever queue implementation is used, the operational loop still needs an owner: inspect a stable failure reason, fix or classify the input, and replay through the normal idempotent path. Replaying by directly calling the sender bypasses the ledger and creates a second production workflow to debug. I would test the crash boundaries explicitly: stop a worker after reservation, after lease acquisition, after the sender accepts the key, after ledger completion, and before acknowledgement; then advance the clock beyond the lease and verify the final state. Include two admissions 299 seconds apart and another after 301 seconds. Those numbers don't prove every storage implementation is correct, but they expose off-by-one window logic and clarify whether expiry is inclusive. I'm not sure how much partition skew a particular game will produce without its account traffic distribution. Measure it. A single celebrity account, guild, or regional batch can dominate one key, so alert on per-partition age and keep repartitioning possible without changing the business idempotency key. Finally, use a reconciliation job as the safety net. It should scan durable renewal deadlines and completed reminder records, find overdue work with no completion, and feed those identities back through the same scheduling path. Run it slowly enough to respect the normal rate limit. This is intentionally boring work — which is good — because operational recovery should depend on durable comparisons, not memory of which worker was alive.

## What I would change at scale, and where this is unsuitable

At higher volume, I would separate the deadline index from ready queues, shard ready work by the narrowest ordering key the business accepts, and lease batches rather than single records. I would keep the same idempotency identity through those changes. That lets the storage and queue topology evolve without changing the meaning of “this renewal reminder already happened.”

The catch is that a five-minute deduplication window is not suitable when duplicates can be generated hours apart and must be rejected before they consume queue capacity. Use a durable uniqueness constraint for that admission rule. Strict FIFO is also the wrong choice when throughput matters more than cross-job order; use partitioned ordering or an unordered work queue instead. And if a reminder side effect cannot accept an idempotency key and cannot participate in a transaction with the ledger, the design provides duplicate reduction rather than a hard exactly-once effect. The business must either tolerate that boundary or choose an effect system with a stronger contract.

There is some operational cost: an outbox, an idempotency ledger, expiry cleanup, reconciliation, and dead-letter review. For a tiny workload with a process that can safely rebuild every reminder from one database table, a periodic database scan may be the better system. It has fewer moving parts. Outsource undifferentiated queue machinery only after the recovery contract is written down; a managed service cannot decide the identity of a renewal reminder or the correct replay policy.

The decision rule is simple. Choose the least elaborate design that can reconstruct overdue work, rate-limit across all workers, and demonstrate an idempotent outcome after every crash boundary. Ship that, test recovery in each weekly release, and add queue machinery only when measured backlog or contention demands it.

## References

- Transactional Outbox pattern: https://microservices.io/patterns/data/transactional-outbox.html
- RabbitMQ dead letter exchanges documentation: https://www.rabbitmq.com/docs/dlx

## Further reading

- https://microservices.io/patterns/data/transactional-outbox.html
- https://www.rabbitmq.com/docs/dlx
