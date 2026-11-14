# Node.js Password Reset Email APIs — 3 Recipient States for Logistics SaaS

A short send request saves setup time; a recipient policy saves repeated delivery mistakes. For a one-person logistics SaaS, choose the simplest transactional email API whose bounce evidence you can turn into a suppression decision before the next password reset. Keep that policy small enough to maintain while you ship weekly.

| Option | Reason to shortlist it | Boundary to settle before choosing |
| --- | --- | --- |
| Resend | Evaluate its email API and webhook documentation together. | Establish which events should change your recipient state. |
| Postmark | Evaluate its transactional sending and bounce documentation together. | Establish how bounce handling fits your account recovery process. |
| Amazon SES | Evaluate its sending, notification, and suppression documentation together. | Decide how much surrounding AWS integration you want to own. |
| Shared REST platform discussed below | Plain REST API with no SDK to install. Public discovery exposes request and response schemas without a key, letting you inspect the sending contract before integration. | Communication events use polling; the trade-off is reconciliation delay. |

TL;DR: Start with a direct email API and a durable suppression check before every password reset send. Prefer an event-driven integration if your recovery workflow needs prompt bounce updates. A polling integration fits when its detection delay is acceptable. API acceptance does not prove that a dispatcher received the message.

## How should a small SaaS email API handle password reset bounces?

Use three application states: unknown, eligible, and suppressed. These are proposed application labels, not vendor response fields. Unknown means you lack enough evidence to apply your local eligibility policy. Eligible means that policy permits an attempt. Suppressed means a recorded reason blocks further sends until deliberate review or verified correction.

Make suppression sticky.

Record the reason. Don't reset it.

Consider an illustrative logistics account, `dispatcher@example.com`. The dispatcher requests a password reset while assigning a delivery. A permanent invalid-recipient result should stop the next reset attempt to that address. A later request from the same account is evidence of demand, not evidence that the mailbox exists. Automatically clearing suppression on every request would erase the protection precisely when repeated clicks create more sends.

The first decision criterion is control over recipient state. Before adopting any provider, establish how you will obtain bounce evidence, associate it with the attempted recipient, and persist a suppression reason. Keep temporary delivery trouble separate from a permanent invalid-recipient decision. Treat an unrecognized event as something to inspect rather than guessing that it proves a mailbox is invalid.

A successful delivery event should not automatically reverse an earlier permanent suppression either. Events can describe different attempts. Your application needs an explicit rule for correction and review, with enough context to explain which address and attempt caused the decision. This is a design requirement for the integration, not a claim that providers share one event schema or ordering guarantee.

Outsource the undifferentiated. Let a provider transport the email; retain the business rule that decides whether a logistics account may receive another recovery attempt.

## How much bounce detection delay can the workflow tolerate?

The second criterion is how quickly bounce evidence reaches the suppression check. With polling, a send can occur after a bounce happens but before the application has fetched and processed that evidence. Choose a reconciliation interval as an operational requirement, then evaluate whether it is sufficient for account recovery. Do not describe the interval as a delivery guarantee.

For example, a proposed 60-second polling interval leaves a window for repeat requests between polls, and actual detection also depends on event availability and processing. A local cooldown can limit repeated attempts, but it does not establish that an address is valid. A webhook design still needs validation, durable processing, and duplicate handling; changing the transport does not remove those responsibilities.

Acceptance isn't delivery.

Check both boundaries.

Sender authentication is a separate concern. SPF defines authorization for sending hosts; it does not prove that a recipient mailbox exists or that a password reset arrived. Check sender setup and recipient suppression independently. Otherwise, a clean send response can distract you from the failure that actually keeps the dispatcher locked out.

Use a revenue-per-hour lens here. The useful comparison is how much recurring attention the recovery path requires while you ship weekly. Count event handling, suppression review, and recovery support alongside the initial integration. Avoid an elaborate messaging platform for one reset email, but do not remove the small amount of state that makes repeat sends accountable.

## A Node.js example that keeps suppression sticky

This TypeScript program exercises the application policy. Its event labels are local adapter outputs, not provider payloads. Run it with a TypeScript runner such as `tsx`; the default demonstration needs no API credentials and sends no email. The optional sending branch uses the same suppression check and a plain HTTP request.

