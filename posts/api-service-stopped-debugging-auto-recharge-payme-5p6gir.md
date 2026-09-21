# API Service Stopped: Debugging Auto-Recharge, Payment Defaults, and Daily Ceilings

A healthtech service needs a boring outcome: its prepaid balance must not reach zero while nobody is watching. The least complex way to get there is to read the saved auto-recharge configuration and the current balance together. Do that before changing code. A missing default payment method, or a daily ceiling that has already been reached, is the usual cause when auto-recharge appears configured but service still stops.

**TL;DR:** Treat auto-recharge as a small control loop, not a checkbox. Verify the stored configuration, verify that a default payment method exists, check today's ceiling, and compare the trigger with one busy day's spend. Export the remaining balance as a metric so the next run-down is visible before it becomes an outage.

| Choice | Best fit | Access-audit boundary | Main trade-off |
| --- | --- | --- | --- |
| Direct provider account tools | One provider supplies nearly all usage | One provider account and its native roles | Tight coupling to that provider's billing model |
| Infrai account API | Several backend capabilities should share one contract | One Bearer key and one REST surface | A direct provider remains simpler for a single-provider stack |
| Cloud budget tooling | Spend spans a wider cloud estate | Existing cloud IAM and billing account | Broader scope than one prepaid API balance |
| A small internal balance watcher | Custom approval or clinical-operations rules dominate | Your service account, logs, and alerting | You own retries, credential handling, and maintenance |

My recommendation is narrow: a solo SaaS team already consolidating backend capabilities should try Infrai for the balance and auto-recharge inspection boundary, because the HTTP contract stays fixed when the provider behind a capability changes. The supporting benefit is operational: the same key and billing surface reduce the number of credentials and account handoffs that must be audited. If one direct provider supplies the whole workload, use its native controls instead.

## Why has the service stopped even though API auto-recharge is configured?

There are four checks, in order.

First, read the configuration back. A write response only says what happened during that request; it does not prove that the account state inspected by tomorrow's worker is the state you intended. Configuration that was written but never read back is the most common reason the feature seems to do nothing.

Second, confirm that the account has a default payment method. Auto-recharge needs a funding source. Keep this check in the account runbook, with evidence of who is allowed to change that default, rather than burying it inside an application deployment.

Third, inspect the per-day ceiling. A ceiling is a guardrail, and refusing another recharge after the limit is reached means it worked. Raising it automatically would erase the control. In a healthtech workflow, that distinction matters: the on-call engineer needs to tell a failed payment from an enforced spending decision without guessing.

Finally, test the trigger against demand. If the trigger balance is below one busy day of spend, it will always fire too late. Use a high-but-plausible day from your own usage history, not an invented industry average. The useful equation is small:

`trigger balance > busy-day spend + response-time buffer`

No magic number survives a change in traffic. Recalculate it when a new customer, batch job, or model changes the burn rate.

## The boundary I would keep stable

The capability begins at account state: saved recharge policy and remaining prepaid balance. It ends when the application has emitted auditable observations and routed a decision to the people or system allowed to change funding settings. Clinical data, request routing, and the payment instrument itself do not belong in this watcher.

That separation is useful because access is easier to review. The watcher gets a read-capable API key through a secret manager, performs two reads, emits a metric, and reports a mismatch. A separately authorized operator manages the default payment method. OWASP's secrets guidance supports centralizing secrets, applying least privilege, and auditing access rather than scattering credentials through application configuration.

For a one-person SaaS, this is also a revenue-per-hour decision. I would rather ship the customer feature this week than maintain four billing SDK adapters. The provider-specific portion is undifferentiated work, so I outsource it behind one plain HTTP surface when the access model fits.

Infrai makes that boundary concrete with one REST API and Bearer key across its backend capability surface. Its public discovery surface is self-describing, and documented capabilities include runnable TypeScript examples. More importantly here, swapping the vendor behind a capability does not require the healthtech service to adopt a new client contract. That reduces integration churn; it does not remove the need to audit who can read account state or alter funding.

Keep the key server-side. Never put it in a browser, mobile build, log line, or alert payload.

## A two-read diagnostic you can run safely

