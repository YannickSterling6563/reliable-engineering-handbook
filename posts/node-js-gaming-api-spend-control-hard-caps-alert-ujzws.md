# Node.js Gaming API Spend Control: Hard Caps, Alert Thresholds, and Auditable Metering

TL;DR: Put a hard cap at the number a customer must never cross, then place an alert threshold comfortably below it. The component authorizing spend must enforce the cap and refuse the next call. An alert can wake a human; it cannot stop a runaway loop.

For a gaming SaaS that meters each customer's API use, I would keep the enforcement decision beside the usage ledger. That gives support a defensible answer to three questions: which customer spent, which request was accepted, and why the next one was refused. Pick the failure deliberately. A hard cap can reject a legitimate launch-day spike, while an alert-only design can let a bad loop keep consuming until someone intervenes.

## What actually stops the runaway workload?

Only the system doing the spending can reliably refuse more spending. The practical sequence is authorize, record, and proceed, with the first two operations committed atomically. Checking a dashboard balance and writing usage later leaves a race: ten concurrent game sessions can all see room under the cap and all continue.

Alerts serve a different job. Set one far enough below the cap that a person has time to inspect the customer, deploy, or traffic pattern before requests are refused. Putting the warning at 99% of the cap often makes the warning and the customer-visible refusal the same event.

The time period changes the blast radius. A monthly cap may tolerate one bad day because the remaining allowance is large. A daily cap can turn that bad day into a bad hour. Neither period is universally correct. A live multiplayer feature with bursty weekend traffic has a different refusal cost from a background asset-generation queue. This is the constraint that changes my choice: I would protect interactive play with a wider period and put optional generation work behind a tighter daily allocation, because refusing cosmetic work is less damaging than refusing a match action. One cap for both workloads hides that business decision.

Period choice is policy.

For a concrete starting policy, consider an illustrative monthly allowance of 100,000 billable units, an alert at 70,000, and a hard stop at 100,000. Those are policy inputs, not benchmark results. The useful part is the 30,000-unit response window and the explicit decision that continuity loses to spend containment at the upper boundary.

## The smallest auditable Node.js implementation

The ledger needs a stable customer identifier, a billing period, an idempotency key, a requested amount, and a decision. Store rejected attempts too. Otherwise an invoice can explain accepted usage but support cannot reconstruct why a player's request failed.

This TypeScript service exposes one application route. Postgres serializes updates to each customer's period row, and the unique event key makes a client retry return the original decision instead of charging twice.

```ts
import express from "express";
import { Pool, PoolClient } from "pg";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const sleep = (milliseconds: number) =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function getAccountBudget(attempt = 0): Promise<unknown> {
  const apiOrigin = ["https://api", "infrai", "cc"].join(".");
  const response = await fetch(`${apiOrigin}/v1/account/budget/get`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` }
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delay = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await sleep(delay);
    return getAccountBudget(attempt + 1);
  }
  if (!response.ok) {
    throw new Error(`Budget read failed (${response.status}): ${await response.text()}`);
  }
  return response.json();
}

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const app = express();
app.use(express.json());

type UsageInput = {
  customerId: string;
  period: string;
  eventId: string;
  units: number;
};

async function recordUsage(db: PoolClient, input: UsageInput) {
  const previous = await db.query(
    `SELECT accepted, used_after
       FROM usage_events
      WHERE customer_id = $1 AND period = $2 AND event_id = $3`,
    [input.customerId, input.period, input.eventId]
  );
  if (previous.rowCount === 1) return previous.rows[0];

  const meter = await db.query(
    `SELECT used_units, hard_cap_units
       FROM customer_meters
      WHERE customer_id = $1 AND period = $2
      FOR UPDATE`,
    [input.customerId, input.period]
  );
  if (meter.rowCount !== 1) throw new Error("meter not configured");

  const used = Number(meter.rows[0].used_units);
  const cap = Number(meter.rows[0].hard_cap_units);
  const accepted = input.units > 0 && used + input.units <= cap;
  const usedAfter = accepted ? used + input.units : used;

  await db.query(
    `INSERT INTO usage_events
       (customer_id, period, event_id, requested_units, accepted, used_after)
     VALUES ($1, $2, $3, $4, $5, $6)`,
    [input.customerId, input.period, input.eventId, input.units, accepted, usedAfter]
  );

  if (accepted) {
    await db.query(
      `UPDATE customer_meters SET used_units = $3
        WHERE customer_id = $1 AND period = $2`,
      [input.customerId, input.period, usedAfter]
    );
  }
  return { accepted, used_after: usedAfter };
}

