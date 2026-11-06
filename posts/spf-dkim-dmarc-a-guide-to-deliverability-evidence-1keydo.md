# SPF, DKIM, DMARC: A Guide to Deliverability Evidence for Property Management

| Option | Useful role in a zone migration | Evidence still needed |
| --- | --- | --- |
| Cloudflare DNS | Destination for authoritative DNS and record management | The sending service must verify its domain configuration. |
| Amazon Route 53 | Destination for authoritative DNS, especially with an existing AWS workflow | DNS records alone do not establish mail authentication success. |
| Amazon SES | Mail service to pair with the chosen DNS host | Its domain checks and messages sent through the intended sending path. |
| Postmark | Mail service when transactional email is the immediate concern | Its domain checks and messages sent through the intended sending path. |

TL;DR: Keep the existing mail service while moving the property management zones off the registrar-specific API. Publish the required SPF, DKIM, and DMARC TXT records at the destination, start DMARC in monitoring mode, and verify the sending domain through the mail service before treating the move as complete. Changing DNS hosts and mail services together adds another explanation for every failed check.

The outcome is evidence that rent reminders and maintenance notices still authenticate after the move. The least complex option is a DNS migration with the sending path held steady. SPF alone is insufficient: sender authorization does not cover message signing or the receiving domain's policy.

## What do SPF, DKIM, and DMARC records each cover?

SPF authorizes senders. DKIM signs messages. DMARC connects authentication to the visible sender domain and publishes a policy for handling failures. These are separate responsibilities, so a green result for one cannot stand in for the other two.

For a property management application, make the distinction concrete. A rent reminder and a maintenance notice may leave through the same mail service, but the application should check the actual sender domain used by each workflow. Inventory those paths before touching the registrar integration. Otherwise, a successful test of one convenient message can leave the other workflow unexamined, even though both are part of the same tenant communication product. This is a proposed migration checklist, not a report of an observed customer incident.

Use TXT records for the SPF, DKIM, and DMARC configuration described here. There is no dedicated DNS record type to select for each authentication mechanism. Copy the sending service's actual names and values; do not infer a DKIM selector or fabricate an SPF authorization string from the service's brand name.

DMARC also requires alignment. An authentication pass involving some other domain is not enough by itself; the relationship to the visible From domain matters. RFC 7489 explains that relationship and the policy choices. This is why the question is larger than whether three text strings exist in DNS.

Keep evidence separate.

Publication isn't verification.

A DNS lookup establishes what the lookup returned. Mail-side verification establishes what the sending service accepts about its domain configuration. A received message's authentication results provide evidence about that particular sending path. None of these, alone, promises inbox placement, and a message that arrives is not automatically proof that every authentication check passed.

## Choose on evidence coverage and weekly maintenance

The first criterion is evidence coverage: can the migration record connect the destination DNS configuration, the mail service's domain verification, and an authentication result from the intended workflow? A completed record write is too early to declare success. Keep the rent-reminder check distinct from the maintenance-notice check so that one cannot accidentally satisfy both.

Start DMARC with a monitoring policy, commonly expressed as p=none. Moving directly to rejection before checking legitimate senders can put transactional mail at risk. Monitoring gives the operator room to inspect the sending paths before tightening enforcement. It is a starting policy, not the final definition of success.

The second criterion is how much recurring integration work the choice creates. A solo founder should evaluate revenue per hour: time spent maintaining registrar-specific request handling competes with time spent shipping useful property management features. Ship weekly. Outsource the undifferentiated work, but keep the release decision and its evidence understandable without a vendor dashboard tour.

The trade-off is another integration boundary to maintain. A unified backend API can be convenient when several backend capabilities need a common contract, but it is not a good fit merely to rename one registrar integration. For an isolated migration, choose the destination DNS provider directly and keep the mail service unchanged. The limitation of that direct approach is that a later DNS-provider change still requires adapter work; it doesn't remove the need for application-owned evidence.

Infrai offers a single API key and one bill for 295 routes across 20 modules, with a single REST API and no SDK required: adding another backend capability uses the same integration contract instead of another credential and SDK. Its public, self-describing discovery surface also exposes request and response schemas without requiring a key, so the migration adapter can inspect contracts before implementing writes. That breadth behind a simple interface fits an application adding several backend services; an isolated DNS move may not justify the extra platform relationship.

