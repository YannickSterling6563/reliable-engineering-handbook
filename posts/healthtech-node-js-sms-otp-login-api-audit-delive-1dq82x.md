# Healthtech Node.js SMS OTP Login API: Audit Delivery Before Retry

**Short answer:** for a healthtech Node.js SMS OTP login API, create the challenge and audit record before sending, keep resend cooldown and rate limit decisions on the server, and issue a session only after one unexpired code is verified exactly once.

That is the boundary I would ship as a one-person SaaS founder. The message provider is a delivery dependency. It is not the authority that decides whether a user may log in. My scarce resource is revenue per hour, so I outsource the undifferentiated transport and keep the small, consequential state machine in my own application.

The scenario matters. A healthtech compliance notice needs an auditable delivery record, while an OTP login needs a safe verification record. Those are related events, not the same event. “The provider accepted the request” does not mean “the notice reached the phone,” and neither statement means “the account got a session.”

## Why does a Node.js SMS OTP login need an audit record?

Because a retry can otherwise change the meaning of the login. A browser may retry after a timeout even though the first send succeeded. A second tab may press Resend. A worker may replay a queue item. If the application stores only the latest code and a client-side countdown, it cannot explain which challenge was active, which destination was used, or why a session was created.

I use a server-side challenge with these fields:

| Field | Purpose |
| --- | --- |
| `challengeId` | Stable correlation key for logs and support queries |
| `loginId` | Binds the challenge to one account or login transaction |
| `destinationHash` | Avoids putting a phone number in routine logs |
| `codeDigest` | Stores a verifier, never the plaintext OTP |
| `expiresAt` | Makes old codes invalid without a cleanup race |
| `resendAt` | Enforces the cooldown on the server |
| `verifyAttempts` | Limits guesses for this challenge |
| `status` | Distinguishes pending, verified, expired, and closed |
| `createdAt` and `verifiedAt` | Supplies the audit timeline |

The compliance notice gets its own delivery event linked by `challengeId` or another request correlation ID. Record intent, provider acceptance, later delivery evidence, and the final business action separately. A single `sent: true` boolean is too vague for an audit and too tempting to misuse as an authentication result.

SMS also has a security limit that product copy should not hide. NIST describes out-of-band authentication over the public switched telephone network as restricted, with risks such as interception and reassignment. SMS can be a practical recovery or second-factor channel for some users, but it is not a reason to pretend that every account has phishing-resistant authentication. For higher-risk actions, offer a stronger authenticator and make the risk decision explicit.

## The smallest working state machine

The send path has one job: reserve a challenge and request delivery. It must not mint a session. The verify path has a different job: compare the submitted code with the live challenge, consume that challenge atomically, and then create a session.

Here is the application-side shape. `ChallengeStore` and `SmsTransport` are interfaces so the policy is testable without sending a real text message.

```ts
type ChallengeStatus = "pending" | "verified" | "expired" | "closed";

type Challenge = {
  id: string;
  loginId: string;
  destination: string;
  codeDigest: string;
  expiresAt: number;
  resendAt: number;
  verifyAttempts: number;
  status: ChallengeStatus;
};

type ChallengeStore = {
  createIfAbsent: (challenge: Challenge) => Promise<boolean>;
  find: (id: string) => Promise<Challenge | null>;
  consumeForVerification: (id: string) => Promise<boolean>;
  incrementFailedAttempt: (id: string) => Promise<number>;
};

type SmsTransport = {
  send: (input: {
    destination: string;
    message: string;
    requestId: string;
  }) => Promise<{ accepted: boolean; providerMessageId?: string }>;
};

const OTP_TTL_MS = 5 * 60 * 1_000;
const RESEND_COOLDOWN_MS = 60 * 1_000;
const MAX_VERIFY_ATTEMPTS = 5;

async function startLogin(
  store: ChallengeStore,
  sms: SmsTransport,
  loginId: string,
  destination: string,
  codeDigest: string,
  code: string,
  now = Date.now(),
): Promise<{ challengeId: string; accepted: boolean }> {
  const challengeId = crypto.randomUUID();
  const challenge: Challenge = {
    id: challengeId,
    loginId,
    destination,
    codeDigest,
    expiresAt: now + OTP_TTL_MS,
    resendAt: now + RESEND_COOLDOWN_MS,
    verifyAttempts: 0,
    status: "pending",
  };

  const reserved = await store.createIfAbsent(challenge);
  if (!reserved) throw new Error("challenge reservation was not unique");

  const result = await sms.send({
    destination,
    message: `Your sign-in code is ${code}. It expires in 5 minutes.`,
    requestId: challengeId,
  });

  return { challengeId, accepted: result.accepted };
}

async function verifyLogin(
  store: ChallengeStore,
  challengeId: string,
  submittedCode: string,
  matchesDigest: (code: string, digest: string) => Promise<boolean>,
  issueSession: (loginId: string) => Promise<string>,
  now = Date.now(),
): Promise<string> {
  const challenge = await store.find(challengeId);
  if (!challenge || challenge.status !== "pending" || challenge.expiresAt <= now) {
    throw new Error("challenge is not active");
  }

  const matches = await matchesDigest(submittedCode, challenge.codeDigest);
  if (!matches) {
    const attempts = await store.incrementFailedAttempt(challengeId);
    if (attempts >= MAX_VERIFY_ATTEMPTS) {
      // The store closes the challenge in the same transaction as this counter.
      throw new Error("challenge is closed");
    }
    throw new Error("invalid code");
  }

  const consumed = await store.consumeForVerification(challengeId);
  if (!consumed) throw new Error("challenge was already consumed");
  return issueSession(challenge.loginId);
}
```