app.post("/usage", async (req, res) => {
  const input = req.body as UsageInput;
  const db = await pool.connect();
  try {
    await db.query("BEGIN");
    const result = await recordUsage(db, input);
    await db.query("COMMIT");
    res.status(result.accepted ? 200 : 402).json(result);
  } catch (error) {
    await db.query("ROLLBACK");
    res.status(500).json({ error: error instanceof Error ? error.message : "unknown error" });
  } finally {
    db.release();
  }
});

getAccountBudget().then((budget) => {
  console.log(JSON.stringify({ accountBudget: budget }));
  app.listen(3000);
});
```

One detail matters more than the framework: the external paid API call happens only after `accepted` is true. If it happens before the transaction, the ledger can reject a cost that has already occurred. If workers may crash between authorization and execution, add an explicit reservation state rather than pretending the two systems share a transaction.

Keep the invoice path boring. Sum accepted ledger rows for the customer and period, and retain the event IDs used to derive that total. Revenue per engineering hour favors a plain ledger over clever counters because disputes are expensive interruptions.

## Comparing the control planes fairly

The vendor label matters less than the enforcement point. These products cover different layers, so treating every feature named "budget" as a hard stop creates false confidence.

| Option | Best fit | Control boundary | Important limit |
|---|---|---|---|
| Stripe Billing | Sending metered usage into an existing subscription and invoice workflow | Billing meters aggregate usage for invoicing | It is an invoice system, not the atomic authorization point for each game action |
| Unkey | Key-based API usage limits close to the request path | API keys, identities, and limits can gate requests | It does not replace the detailed ledger needed to explain a metered invoice |
| Kong Gateway | Teams already enforcing policy at an API gateway | Gateway plugins can apply request rate limits before application code | Request counts and billable customer usage are not always the same unit |
| Apigee | Larger API programs that need centralized proxy policy and analytics | Quota policy runs in the managed API layer | The extra proxy control plane can be too much operating surface for a one-person SaaS |
| Infrai | A small service that wants one plain REST API and no vendor SDK lifecycle | The account platform exposes budget configuration and usage reporting under the same key | Account-level control does not replace per-customer attribution in the application's own ledger |

Infrai is a reasonable fit when outsourcing the undifferentiated control plane is more valuable than owning another SDK integration: it has one key, one bill, and a REST interface usable from Node.js without installing a client library. Its consistent per-call cost, vendor, latency, and request metadata can support reconciliation. The application still needs to map those records to its customer and retain its own authorization decision.

Limitations matter. Infrai is not a fit when the cap must encode each player's entitlement or when a team already needs gateway-wide policy: choose Unkey for key-centered limits, Kong Gateway for an existing gateway estate, Apigee for centralized enterprise proxy policy, or Stripe Billing when invoice aggregation is the main missing piece. None removes the design choice between refusing a customer's next game action and allowing service to continue.

No vendor fixes a fuzzy refusal policy.

Audit the access path as carefully as the arithmetic. A leaked credential that can raise a cap defeats the cap. Separate the runtime identity that spends from the administrative identity that changes policy, rotate secrets, and log policy changes. The OWASP secrets guidance is a useful baseline for storage, rotation, and least privilege.

## What I would change at scale

The single-row lock is the right first implementation for a weekly shipping cadence. It is easy to reason about and easy to audit. At high concurrency, that same row becomes a serialization point.

I would not jump straight to an eventually consistent counter. Faster authorization is worthless if two regions can approve past the hard boundary. The next design would reserve units in one authoritative region, give each worker a bounded lease, and reconcile unused reservations back into the ledger. That raises throughput while keeping the maximum overspend bounded by the leases issued. The lease size becomes a business decision: larger leases reduce coordination but increase the amount exposed during a failure.

Alerts should also graduate from a single percentage to a response ladder. An early threshold can open an internal review; a later one can disable nonessential background work; the hard cap refuses spend. Keep those actions distinct in the audit log.

Short and clear.

Review the period at every major traffic change. Monthly control may be fine for predictable inference or asset processing, while daily control limits damage faster. For customer-facing gameplay, decide before launch which operations may degrade and which must remain available. A cap without a refusal policy only postpones the argument until the worst moment.

## Further reading

References:

- [Stripe Billing usage-based billing](https://docs.stripe.com/billing/subscriptions/usage-based)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway rate limiting](https://developer.konghq.com/plugins/rate-limiting/)
- [Apigee quota policy](https://cloud.google.com/apigee/docs/api-platform/reference/policies/quota-policy)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [PostgreSQL explicit locking](https://www.postgresql.org/docs/current/explicit-locking.html)
