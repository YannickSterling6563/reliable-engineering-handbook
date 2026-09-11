# Malformed Password Reset Variables: Validate Email Templates Before API Submission

Short answer: validate password-reset template variables against an explicit contract, render a local preview, and record the rendered message's identifiers before any transactional email API call.

That order matters for an edtech SaaS sending a security notice that needs an auditable delivery record. A provider can accept a syntactically valid request even when the application supplied the wrong field name. The useful boundary is therefore inside the application: malformed template input stops there, while a valid message leaves behind enough metadata to trace what was requested without storing the reset secret.

For a one-person SaaS, this is an integration-effort decision. The mail transport is undifferentiated infrastructure; the template contract and audit semantics belong to the product. Keep those two concerns separate and a provider change doesn't force a rewrite of the safety checks.

## Why can a password reset email preview miss malformed template variables?

There are three representations in play: application data, template placeholders, and the API payload. They can disagree while each still looks reasonable in isolation. The application might produce `resetUrl`, the template might ask for `reset_url`, and the transport adapter might serialize a `variables` object without knowing which spelling the template expects.

Previewing only after an API call makes that disagreement expensive to diagnose. It mixes a deterministic rendering problem with credentials, network access, provider-specific validation, and delivery state. A local preview gives a much tighter question: given this exact template revision and these exact variable names, can the application produce the intended subject, text, and HTML?

The preview is not proof of delivery.

It is proof that rendering completed under the application's contract. Delivery needs a separate record from the transport adapter. That distinction is especially useful for a school account reset: support may need to establish which template revision was requested, for which internal user, and when, but nobody needs the raw token copied into an audit table.

## The constraint that changed the design

Names drift.

The tempting design is a loose dictionary passed straight to a hosted template. It ships quickly, right up to the first rename. Imagine a profile model changing `studentName` to `learnerName` while a reset handler still builds the old object and a remote template expects `student_name`. TypeScript can verify the handler's local shape, but it cannot infer the remote placeholder contract from an untyped dictionary. A provider-side preview then adds another translation layer: the adapter may rename the container to `variables`, yet preserve the mismatched key inside it. By the time an API error reaches a log, the team is comparing three names across three systems while a password-reset request waits. The durable fix isn't another conditional around that response. It is to choose one application-level name, validate it at the renderer boundary, and make every transport adapter consume the already-rendered result. One contract turns a vague delivery failure into a local, repeatable test.

The better small-system boundary is boring: one typed input, one renderer, and one transport interface. The renderer owns required fields. The adapter owns the provider request and maps its response to a small delivery receipt. The audit writer owns neither message content nor provider behavior; it records the transition between product intent and transport acceptance.

That split also keeps the evidence honest. An `accepted` result means the transport accepted the request. It does not mean the recipient opened the message, the mailbox placed it in an inbox, or the reset succeeded. I'm not sure any single event can answer all three questions across every mail system; resolving them requires separate transport events and product-side reset-consumption data.

Fail locally.

Use an application error code such as `TEMPLATE_VARIABLE_MISSING`, rather than leaking a downstream provider's wording through the rest of the codebase. It gives tests and logs a stable failure category even if the delivery adapter changes.

## The smallest working TypeScript implementation

The template below is intentionally local. Its two placeholders form a contract, and the renderer fails before transport when either value is absent or blank. HTML and text are rendered separately because an email client may use either body. The token stays inside the generated URL and never enters the audit event.

```ts
type ResetTemplateInput = {
  studentName: string;
  resetUrl: string;
};

type RenderedEmail = {
  subject: string;
  text: string;
  html: string;
  templateRevision: string;
};

class TemplateContractError extends Error {
  readonly code = "TEMPLATE_VARIABLE_MISSING";

  constructor(readonly field: keyof ResetTemplateInput) {
    super(`Required template variable is missing: ${field}`);
  }
}

function requireValue(
  input: ResetTemplateInput,
  field: keyof ResetTemplateInput,
): string {
  const value = input[field]?.trim();
  if (!value) throw new TemplateContractError(field);
  return value;
}

function escapeHtml(value: string): string {
  return value
    .replaceAll("&", "&amp;")
    .replaceAll("<", "&lt;")
    .replaceAll(">", "&gt;")
    .replaceAll('"', "&quot;")
    .replaceAll("'", "&#039;");
}

function renderResetEmail(input: ResetTemplateInput): RenderedEmail {
  const studentName = requireValue(input, "studentName");
  const resetUrl = requireValue(input, "resetUrl");

  const parsedUrl = new URL(resetUrl);
  if (parsedUrl.protocol !== "https:") {
    throw new Error("Reset URL must use HTTPS");
  }

  return {
    subject: "Reset your learning account password",
    text: [
      `Hello ${studentName},`,
      "A password reset was requested for your learning account.",
      `Continue: ${parsedUrl.toString()}`,
      "If you did not request this, you can ignore this message.",
    ].join("\n\n"),
    html: [
      `<p>Hello ${escapeHtml(studentName)},</p>`,
      "<p>A password reset was requested for your learning account.</p>",
      `<p><a href="${escapeHtml(parsedUrl.toString())}">Reset password</a></p>`,
      "<p>If you did not request this, you can ignore this message.</p>",
    ].join(""),
    templateRevision: "reset-notice-3",
  };
}
```