The important line is not the OTP length. It is `consumeForVerification`. That operation must be conditional on `status = pending` and happen in the same transaction that changes the status. Two simultaneous verification requests should produce one session at most. The session creation itself must also be idempotent for the login transaction, because a client can retry after receiving a response late.

The example leaves the database adapter out on purpose. A relational row with a conditional update, or a key-value store with an atomic compare-and-set and expiry, can implement the contract. I would test the contract with parallel requests, clock boundaries, duplicate request IDs, and a process restart between reservation and transport. A green happy-path test is not evidence that the audit trail is correct.

## How should a Node.js login API separate delivery, verification, and rate limits?

Treat Resend as a new delivery attempt against the same login policy, not as permission to replace state casually. The server checks the challenge status, `resendAt`, destination limits, account limits, and source limits. It then advances the cooldown in an atomic operation before asking the transport to send. The UI can display the remaining seconds, but it cannot grant the next send.

I normally separate four budgets:

- per challenge: failed verification attempts and expiry;
- per destination: sends in a rolling window, to limit harassment and spend;
- per account or login identifier: repeated attempts across phone changes;
- per source signal: IP, device, and regional risk controls.

The exact numbers are policy, not a universal constant. Your mileage may vary by country, user population, and the damage caused by account takeover. I am not sure a single five-minute rule will fit every healthtech product, so I would start with a documented policy, log each allow or deny decision, and adjust it from abuse and support data.

Do not reset the failed-attempt budget just because a resend creates a new message. An attacker should not get an infinite guessing budget by clicking Resend. A resend should invalidate the previous code when the new challenge is committed; record that transition and test it. Keep one active challenge per login transaction and one current code, with the old digest made unusable in the same write.

Handle transport responses with the same discipline. A timeout is unknown outcome, not automatic permission to send again. A 429 is a rate-limit signal, not a verification failure. A 5xx from a transport should produce a user-safe status and an operational event, while the audit record retains the request ID and outcome classification. Never expose provider response text to the login page.

The compliance notice follows a separate rule: keep the intended notice and its delivery evidence durable even if the user retries the login. DMARC helps receivers evaluate domain alignment and policy for email, but it does not turn SMS delivery into proof of identity; that distinction belongs in the data model, too.

## A provider boundary that does not own the policy

The transport adapter should accept a destination, message, and request ID, then return a small normalized result. It should not know whether the request is a login, a compliance notice, or a support resend. That keeps transport changes and test doubles cheap, which is the kind of outsourcing that improves revenue per hour rather than adding another dashboard to maintain.

Keep it boring.

```ts
type TransportResult =
  | { kind: "accepted"; externalId?: string }
  | { kind: "rate_limited"; retryAfterMs?: number }
  | { kind: "rejected"; reason: "invalid_destination" | "policy" | "unknown" };

async function sendThroughTransport(
  request: { destination: string; message: string; requestId: string },
  post: (request: { destination: string; message: string; requestId: string }) => Promise<Response>,
): Promise<TransportResult> {
  const response = await post(request);
  if (response.status === 429) {
    const seconds = Number(response.headers.get("Retry-After"));
    return {
      kind: "rate_limited",
      retryAfterMs: Number.isFinite(seconds) ? seconds * 1_000 : undefined,
    };
  }
  if (response.status === 400) {
    return { kind: "rejected", reason: "invalid_destination" };
  }
  if (!response.ok) {
    return { kind: "rejected", reason: "unknown" };
  }

  const body = (await response.json()) as { id?: string };
  return { kind: "accepted", externalId: body.id };
}
```

The adapter should be covered by contract tests for authentication headers, timeout behavior, response parsing, and idempotency propagation. The policy layer gets unit and concurrency tests. Those are different test responsibilities; mixing them makes failures hard to diagnose and makes a transport migration more expensive than it needs to be.

## What should I measure before changing the SMS OTP API?

I would ship one weekly increment with a useful audit view before building a large communications platform. The minimum metrics are challenge creation, send acceptance, delivery evidence where available, resend denials, verification success, expired challenges, failed attempts, and session issuance. Count them by correlation ID and keep phone numbers out of ordinary analytics.

Watch the gaps between events. A high acceptance rate with a low verification rate can indicate delivery, UX, or fraud trouble. A high resend-denial rate can mean the cooldown is too aggressive, or it can mean an attacker is probing the endpoint. Those hypotheses require different changes, so do not collapse them into one “OTP success rate.”

The catch is that this design isn't suitable when your product needs a managed multi-channel identity workflow, regulated voice controls, or a mature global fraud policy on day one. In that case, the narrow transport boundary is the wrong scope. Keep the application-side session decision and audit identifiers anyway. A generic SMS endpoint is also a poor fit for a workflow that needs phishing-resistant authentication; use a stronger authenticator for that job.

I would stick with this narrow boundary when the requirement is an auditable healthtech notice plus a modest Node.js SMS OTP login. It is small enough to understand, strict enough to prevent the common replay and resend mistakes, and flexible enough to replace the delivery component later. Ship weekly. Make the state transitions boring. Then spend the next hour on the feature that pays the bills.

## References

- https://pages.nist.gov/800-63-3/sp800-63b.html
- https://datatracker.ietf.org/doc/html/rfc7489
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/429
- https://nodejs.org/api/crypto.html#cryptorandomuuidoptions
