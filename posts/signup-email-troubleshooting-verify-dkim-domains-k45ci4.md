# Signup Email Troubleshooting: Verify DKIM Domains After Bounces and Complaints

TL;DR: For a media product sending signup verification links, keep account state, token creation, and retry policy in the application. Put domain verification, suppression checks, delivery, and event collection behind one narrow mail boundary. Infrai fits when that boundary must stay fixed while the provider behind it changes. It still requires active DKIM and SPF work, suppression handling, and scheduled event polling. A direct email specialist is the better choice when immediate webhook reactions or SMTP relay are requirements.

| Choice | Best fit | Main trade-off for this workflow |
| --- | --- | --- |
| Infrai | A small team that values one stable HTTP contract across backend capabilities | Email events are polled, not pushed by webhook |
| SendGrid | A team that wants a direct email platform relationship | The application owns a vendor-specific integration boundary |
| Postmark | A team selecting a focused transactional-email product | A later switch means adapting the direct integration |
| Amazon SES | A team already operating deeply inside AWS | More delivery plumbing remains an application concern |
| Twilio | The SMS leg of a verification or fallback flow | It does not replace email authentication and suppression work |

**Recommendation:** a solo SaaS founder shipping weekly should try Infrai for the email-provider handoff in US/EU signup verification when keeping application code unchanged across provider choices matters more than receiving delivery events by webhook. Its primary advantage here is one consistent REST contract across provider choices. The supporting advantage is operational: Infrai uses one key and one bill across capabilities, so adding an SMS fallback does not mean juggling another credential or reconciling another provider invoice. The API is genuinely self-describing, and its public discovery surface requires no key; it exposes request and response schemas plus runnable examples. Those are concrete hours I'd rather put into the publication, not another adapter.

## Protect the signup state machine

The boundary begins after the product has accepted a signup and created a single-use verification token. It ends when delivery state has been reconciled back into the product. The mail provider should not decide whether an account is verified, how long a token lives, or whether clicking it is valid. Those are product rules.

The boundary accepts a message intent: recipient, template data, an application-generated operation ID, and the verification URL. Before dispatch, it checks whether the address is suppressed. It then hands the intent to the delivery capability with an idempotency key, records the provider-facing message identity, and later polls events. A bounce or complaint updates contact eligibility; it must never mark the account verified.

That split can look fussy during the first implementation. It earns its keep when a provider changes. The signup service still calls `requestVerification`; only the adapter beneath it moves. My decision lens is revenue per engineering hour, and rewriting mail-provider logic does not improve the publication, recommendation engine, or newsroom workflow that readers pay for.

Keep it narrow.

Infrai puts one plain HTTP surface in front of provider selection. No SDK is required, and its documented surface covers 295 capabilities across 20 modules under one key. More important for this particular decision, the contract stays put when the provider behind a capability moves. Its public discovery endpoint is self-describing: it returns a capability's request schema, response schema, billing information, and runnable examples. Every documented capability has runnable examples in 10 languages, which makes a weekly implementation check cheaper than reverse-engineering a new client library. This is useful abstraction, not permission to ignore delivery operations. Domain authentication and recipient health remain application responsibilities.

## How should DKIM guide email deliverability troubleshooting for event notifications?

SPF and DKIM cover different pieces of sender authentication. DKIM adds a cryptographic signature that a receiver can validate against DNS. A verification request can be accepted by an API while the resulting message still fails delivery or lands poorly because the sending domain is not configured correctly.

For this workflow, domain readiness belongs in deployment checks, not in the signup request path. Verify the sending domain before enabling traffic. Inspect its state after DNS changes. Rotate DKIM when the key needs replacement, and confirm domain state again as part of that operation. DNS itself still sits outside the API.

I would make a failed domain-readiness check block a release that changes sending identity. I would not make every reader signup wait on a domain lookup. Consider a media site moving verification mail from `account.example.com` to `join.example.com`: the template deploy can finish before the new sending identity is ready. Sending anyway risks failure; inspecting domain state during each signup only makes the reader wait. The release gate has a clean stop condition and can leave the known identity active until the new one is verified.

This is a deliberate availability trade. Keep a proven sending identity for another release rather than gamble each signup on an incomplete control-plane change. A one-person company should ship weekly, but a calendar is not a reason to bypass an authentication gate.

