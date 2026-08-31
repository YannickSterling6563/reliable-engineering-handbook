# A Coalescing Window for Password Reset Email Retry Safety After Timeout

| Design | What a repeated request does | What a send timeout means | Operating cost |
| --- | --- | --- | --- |
| Synchronous send | Creates and sends again | Usually retried as failure | Low until recovery gets noisy |
| Queue after database commit | Publishes another job unless separately guarded | Worker policy decides | Moderate, with a handoff gap |
| Transactional outbox plus coalescing window | Reuses one live intent and one event | Unknown outcome; retry same event | Moderate and explicit |
| No retry | May create a new intent later | Drops the attempt | Low, but risks missing mail |

Short answer: coalesce repeated password reset requests into one live intent, commit one email event with it, and retry that same event after a timeout. Don't promise exactly-once email delivery. Preserve one reset link and one stable delivery identity instead.

For a one-person SaaS, the transactional outbox is the practical default. It keeps account recovery state in the database I already operate and moves slow delivery work off the request path. The extra table and worker must earn their keep, though. The point is fewer ambiguous states, not architectural decoration.

## How should a password reset email retry after a timeout avoid duplicate sends?

A timeout says the caller stopped waiting. It does not say whether the mail transport accepted the submission. Those two outcomes can look identical from the application side: no response arrived before the deadline. Replaying the entire password reset handler turns that uncertainty into a second token, a second event, and potentially multiple emails.

Treat the user action and the delivery attempt as separate state machines. The user action is a reset intent with a defined lifetime. The delivery attempt is an outbox event tied to that intent. A repeated form submission inside the coalescing window finds the live intent instead of creating another one; a worker timeout changes attempt metadata instead of event identity.

Small distinction. Large effect.

This is not literal exactly-once transport. A worker can submit a message, lose the acknowledgment, and restart before recording acceptance. If the receiver documents idempotent submission, the worker can reuse the event ID as its key. If that contract is absent, at-least-once submission remains possible. The application can still stop duplicate requests from producing competing reset links, which is the boundary it actually controls.

DKIM solves a different problem. RFC 6376 defines a domain-level signature that lets a verifier associate a message with a signing domain. It does not coordinate a database transaction with an email submission, and it does not deduplicate retries. Configure authentication for deliverability and trust, but keep retry identity in application state.

SMS follows the same application rule even though the channel has different interoperability and compliance concerns. CTIA publishes messaging guidance for that ecosystem. A delivery receipt or channel status should remain evidence about an attempt, not become the source of truth for the reset intent.

## The two decision criteria that matter

The first criterion is atomicity. Creating the reset intent and its outbox event in one database transaction removes the gap between “credential state committed” and “delivery work recorded.” If the transaction commits, the worker has durable work to discover. If it rolls back, neither record exists. Publishing to a queue only after the commit can lose the handoff when the process stops between those operations; publishing before the commit can expose work whose reset intent never became durable.

The second criterion is stable identity across every replay. Put a uniqueness constraint behind the active reset scope, and derive one immutable event ID from the chosen intent. Concurrency matters here: two browser tabs can submit before either request sees the other's insert. A read-then-insert check without a database constraint is only a hopeful optimization. The constraint makes the policy real, while the application handles the conflict by loading the winner.

I use a revenue-per-hour test for the rest. A lease, retry schedule, and three useful metrics are worth maintaining because they replace mailbox archaeology during an account-recovery complaint. A sprawling orchestration layer isn't. Ship the smallest durable state machine, instrument it, and return to product work.

The states should describe evidence rather than guesses: `pending`, `leased`, `accepted`, and `review`. Track `attemptCount`, `nextAttemptAt`, and lease expiry separately. A timeout returns the event to `pending` with the same ID. An explicit permanent rejection goes to `review`; it should not spin forever. I'm not sure a transport deduplicates concurrent submissions unless its published contract says so. Documentation plus a concurrency test resolves that question; a dashboard showing one message does not.

## A focused TypeScript pattern

The example leaves token generation and hashing behind narrow interfaces because those details belong to the authentication layer. The important part is the transaction shape and the reuse of `event.id` on every send attempt.

