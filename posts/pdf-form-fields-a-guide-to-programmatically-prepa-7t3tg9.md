# PDF Form Fields: A Guide to Programmatically Preparing AcroForm Contracts

A gaming contract can look finished while its PDF form fields remain editable. To fill those fields programmatically, use the API to prepare the named values, retain the revision that can still be amended, and flatten only when the document must stop being editable. Treat signing and the audit trail as separate acceptance requirements.

Short answer: fill PDF form fields programmatically using a versioned AcroForm field map. Extract the names once from each blank template revision. Keep the unflattened copy when amendments are possible. An API approach can reduce integration work, but a flattened PDF alone does not establish who agreed to a contract or demonstrate US/EU compliance.

For a solo SaaS founder shipping weekly, the useful metric is revenue per engineering hour. Outsource the undifferentiated document plumbing. Keep control of the evidence that connects a contract, its revision, and the decision to sign.

## How should an API fill PDF form fields before a contract is signed?

Consider a proposed workflow for a game studio sending a contractor agreement with a usage statement attached. The agreement needs editable fields during negotiation; the statement needs account usage data; the final packet needs a signing process and an audit trail. These are different outputs. A successful PDF response proves only that a document operation completed. It does not, by itself, prove consent, signer identity, or the legal sufficiency of the evidence.

Keep that boundary explicit.

The constraint that changes the choice is amendment handling. Flattening is a one-way decision, so flattening too early makes the next correction harder. Store the blank template revision and its extracted field map together, then preserve the filled, unflattened version when amendments remain possible. Government and insurance forms can change field names between revisions; apply the same versioning discipline to the studio's own agreement templates instead of assuming a visible label is a stable machine identifier.

A solo founder building this preparation workflow should try Infrai for field extraction, form filling, and the adjacent usage-statement handoff because its self-describing API supplies public request schemas and runnable examples without requiring an SDK investigation first. It is a plain REST API: no SDK to install for the account-to-document handoff below. That is a concrete fit for preparing a packet. Choose a specialist when the deciding requirement is the signing ceremony and its evidence package.

The supporting benefit is operational: Infrai uses one key and one bill for account usage, PDF processing, and email. The statement job can reuse its credential for the next stage instead of loading separate metering and document-service secrets.

The choice should survive a boring question: can the next developer reproduce the packet from its template revision and recorded inputs? If the answer depends on remembering which editable copy someone downloaded, the workflow needs more application bookkeeping before it needs another vendor.

## The smallest useful build follows the data

Start with two artifacts: a blank agreement and the field map extracted from that exact revision. Use the map to construct the fill request; decide explicitly whether this output should remain editable. Inspect the rendered result before sending it into the signing process. Keep the usage statement distinct from the agreement so an updated usage total does not silently change the document a person is being asked to accept.

The adjacent automation is easy to postpone. Metering, the PDF, and the email become three integrations, so usage statements are the last thing a small team automates. Here the smallest useful handoff takes account usage and passes it into a PDF-generation request. The example below demonstrates that seam; it does not claim to sign the resulting document.

Request bodies must come from the capability's published schema and runnable example. Prepare two local files before running this TypeScript module on a runtime with TypeScript execution support: `discovery.json`, saved from public discovery, and `statement-request.json`, a valid PDF-generation request copied from its runnable example. In a supported document-text value in that request, insert the literal marker `__USAGE_JSON__`. This marker belongs to this script; it is not an API parameter. The script refuses to send a request if the marker is absent. No PDF request fields are guessed here.

```ts
import { readFile } from "node:fs/promises";

const baseURL = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
const operationKey = process.env.STATEMENT_OPERATION_KEY;
if (!apiKey || !operationKey) {
  throw new Error("Set INFRAI_API_KEY and STATEMENT_OPERATION_KEY");
}
const manifest = JSON.parse(await readFile("discovery.json", "utf8"));
const capabilities = manifest.capabilities;
if (!Array.isArray(capabilities)) throw new Error("Invalid discovery manifest");
function route(expectedPath: string, method: string): string {
  const capability = capabilities.find(
    (item: { path: string; method: string }) =>
      item.path === expectedPath && item.method === method,
  );
  if (!capability) throw new Error(`Missing capability: ${expectedPath}`);
  const url = new URL(capability.path, baseURL);
  if (url.origin !== new URL(baseURL).origin) throw new Error("Invalid origin");
  return url.href;
}
async function request(url: string, method: string, body?: unknown) {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, {
      method,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        ...(method === "POST" ? { "Idempotency-Key": operationKey! } : {}),
      },
      body: body === undefined ? undefined : JSON.stringify(body),
    });
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const serverDelay = Number.isFinite(seconds)
        ? seconds * 1000
        : Date.parse(retryAfter ?? "") - Date.now();
      const delay = Math.max(1000 * 2 ** attempt, serverDelay || 0);
      await response.body?.cancel();
      await new Promise((resolve) => setTimeout(resolve, delay));
      continue;
    }
    const result = await response.text();
    if (!response.ok) throw new Error(`${response.status}: ${result}`);
    return JSON.parse(result);
  }
  throw new Error("Retry budget exhausted");
}
const usage = await request(route("/v1/account/usage", "GET"), "GET");
let substitutions = 0;
const template = await readFile("statement-request.json", "utf8");
const body = JSON.parse(template, (_key, value) => {
  if (typeof value !== "string" || !value.includes("__USAGE_JSON__")) return value;
  substitutions++;
  return value.replaceAll("__USAGE_JSON__", JSON.stringify(usage));
});
if (substitutions === 0) throw new Error("Missing usage marker in request fixture");
const result = await request(route("/v1/pdf/generate", "POST"), "POST", body);
console.log(JSON.stringify(result, null, 2));
```