The preview can now run in a test or a protected internal screen without sending mail. Use inert example data. Don't paste a production token into a fixture.

```ts
const preview = renderResetEmail({
  studentName: "Avery Chen",
  resetUrl: "https://accounts.example.edu/reset?token=example-token",
});

if (!preview.html.includes("Avery Chen")) {
  throw new Error("Preview did not render the student name");
}

if (!preview.text.includes("https://accounts.example.edu/reset")) {
  throw new Error("Preview did not render the reset link");
}
```

The transport boundary stays small. This is the bit worth outsourcing because SMTP details and provider payloads don't create product differentiation. The application's interface asks only for an accepted-message identifier; each adapter can translate its vendor-specific response behind that boundary.

```ts
type DeliveryRequest = {
  recipient: string;
  message: RenderedEmail;
};

type DeliveryReceipt = {
  messageId: string;
  acceptedAt: string;
};

interface EmailTransport {
  send(request: DeliveryRequest): Promise<DeliveryReceipt>;
}

type AuditEvent = {
  event: "password_reset_email_accepted";
  accountId: string;
  messageId: string;
  templateRevision: string;
  acceptedAt: string;
};

async function sendResetNotice(
  transport: EmailTransport,
  accountId: string,
  recipient: string,
  input: ResetTemplateInput,
): Promise<AuditEvent> {
  const message = renderResetEmail(input);
  const receipt = await transport.send({ recipient, message });

  return {
    event: "password_reset_email_accepted",
    accountId,
    messageId: receipt.messageId,
    templateRevision: message.templateRevision,
    acceptedAt: receipt.acceptedAt,
  };
}
```

Notice what is missing from `AuditEvent`: the reset URL, token, rendered HTML, and email address. The event connects an internal account, a template revision, and a transport identifier. Retention and access rules still depend on the school's obligations and the SaaS's policy, so this shape is a starting boundary rather than a universal compliance claim.

## What I would change at scale

At low volume, a synchronous render followed by a send is easy to understand and fast to ship. At higher volume, move the delivery request to a durable queue and give every request an application-generated idempotency key. The worker can retry transient transport failures while the product stores one logical request. Define the adapter's retry classification from the chosen transport's documented responses; don't guess from a status code alone.

Template changes also deserve release discipline. Compile every known fixture in continuous integration, snapshot the subject and both bodies, and make a reviewer inspect the rendered link target. A fixture with `studentName: "Sam & Jo"` catches escaping mistakes that a plain name will not. Another with a missing `resetUrl` should assert the exact local code `TEMPLATE_VARIABLE_MISSING` and field name. That's a small gate, but it protects a weekly shipping cadence from a class of production-only surprises.

Operationally, track counts for render rejection, transport acceptance, delivery events when the transport supplies them, and successful reset consumption. Keep those states distinct. A dashboard that collapses them into “email sent” produces a clean number with vague meaning — and vague evidence is exactly what an audit trail should avoid.

## Trade-offs and the shipping rule

This approach is suitable when integration effort is the main constraint and the application can own a small rendering contract. It makes previews deterministic, keeps provider vocabulary at the edge, and leaves a narrow interface that can be replaced without touching reset logic.

The catch is that local rendering makes the application responsible for escaping, multipart output, and template deployment. Stick with a hosted template editor when non-engineers must publish copy independently and its preview plus variable validation meet the required release process. For complex localization, accessibility review, or many brands, a dedicated template pipeline may justify its extra integration work. A solo operator shouldn't quietly build a content-management system inside a password-reset function.

The shipping rule is simple: reject missing fields before the network, preview the exact renderer used for delivery, and store an audit event that proves acceptance without retaining the secret. Everything else can evolve behind those boundaries.

## References

- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API