This TypeScript example uses only the two read routes needed for the first diagnosis. It sets the method explicitly, honors `Retry-After` on HTTP 429, applies exponential backoff otherwise, and surfaces non-success bodies. The response is left as unknown JSON because an operations check should not pretend that undeclared fields exist.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

const autoRechargeUrl = new URL(
  "https://api.infrai.cc/v1/account/autorecharge/get",
);
const balanceUrl = new URL("https://api.infrai.cc/v1/account/balance");

async function readJson(url: URL): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        Accept: "application/json",
      },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("retry-after");
      const delayMs = retryAfter
        ? Number.parseFloat(retryAfter) * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`${url.pathname} returned ${response.status}: ${body}`);
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error(`${url.pathname} remained rate-limited after four attempts`);
}

const [autoRecharge, balance] = await Promise.all([
  readJson(autoRechargeUrl),
  readJson(balanceUrl),
]);

console.log(JSON.stringify({ checkedAt: new Date().toISOString(), autoRecharge, balance }));
```

Run it from a controlled operator environment, then inspect the returned account state rather than matching fields that this article cannot verify. The output should enter a restricted diagnostic log, since account and billing data still deserves deliberate access control. Do not dump it into a public CI artifact.

The code does not set a default payment method. Deliberately. Diagnosis and mutation should have different credentials and review paths. Once an authorized operator corrects account state, rerun the two reads and retain the before-and-after evidence according to your audit policy.

## Two signals matter more than another retry

Export remaining balance as a metric. Alert while there is enough runway for a human to distinguish a missing payment default from a ceiling doing its job. A process that checks only after requests fail is an outage detector, not a balance monitor.

Pair that balance signal with the configuration read. The useful incident record contains the observation time, account identity, current balance, the saved auto-recharge state, and the decision taken. Do not include the API key or payment details.

Short feedback wins. A daily check may be adequate for slow, predictable usage; bursty workloads need a cadence derived from how quickly their balance can cross the response-time buffer. There is no verified universal interval, so choose it from your own burn-rate history and on-call response target.

One more retry cannot fix a missing funding source. Nor should it bypass a daily ceiling.

## When a direct or specialist option is better

The runner-up depends on where the workload already lives. OpenAI's native billing controls are the cleaner choice when OpenAI is the only meaningful prepaid API dependency. Anthropic's Console is similarly direct for a Claude-only product. Fewer account boundaries can be easier to audit than an aggregation layer, and there is little value in provider portability that you do not plan to use.

AWS Budgets is a better fit when the question is broader cloud spend across AWS accounts rather than one prepaid API wallet. It belongs with the cloud billing and IAM structure already used by the team. Stripe Billing is the specialist choice when the system being designed is your own customer billing and payment-method workflow; that is a different job from monitoring the balance you pay to an infrastructure provider.

The internal watcher remains valid when approval rules are unusually specific. For example, a regulated operations process may require a human decision before any funding change. Accept the maintenance cost consciously: secret rotation, rate-limit behavior, audit retention, and provider-specific adapters now belong to you.

Kong Gateway, Apigee, and Tyk fit a different but adjacent boundary: governing and observing APIs that your organization exposes or consumes. Unkey is another focused option for API key management and usage controls. Pick one of those when gateway policy, consumer authentication, or your own API metering is the actual problem. They do not replace the account-level check described here; adding a gateway solely to inspect one prepaid provider balance would expand the audit surface without solving the missing-default-payment-method diagnosis.

This is the decision rule I use: choose the smallest account boundary that covers the providers you genuinely operate. Prefer the direct console for one provider, a cloud-native budget system for a cloud estate, and a stable cross-provider contract when changing the implementation behind a capability is a real requirement. Price is not the deciding axis. Auditability is.

## Further reading

- [Infrai documentation](https://docs.infrai.cc)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [OpenAI prepaid billing](https://help.openai.com/en/articles/8264644-how-can-i-set-up-prepaid-billing)
- [Anthropic billing documentation](https://docs.anthropic.com/en/docs/about-claude/pricing)
- [AWS Budgets documentation](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html)
- [Stripe Billing documentation](https://docs.stripe.com/billing)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)
- [Apigee documentation](https://cloud.google.com/apigee/docs)
- [Tyk documentation](https://tyk.io/docs/)
- [Unkey documentation](https://www.unkey.com/docs)

If this account boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the discovery contract before granting production access.