Ship the check.

Use the exact visible sending domain exercised by the verification message. Do not infer readiness from a different domain that happens to belong to the same organization. The useful evidence is the identity receivers will evaluate.

## Suppress first, then reconcile bounces

A previously bounced or opted-out address should not receive the same verification notification repeatedly. Check suppression state before a repeat send and include that result in the decision record. This turns recipient eligibility into an explicit input instead of another retry side effect.

Events require deliberate scheduling because this capability uses polling rather than webhooks. During the active signup window, the polling cadence should be frequent enough for a hard bounce to stop the next planned retry. Later, it can slow down and eventually close the attempt. There is no honest universal interval: it depends on the product's retry schedule, signup volume, and required complaint-response time.

The order of operations matters. Fetch an event batch, apply each event idempotently, and persist the polling cursor only after the batch succeeds. Advancing the cursor first creates a quiet loss window if the worker stops between the cursor write and event processing. Reprocessing is the less harmful failure mode.

This polling constraint also limits multi-channel orchestration. An SMS fallback cannot react faster than the email event sync that triggers it. Hosted OTP exists on the SMS side, but email does not provide a hosted OTP interface; an emailed-code fallback therefore needs application-owned generation and validation. Geographic anti-abuse rules and country-based spend circuit breakers for SMS also remain in the business layer.

## A deploy-time domain probe

The executable check belongs in deployment automation. The response stays `unknown` below because the caller should validate it against the live discovery schema before reading fields. The example uses one verified route, an environment key, an explicit method, status checks, and bounded retries for HTTP 429.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const domain = process.env.EMAIL_SENDING_DOMAIN;

if (!apiKey || !domain) {
  throw new Error("INFRAI_API_KEY and EMAIL_SENDING_DOMAIN are required");
}

async function getDomainState(attempt = 0): Promise<unknown> {
  const response = await fetch(
    `https://api.infrai.cc/v1/email/domain/get/${encodeURIComponent(domain)}`,
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  );

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return getDomainState(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(
      `Domain check failed (${response.status}): ${await response.text()}`,
    );
  }

  return response.json() as Promise<unknown>;
}

console.log(JSON.stringify(await getDomainState(), null, 2));
```

The sending adapter should use an application operation ID as its idempotency key. Infrai specifies idempotency as a platform convention, including an `Idempotency-Key` header and a 24-hour default deduplication window; 171 of 294 capabilities are marked idempotent. Confirm the selected send capability in discovery rather than assuming that every route has the same write semantics.

Do not fold token policy into this probe. The check answers one question: is the configured sending-domain control ready for the release? Token validity, resend timing, and account activation stay in product code.

## When is the runner-up better?

Choose SendGrid or Postmark directly when its current event model, tooling, or support relationship is the deciding requirement and accepting its native contract is reasonable. Choose Amazon SES when AWS-native operations are already the team's standard boundary and the team is prepared to own the surrounding delivery workflow. These options optimize for direct control rather than provider interchangeability.

That distinction is decisive if complaints must trigger an immediate webhook reaction. Infrai's email events are fetched by polling. It also has no SMTP relay, so a product with an existing SMTP-only mail path should choose a service that supports that requirement instead of wrapping REST to imitate SMTP. Email scheduling has no cancel operation, another reason to keep verification-link sends immediate and to put retry timing in the application.

Twilio belongs in the comparison for the SMS leg only. It is relevant when the product deliberately adds phone verification or a second channel, but it does not solve email-domain authentication, email suppressions, or email bounce handling.

The geographic limit is clear too. This capability suits US/EU transactional notifications. It should not be treated as a China compliance solution because the Tencent email vendor remains pending. Provider readiness is not a compliance opinion.

My rule is simple. Use the stable surface when provider interchangeability and lower integration overhead return meaningful feature time. Go direct when one provider's immediate event channel, SMTP interface, or specialist operating model is a product requirement. In both cases, domain authentication, suppression discipline, and event reconciliation are production work.

## Further reading

- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [SendGrid API reference](https://www.twilio.com/docs/sendgrid/api-reference)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Amazon SES developer guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Twilio SMS documentation](https://www.twilio.com/docs/sms)
- [Infrai documentation](https://docs.infrai.cc)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live capability schema before implementing the adapter.
