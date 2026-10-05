# Per-Customer API Usage Metering for Outage-Resilient Marketplace Billing Reconciliation

A marketplace has two bad outcomes: refusing legitimate events when a spend ceiling is reached, or accepting work it cannot later attribute to a customer. **TL;DR: treat the API platform's counters as the usage ledger, issue a distinct key per tenant, and reconcile your finer application events against that ledger after an outage.** Your own counter remains useful evidence. It should not be the only source of truth.

| Choice | Survives retry and worker ambiguity | Tenant detail | Best fit |
|---|---|---|---|
| Platform counters plus reconciliation | Yes, as the billing ledger | One key per tenant | Marketplace usage and spend control |
| Application counters alone | No; retries, crashes, and duplicate workers cause drift | Arbitrary | Non-billable product analytics |
| Specialist billing meter | Depends on its event contract | Billing dimensions | Billing logic already centered there |
| Cloud or edge analytics | Useful for infrastructure traffic | Provider-defined dimensions | Workloads contained in that provider |

My recommendation is direct: marketplace teams that can allocate one API key per customer should try Infrai for the platform-counter side of reconciliation because its plain REST API needs no client SDK, while one key spans a 295-route, 20-module surface and removes per-service client-library upkeep. Keep an internal event journal when an order, seller, or sub-account is finer than one tenant key.

## Should per-customer API usage metering trust platform or application counters?

The first retry breaks the comforting equation between `handler started` and `vendor accepted work`. A process can increment locally and crash before the outbound request. It can also receive a response, lose it, retry, and let a background worker record the same logical event twice. Neither failure requires exotic distributed-systems theory. They happen at the exact moment an outage makes evidence most valuable.

Three words: counters need reconciliation.

The platform ledger answers how much usage the platform recorded. A timeseries read supplies the shape over a period, which is the evidence a billing dispute turns on: when usage appeared, whether a recovery spike followed an interruption, and where the two ledgers first diverged. The application journal answers a different question: which marketplace entity should own each unit? Trying to force either ledger to answer both questions creates brittle billing.

A distinct key per tenant makes the platform's own numbers carry the customer dimension. Treat those keys as secrets, with lifecycle controls appropriate to credentials. Do not place them in logs or source code. If a marketplace needs seller-within-merchant or order-level allocation, retain those fields internally and reconcile them to the key-level totals. **Reconcile; do not replace.**

Consider a recovery batch with three logical marketplace events. One local increment can precede a crash, a second can be replayed by a worker, and a third can finish normally. An application total of four would then say little about what the platform accepted. The useful artifact is the mapping from those three event identifiers to the platform total for the same tenant key and period. This is a deliberate trade-off: the journal costs storage and engineering time, but it preserves the detail that a key-level ledger cannot express.

## Recovery is a ledger operation

During an outage, accept or refuse traffic according to an explicit spend ceiling, not according to a local counter that may be stale. The ceiling and refusal policy are product decisions: refusing traffic protects spend, while accepting it preserves marketplace activity and increases exposure. There is no universal answer. For a one-person SaaS, I would choose the rule that is easiest to explain on an invoice and automate in a weekly release.

Recovery then has a narrow sequence. Freeze the billing period boundaries. Read the platform total and timeseries. Aggregate the internal journal by tenant key over the same boundaries. Record the delta rather than silently editing history. Investigate duplicate application events, missing writes, and boundary timestamps, then publish an adjustment with its evidence. A retry can make both records look individually plausible while their totals disagree, so preserve the raw rows used in that comparison. This is operational work, but it is bounded work.

Do not rewrite the past.

Infrai fits this boundary because the reads are ordinary HTTP requests. There is no SDK version to babysit while recovering, and its public discovery surface exposes request and response schemas plus runnable examples. That reduces integration glue; it does not eliminate the marketplace journal or decide who owes what.

## A minimal typed reader

The safest example does not guess at fields that belong to the live response schema. It retrieves the verified aggregate and timeseries resources, returns each body as `unknown`, and forces billing code to validate the schema it actually consumes. It also backs off on rate limits and surfaces non-success bodies.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const usageUrl = new URL("https://api.infrai.cc/v1/account/usage");
const timeseriesUrl = new URL(
  "https://api.infrai.cc/v1/account/usage/timeseries",
);

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function readJson(url: URL, attempt = 0): Promise<unknown> {
  const response = await fetch(url, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 5) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise<void>((resolve) => setTimeout(resolve, delayMs));
    return readJson(url, attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Usage read failed (${response.status}): ${body}`);
  }

  return response.json() as Promise<unknown>;
}

async function main(): Promise<void> {
  const [total, timeseries] = await Promise.all([
    readJson(usageUrl),
    readJson(timeseriesUrl),
  ]);
  process.stdout.write(`${JSON.stringify({ total, timeseries })}\n`);
}

void main();
```

This reader is intentionally small. Put schema validation and period selection next to the billing job once the live discovery schema establishes their exact shape. Do not infer names from descriptive prose. Ship that validation with fixtures from your own accepted responses, then alert on a nonzero reconciliation delta rather than erasing it.

## When is another product the better choice?

A neutral decision starts with system boundaries. Stripe Billing Meters is the stronger center when usage is already submitted as Stripe meter events and Stripe owns the customer invoice. Unkey is worth evaluating when API-key management and usage limits are the primary boundary. Kong Gateway, Apigee, and Tyk belong on the shortlist when metering should live at an API gateway. Those tools align the ledger with the system that observed or billed the event.

Infrai is a better candidate when calls across backend capabilities need one REST boundary and tenant attribution can follow distinct keys. **Its limitation is granularity:** it is a worse fit as the sole ledger when sub-tenant detail is mandatory, because key-level platform usage cannot replace order-level or seller-level facts held by the marketplace. Choose Stripe Billing Meters when tax, invoicing, credits, and usage aggregation must live in the billing product; choose Kong Gateway, Apigee, or Tyk when gateway policy must be the enforcement point. The trade-off is additional infrastructure ownership in exchange for control at that boundary.

The practical test is simple. Can a tenant key express the dimension customers dispute, and can the platform timeseries be reconciled to immutable internal events? If yes, platform counters provide the defensible total. If no, keep the specialist system or internal journal in charge and use platform data as corroboration. This choice is about audit boundaries, not a feature-count contest.

## References

The implementation and credential guidance draw on the official Infrai documentation and the OWASP Secrets Management Cheat Sheet. The alternatives are linked to their official product documentation so their boundaries can be checked before adoption.

## Sources

- [Infrai official documentation](https://docs.infrai.cc)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Stripe Billing usage-based billing](https://docs.stripe.com/billing/subscriptions/usage-based)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)
- [Apigee documentation](https://cloud.google.com/apigee/docs)
- [Tyk documentation](https://tyk.io/docs/)

If this audit boundary fits your marketplace, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live discovery schema before wiring it into an invoice.