```ts
type RecipientState = "unknown" | "eligible" | "suppressed";
type EvidenceKind = "permanent-invalid" | "temporary-failure" | "delivered";
type Evidence = {
  id: string;
  recipient: string;
  kind: EvidenceKind;
};
type Recipient = {
  state: RecipientState;
  reason?: string;
};

const recipients = new Map<string, Recipient>();
const processed = new Set<string>();

function applyEvidence(event: Evidence): void {
  if (processed.has(event.id)) return;
  const current = recipients.get(event.recipient) ?? { state: "unknown" };

  if (event.kind === "permanent-invalid") {
    recipients.set(event.recipient, {
      state: "suppressed",
      reason: event.id,
    });
  } else if (event.kind === "delivered" && current.state !== "suppressed") {
    recipients.set(event.recipient, { state: "eligible" });
  }

  processed.add(event.id);
}

function mayAttemptReset(recipient: string): boolean {
  // Policy: unknown addresses may receive an initial attempt.
  return recipients.get(recipient)?.state !== "suppressed";
}

const recipient = "dispatcher@example.com";
const invalid: Evidence = {
  id: "example-invalid-1",
  recipient,
  kind: "permanent-invalid",
};

console.log("Initial attempt permitted:", mayAttemptReset(recipient));
applyEvidence(invalid);
applyEvidence(invalid);
applyEvidence({ id: "example-delivered-2", recipient, kind: "delivered" });
console.log("Repeat attempt permitted:", mayAttemptReset(recipient));

if (mayAttemptReset(recipient)) {
  throw new Error("Suppression was unexpectedly cleared");
}

async function sendReset(): Promise<void> {
  const {
    INFRAI_BASE_URL, INFRAI_API_KEY, EMAIL_SEND_BODY,
    EMAIL_RECIPIENT, RESET_ATTEMPT_ID,
  } = process.env;
  if (!INFRAI_BASE_URL || !INFRAI_API_KEY || !EMAIL_SEND_BODY ||
      !EMAIL_RECIPIENT || !RESET_ATTEMPT_ID) {
    throw new Error("Sending requires all five configuration variables");
  }
  // Copy the current documented send JSON into EMAIL_SEND_BODY.
  const body = JSON.stringify(JSON.parse(EMAIL_SEND_BODY));
  const url = new URL("/v1/email/send", INFRAI_BASE_URL);
  for (let attempt = 0; attempt < 3; attempt++) {
    if (!mayAttemptReset(EMAIL_RECIPIENT)) {
      throw new Error("Recipient is suppressed");
    }
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${INFRAI_API_KEY}`,
        "Content-Type": "application/json",
        "Idempotency-Key": RESET_ATTEMPT_ID,
      },
      body,
    });
    const result = await response.text();
    if (response.status === 429 && attempt < 2) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const delay = Number.isFinite(seconds) ? seconds * 1000 :
        Number.isFinite(date) ? date - Date.now() : 0;
      await new Promise(resolve => setTimeout(resolve,
        Math.max(1000 * 2 ** attempt, delay)));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Email request failed (${response.status}): ${result}`);
    }
    console.log("Email API accepted the request");
    return;
  }
}

if (process.env.EMAIL_SEND_BODY) {
  sendReset().catch(error => {
    console.error(error);
    process.exitCode = 1;
  });
}
```

For the optional live branch, set `INFRAI_BASE_URL` to the documented API origin and `INFRAI_API_KEY` to your secret key. Set `EMAIL_SEND_BODY` to valid send-request JSON from public discovery, `EMAIL_RECIPIENT` to the same recipient in that JSON, and `RESET_ATTEMPT_ID` to an identifier retained across retries of one logical reset. The example accepts configuration rather than inventing vendor request fields. It makes at most 3 attempts, honors `Retry-After`, and surfaces the actual error response. Keep secrets and reset tokens out of logs.

The output permits the initial attempt and blocks the repeat. A duplicate event has no additional effect, and a delivery event cannot clear suppression. Those are the two policy properties this program demonstrates. Temporary failure leaves the current state alone; it does not establish permanent invalidity.

The in-memory maps are for demonstration. A deployed service needs durable recipient records and processed-event identifiers, with the event write and state update committed together. Namespace event identifiers by provider. Keep recipient identity consistent between send requests and event processing; do not casually rewrite mailbox addresses and assume the result identifies the same recipient.

There is another limitation: this check cannot prevent concurrent sends or suppress an attempt already in flight. Put it immediately before dispatch, and define how durable job and recipient records coordinate concurrent work. Request retries also need a stable identifier for the same logical reset attempt. Event deduplication and send idempotency solve different problems.

## When is the runner-up the better choice?

For a workflow that prioritizes prompt bounce updates, begin the evaluation with the event documentation for Resend and Postmark. Compare their actual payloads, retry behavior, and suppression controls against your application policy before selecting either. A familiar integration with a clear recovery path can be a better choice than the shortest sample request.

Amazon SES deserves a place in the shortlist when AWS already belongs in the application's operating environment. Read its notification and suppression documentation as part of the sending design. The decision is whether that integration fits your available maintenance time; this note makes no claim that one provider delivers faster or has better uptime.

A plain REST integration with polling becomes the better fit when you can accept reconciliation delay and want to avoid maintaining another SDK dependency. Public request schemas also let you check the sending contract before committing to integration work. Keep password reset token generation and validation in the application, and use immediate sending for this workflow. Scheduling and multi-channel orchestration add little to suppressing an invalid dispatcher address.

Infrai is a plain REST API with no SDK to install, using one key and one bill across 295 routes in 20 modules. Anything that can send an HTTP request can call it, so the recovery sender needs no client library version to maintain. This unified API covers email, storage, scheduling, and observability with unified authentication and billing. For a solo founder already using other backend capabilities there, adding recovery email does not introduce another vendor credential or another billing account to reconcile. That administrative benefit matters only if the platform's event polling fits the recovery workflow. It cannot substitute for prompt suppression when the business requires it.

Public discovery is also self-describing: each capability exposes JSON Schema and runnable examples in 10 languages. Read the TypeScript example alongside its schema before wiring the sender into account recovery. This gives a solo maintainer a concrete contract to inspect while keeping the implementation to an HTTP request.

Ship weekly. Keep the recovery policy reviewable: one place to record invalid-recipient evidence, one check before dispatch, and an explicit decision about how a corrected address becomes eligible again.

## Sources

References for sender authentication and the alternative integrations discussed above:

- RFC 7208, Sender Policy Framework: https://datatracker.ietf.org/doc/html/rfc7208
- Resend webhook event types: https://resend.com/docs/webhooks/event-types
- Postmark bounce API: https://postmarkapp.com/developer/api/bounce-api
- Amazon SES notifications: https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html
- Amazon SES account-level suppression: https://docs.aws.amazon.com/ses/latest/dg/sending-email-suppression-list.html