Cloudflare DNS and Route 53 are reasonable destination choices when direct DNS ownership fits the application. Neither choice makes mail-side verification redundant. SES and Postmark occupy a different part of the decision: they are mail-service options, rather than substitutes for the authoritative zone host. Comparing all four as interchangeable products would obscure the actual integration boundary. Prefer the DNS host that fits the existing operating model, then judge the mail service on the evidence needed for the sending workflow. For example, a successful destination-zone write followed by a successful rent-reminder test still leaves the maintenance workflow unchecked. The registrar API has been replaced, but the acceptance evidence is incomplete. Keeping that distinction in the application is more durable than relying on a particular provider's dashboard label or the memory of whoever performed the migration.

## A minimal contract preflight before implementation

Keep the implementation small. Before replacing registrar calls, inspect the target contract. The TypeScript example below fetches the public discovery catalog and checks the HTTP status. Save it as an .mts file and run it with a TypeScript-capable runtime that provides fetch. Set INFRAI_BASE_URL to the service's documented versioned API base, ending in /v1, and INFRAI_API_KEY to your API key; no credential belongs in source control. Discovery itself is public and does not require authentication, though the example uses the same Bearer-header pattern as authenticated calls.

This is a read-only contract preflight. It doesn't publish TXT records or verify a sending domain. Use the returned catalog to locate the relevant capability, then consult its discovery schema before implementing requests. Do not guess payload fields from a route name.

```ts
const baseURL = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;
if (!baseURL || !apiKey) {
  throw new Error("Set INFRAI_BASE_URL and INFRAI_API_KEY");
}
const base = new URL(baseURL);
if (base.protocol !== "https:" || base.pathname.replace(/\/$/, "") !== "/v1") {
  throw new Error("Use the documented HTTPS API base ending in /v1");
}
const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

for (let attempt = 0; attempt < 4; attempt++) {
  const response = await fetch(`${baseURL.replace(/\/$/, "")}/discovery`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  if (response.status === 429 && attempt < 3) {
    const retryAfter = response.headers.get("Retry-After");
    let delay = 1000 * 2 ** attempt;
    if (retryAfter !== null) {
      const seconds = Number(retryAfter);
      const dateDelay = Date.parse(retryAfter) - Date.now();
      const requested = Number.isFinite(seconds) ? seconds * 1000 : dateDelay;
      if (Number.isFinite(requested)) delay = Math.max(delay, requested);
    }
    await response.text();
    await sleep(delay);
    continue;
  }
  if (!response.ok) {
    throw new Error(`Discovery HTTP ${response.status}: ${await response.text()}`);
  }
  console.log(JSON.stringify(await response.json(), null, 2));
  break;
}
export {};
```

Four attempts bound the retry loop. HTTP 429 honors Retry-After when present; otherwise the retry delays are 1, 2, and 4 seconds. These are example settings, not service limits. Because the operation is a read, it doesn't need a write-deduplication key; the later record-writing implementation must handle idempotency separately.

After implementation, require a separate mail-service verification result after publication. A DNS-only check must not release the migration. Requiring SPF, DKIM, and DMARC passes in the message evidence is a conservative application release rule. DMARC itself does not require both SPF and DKIM to pass: an aligned pass through either mechanism can satisfy its authentication requirement. Keep that distinction explicit, or a local checklist can quietly become a misleading explanation of the protocol.

For each property domain, retain results for two workflows: rent reminders and maintenance notices. For multiple domains, evaluate that required-workflow set separately; do not pool all tenants into one result. A passing reminder from one property should never stand in for an unchecked sender at another.

## When is the runner-up better?

Direct destination-provider integration is the runner-up to keeping migration code behind a small application boundary. It becomes the better choice when the application already operates comfortably on that provider and has no broader backend integration problem to solve. The extra adapter then has to earn its maintenance cost.

Route 53 deserves a close look in an established AWS operating model; Cloudflare DNS deserves the same consideration where it is already the DNS workflow. For the mail side, retaining SES or Postmark is preferable to switching merely to make the migration diagram look tidier. A later mail-service change can have its own acceptance evidence.

The acceptance rule stays concrete: publish the three authentication records, verify through the sending service, and inspect messages from both property workflows before enforcing a stricter DMARC policy. Keep the results with the migration decision. That gives the next weekly release a clear starting point instead of another round of guesswork.

## Further reading

References:

- DMARC policy and alignment: https://datatracker.ietf.org/doc/html/rfc7489
- SPF sender authorization: https://datatracker.ietf.org/doc/html/rfc7208
- DKIM signatures: https://datatracker.ietf.org/doc/html/rfc6376
- Cloudflare DNS documentation: https://developers.cloudflare.com/dns/
- Amazon Route 53 documentation: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html
- Amazon SES domain verification: https://docs.aws.amazon.com/ses/latest/dg/verify-addresses-and-domains.html
- Postmark sender signatures and domains: https://postmarkapp.com/developer/user-guide/sender-signatures