```ts
type ResetIntent = {
  id: string;
  userId: string;
  tokenHash: string;
  expiresAt: Date;
};

type EmailEvent = {
  id: string;
  intentId: string;
  recipient: string;
  status: "pending" | "leased" | "accepted" | "review";
  attemptCount: number;
  nextAttemptAt: Date;
};

interface ResetTransaction {
  findLiveIntent(userId: string, now: Date): Promise<ResetIntent | null>;
  insertIntent(input: Omit<ResetIntent, "id">): Promise<ResetIntent>;
  insertEventOnce(event: EmailEvent): Promise<void>;
}

interface ResetStore {
  transaction<T>(work: (tx: ResetTransaction) => Promise<T>): Promise<T>;
  leasePending(now: Date, limit: number): Promise<EmailEvent[]>;
  markAccepted(eventId: string): Promise<void>;
  retryLater(eventId: string, nextAttemptAt: Date): Promise<void>;
  markForReview(eventId: string): Promise<void>;
}

type Submission =
  | { outcome: "accepted" }
  | { outcome: "rejected"; permanent: boolean };

interface EmailTransport {
  submit(input: {
    recipient: string;
    template: "password-reset";
    resetIntentId: string;
    idempotencyKey: string;
  }): Promise<Submission>;
}

async function recordResetRequest(
  store: ResetStore,
  userId: string,
  recipient: string,
  tokenHash: string,
  now: Date,
): Promise<void> {
  await store.transaction(async (tx) => {
    const live = await tx.findLiveIntent(userId, now);
    const intent = live ?? await tx.insertIntent({
      userId,
      tokenHash,
      expiresAt: new Date(now.getTime() + 30 * 60_000),
    });

    await tx.insertEventOnce({
      id: `reset-email:${intent.id}`,
      intentId: intent.id,
      recipient,
      status: "pending",
      attemptCount: 0,
      nextAttemptAt: now,
    });
  });
}

async function dispatchResetEmail(
  store: ResetStore,
  transport: EmailTransport,
  event: EmailEvent,
  now: Date,
): Promise<void> {
  try {
    const result = await transport.submit({
      recipient: event.recipient,
      template: "password-reset",
      resetIntentId: event.intentId,
      idempotencyKey: event.id,
    });

    if (result.outcome === "accepted") {
      await store.markAccepted(event.id);
    } else if (result.permanent) {
      await store.markForReview(event.id);
    } else {
      await store.retryLater(event.id, new Date(now.getTime() + 60_000));
    }
  } catch {
    // No acknowledgment is an unknown outcome, so preserve event identity.
    await store.retryLater(event.id, new Date(now.getTime() + 60_000));
  }
}
```

The database still needs a uniqueness rule for the live-intent scope and for `EmailEvent.id`. The worker needs expiring leases so abandoned work becomes eligible again. Use bounded exponential backoff with jitter rather than a fixed one-minute delay from the compact example. Never log raw reset tokens, and avoid retaining recipient data in attempt logs when an event ID is enough. Test the ugly boundaries before release: two concurrent reset requests, a process stop after submission but before `markAccepted`, an expired worker lease, an explicit permanent rejection, and a second request after the coalescing window. Assertions should check the number of intents and stable event IDs, not merely how many times a mock function ran. Also verify that the public response does not reveal whether the account exists. Observability can stay lean: alert on oldest pending-event age and a rising review count, then graph attempts per event while separating timeouts from explicit rejections because they carry different evidence. Together, those tests and signals answer the operator's real questions without building a monitoring project around one email: are users waiting, are retries concentrating, did concurrency create another live intent, and does a human need to inspect a permanent outcome?

Keep it boring.

## When is the runner-up a better choice?

The catch is the outbox's operational surface: schema retention, polling, leases, retries, and alerts. It is not suitable when the application has no transactional database, or when the hosting platform already provides a documented atomic handoff between committed state and queued work. In that case, stick with the native handoff, but preserve the same intent ID and replay rules.

A synchronous send is acceptable for a disposable internal prototype where a missed or repeated message has no security consequence. Don't carry that shortcut into public account recovery. The first ambiguous acknowledgment spreads evidence across request logs, worker state, and transport records.

If the email transport has no documented idempotency contract, an application-owned delivery gateway with a deduplication ledger is the stronger runner-up at high volume or across several channels. It costs more engineering time. For a solo SaaS, I would add it only when measured duplicate submissions justify another service; otherwise, the outbox still protects token and intent identity while accepting the channel's at-least-once boundary.

The release decision is plain: coalesce one live intent, commit one durable event, and replay only that identity. This does not make the network exactly once. It makes uncertainty visible and keeps the security-sensitive part under application control.

## References

- RFC 6376, DomainKeys Identified Mail (DKIM): https://datatracker.ietf.org/doc/html/rfc6376
- CTIA, Messaging Interoperability and Compliance Best Practices: https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
