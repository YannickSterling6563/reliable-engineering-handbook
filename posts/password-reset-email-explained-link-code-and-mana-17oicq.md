# Password Reset Email Explained: Link, Code, and Managed OTP Trade-offs

A one-person SaaS has a hard constraint: every hour spent maintaining authentication plumbing is an hour not spent shipping the weekly product change. My choice is to keep password-reset policy in the application, send a single-use reset link by default, and place email behind the same narrow delivery boundary used for order receipts after payment settles. Add an email code only when users genuinely need cross-device recovery. Use managed OTP when outsourcing expiry, attempt limits, and verification state is worth the integration and migration cost.

**TL;DR:** links minimize user input and application surface area; codes travel across devices more easily but need a verification flow; managed OTP removes sensitive state from the application but increases external coupling. Treat a second method as an alternate interaction over the same recovery transaction, not as an automatic retry that doubles the attack surface.

## Should a password reset email use a link, fallback code, or managed OTP?

An order receipt and a password reset are both transactional messages, but they do not carry the same risk. The receipt is evidence for the customer after payment settles. A reset message grants a path to account control. Sharing a transport adapter, template renderer, delivery telemetry, and deployment process is useful. Sharing tokens, retry policy, or business events is not.

That distinction drives the integration-effort calculation. The application already needs to accept a `payment.settled` event, render an order identifier and amount, submit a message, and record a provider-neutral delivery ID. Recovery can reuse the submission boundary without teaching the authentication domain about SMTP responses or a commercial API. One adapter. Separate policies.

Keep the boundary small:

| Concern | Receipt | Password reset |
| --- | --- | --- |
| Trigger | Payment becomes settled | User requests recovery |
| Secret in message | No | Single-use link token or code |
| Retry goal | Deliver the record | Do not create parallel valid challenges |
| User response | Usually none | Redeem before expiry |
| Durable record | Order and send outcome | Audit event, never the raw secret |

The table is the reason I would not bolt a second email SDK directly into the recovery handler. The saved setup hour becomes recurring coordination work every time logging, templates, credentials, or deployment changes. For a solo operator, maintenance frequency matters more than the length of the initial quickstart.

Links have limits.

Picture the ordinary awkward case. A customer pays on a laptop, receives the order receipt on a phone, and later starts password recovery on that laptop. The receipt only needs to arrive and remain readable. A reset link opened from the phone moves recovery into a different browser, where the customer may not have the context that prompted the request. A code avoids that handoff, but now the laptop needs an entry form and the backend must distinguish a typo from an expired, consumed, superseded, or over-attempted challenge without leaking account state. Managed OTP can absorb part of that machinery, yet its lifecycle becomes an external contract that has to fit the application's support and security policy. This concrete path is why device switching, not the email-sending call, should decide between the three options.

Codes do too.

## The concrete constraint that changed the choice

Cross-device use is the deciding constraint. A link is pleasant when the inbox and signed-out browser are on the same device. A short code can be read on a phone and entered on a laptop without moving a long URL. That benefit has a bill: the application needs an entry screen, attempt limits, expiry, single-use consumption, and protection against revealing whether an account exists.

Small scope wins.

Start with one recovery transaction and one default interaction. Store a digest of a random secret, its expiry, its consumed state, and the account relationship. Return the same public response for known and unknown addresses. OWASP also recommends consistent messages and response timing, side-channel delivery, random tokens, secure storage, single use, expiry, and rate limiting for forgot-password flows. Those are system requirements, not email-provider features.

A fallback must not mean "send both every time." Let the user deliberately request the code interaction, invalidate the earlier challenge, then issue one replacement. Otherwise two live credentials can exist for one request, support cannot tell which one the user is trying, and retries become harder to reason about.

Managed OTP changes ownership. The external service can own challenge creation and verification, while the application owns account lookup, neutral responses, session invalidation, and the final password change. That is a sensible outsourcing line only if the service's lifecycle rules match the product. Before integrating, verify expiry configuration, resend behavior, attempt limits, idempotency, audit export, regional requirements, and how recovery data can be migrated. Missing any one of those can turn a short integration into permanent operational work.

The limitations are plain. A link-only flow is a poor fit for frequent cross-device recovery. A self-built code flow is a poor fit when nobody can review and operate its attempt controls. Managed OTP is a poor fit when its fixed lifecycle, data handling, or migration path conflicts with product requirements. In each case, choose the alternative whose ongoing ownership matches the hours actually available; none is a universal fallback.