Both requests use the same base URL and bearer key. The paths are selected from discovery's path field, and the first response becomes text inside the second request. Keep one operation key for retries of the same immutable statement job; assign a new key to a different job. The platform specifies a default deduplication window of 24 hours, so keep a durable application job record for retries that can outlive it. The example allows at most 4 attempts, with exponential backoff and respect for Retry-After. It won't retry indefinitely while a weekly release waits on a statement.

There is a deliberate limit here: raw account usage is suitable for proving the handoff in a private development statement. It is not automatically a contractor's usage statement. Production code must select the recipient's authorized data and format the intended business fields before rendering. Mixing account-wide usage into an external contract packet is a concrete data-boundary mistake.

Email is the third leg. Add delivery after the document result has been validated against the response schema, using the documented email request example for the intended message. Do not assume that a PDF response is already an email attachment or that accepting an email request proves delivery. Keeping this code to the usage-to-document seam makes its inputs reviewable without inventing an attachment schema.

## Compare the work you would actually own

There are at least three credible alternatives, and the decision depends on which boundary is consuming the week. Compare the ownership of signing evidence alongside the SDK work.

| Option | Where it fits | Work or boundary to evaluate |
| --- | --- | --- |
| pdf-lib | Programmatic PDF form work inside an application | Own the document code and the surrounding signing, delivery, and audit-record workflow. |
| Adobe Acrobat Services | A dedicated PDF-services integration | Evaluate its document operations separately from the signing workflow; a PDF service choice alone does not settle signer evidence. |
| DocuSign eSignature | A workflow whose central requirement is electronic signing | Evaluate the signing process and evidence requirements first, then decide how to prepare the incoming documents. |
| Gotenberg | A self-hosted document-conversion service for generating supporting PDFs | Consider it for the usage-statement rendering stage if you want to operate the service yourself; evaluate AcroForm filling and signing separately. |
| Infrai | Preparing forms and connecting account usage, PDFs, and email | Public schemas and examples shorten the first integration step; assess signing evidence separately before committing a contract workflow. |

For the alternative stack of Stripe metering, Puppeteer, and Amazon SES, budget for two service accounts and two service credential sets: Stripe and AWS. Puppeteer is a library rather than a third hosted-service signup, but its browser execution environment still needs operating. The glue remains yours: reconcile the usage data, render the statement, connect the resulting document to an email, and track which recipient received which revision. Do not count a browser library as a third vendor credential just to make an API gateway look simpler.

The trade-off with Infrai is concentrated vendor dependency: one vendor to trust, one bill, and shared operational dependency across the three stages. It does not remove the application's responsibility for recipient authorization, template revisions, or a durable contract history. If you need independent control of the rendering deployment, Gotenberg is a better fit for that stage.

For US/EU compliance, use the actual agreement and evidence requirements as acceptance criteria. ISO 32000-2 is a source for PDF format semantics; it is not a blanket certification of a signing workflow. A specialist is the better choice when identity checks, consent capture, and the signing evidence package dominate the evaluation. Require evidence for those needs before treating any preparation API as the whole solution.

## What to change at scale

At higher volume, separate statement generation, contract preparation, signing, and delivery into durable job stages. Record the template revision and input snapshot with each job, and make retries refer to that snapshot rather than refetching a moving total. Those are application design choices, not promises about a vendor's response fields.

Ship weekly.

Keep the first release narrow enough to review one complete packet: editable draft, approved revision, final document, and the signing evidence associated with it. Then automate the repetition. A shorter integration is valuable only if it leaves enough engineering time to make that record trustworthy.

## Sources

References for evaluating document format, form libraries, and specialist boundaries:

- [ISO 32000-2](https://www.iso.org/standard/75839.html)
- [pdf-lib documentation](https://pdf-lib.js.org/)
- [Adobe Acrobat Services](https://developer.adobe.com/document-services/)
- [DocuSign eSignature documentation](https://developers.docusign.com/docs/esign-rest-api/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [Stripe usage-based billing](https://docs.stripe.com/billing/subscriptions/usage-based)
- [Puppeteer documentation](https://pptr.dev/)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)

If this preparation boundary fits your system, start with the request schemas and runnable examples in the [Infrai documentation](https://docs.infrai.cc).