## The smallest implementation I would ship

The useful abstraction is not `sendResetEmail`. It is a transactional message port plus a recovery store. The receipt flow can call the same port with different data and a different template, while authentication rules stay local. Mustache's default escaping matters here: ordinary variables are HTML-escaped, while triple braces or `&` produce unescaped output. Keep untrusted receipt and account fields on escaped variables.

```ts
type TransactionalMessage = {
  to: string;
  template: "order-receipt" | "password-reset-link";
  data: Record<string, string>;
  idempotencyKey: string;
};

interface MessagePort {
  send(message: TransactionalMessage): Promise<{ deliveryId: string }>;
}

interface RecoveryStore {
  replaceActive(input: {
    accountId: string;
    secretDigest: string;
    expiresAt: Date;
  }): Promise<void>;
}

declare function randomRecoverySecret(): string;
declare function digest(secret: string): Promise<string>;
declare function publicResetUrl(secret: string): string;

export async function requestPasswordReset(
  account: { id: string; email: string } | null,
  store: RecoveryStore,
  messages: MessagePort,
  now: Date,
): Promise<{ accepted: true }> {
  if (!account) return { accepted: true };

  const secret = randomRecoverySecret();
  const secretDigest = await digest(secret);
  await store.replaceActive({
    accountId: account.id,
    secretDigest,
    expiresAt: new Date(now.getTime() + 15 * 60 * 1000),
  });

  await messages.send({
    to: account.email,
    template: "password-reset-link",
    data: { resetUrl: publicResetUrl(secret) },
    idempotencyKey: `password-reset:${account.id}:${secretDigest}`,
  });

  return { accepted: true };
}
```

This is deliberately incomplete around token generation and persistence: those belong to reviewed cryptographic and database components, not a copy-pasted article helper. The important shape is visible. The secret is delivered but not stored raw, a new request replaces the active challenge, and the response does not disclose account existence. The `15`-minute value is an example policy, not a universal recommendation; choose and test an expiry against the product's actual delivery latency and support burden.

For the receipt path, call `messages.send` only after the payment state transition has been durably accepted. Give that business event its own idempotency key. A delivery retry should resubmit the same logical receipt, while a recovery resend should follow the recovery policy and must not silently mint a pile of valid secrets. Similar transport does not imply identical retry semantics.

Test the boundaries that can cost a support hour: duplicate settlement events produce one logical receipt, unknown recovery addresses receive the same public response, expired and consumed secrets fail, a newly issued challenge invalidates the prior one, template data stays escaped, and delivery failures remain retryable without replaying the business event. Use a fake `MessagePort` in unit tests and one adapter contract test against the chosen transport.

## What I would change at scale

At higher volume, I would put transactional sends behind a durable queue, split authentication telemetry from general email analytics, and make the recovery transaction a state machine with explicit `active`, `consumed`, and `expired` outcomes. I would also test link scanners: automated security tools may visit URLs before the user, so a GET should display a confirmation page rather than consume the secret or change the password. The state-changing step belongs behind an intentional user action.

For code entry, add an explicit attempt counter and atomic verification. Do not treat resending as a fresh unlimited budget. If a managed OTP service takes over that state machine, keep an internal adapter and contract tests so the rest of the application does not depend on its response vocabulary. This costs a little code now and buys back future migration time. That is a good revenue-per-hour trade when the boundary also serves receipts and other transactional messages.

Compliance should follow message purpose. The FTC's CAN-SPAM guide says the law covers commercial email and distinguishes transactional or relationship content; messages whose primary purpose is transactional are treated differently, while mixed content can change the analysis. Keep reset and receipt templates focused on their requested transaction, and have counsel assess any promotional additions for the jurisdictions served.

## A practical decision rule

Ship the link flow when most users recover on one device and the application can own a small, reviewed token lifecycle. Add a code interaction when cross-device completion is a demonstrated support problem, accepting the extra UI and verification controls. Choose managed OTP when operating that lifecycle is undifferentiated work and the provider's documented controls satisfy the product's policy.

Do not choose by email API call count or a teaser price. Count integration surfaces: state ownership, UI states, retries, observability, testing, incident diagnosis, and exit cost. A solo SaaS should outsource the undifferentiated pieces, but retain the domain boundary that makes replacement possible. Then ship the next feature.

## Sources

- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- https://mustache.github.io/mustache.5.html
- https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
