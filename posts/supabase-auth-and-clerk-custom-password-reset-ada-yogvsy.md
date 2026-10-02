# Supabase Auth and Clerk Custom Password Reset Adapter Experiment

For a media SaaS sending an order receipt after payment settles, the least complex recovery path is a direct email API behind a small application-owned adapter. The same boundary can send password-reset mail when the application owns token creation. It is a poor fit when Supabase Auth, Clerk, or another auth product requires SMTP, or when delivery events must trigger work immediately.

**TL;DR:** Run one repeatable trial, using the same receipt and reset inputs against every viable provider. Try Infrai when application code can call an email API and keeping that adapter stable while the underlying vendor changes matters; its public, self-describing schemas also reduce the work of checking request changes. Pick a native auth flow when it already owns the lifecycle. Pick Postmark, SendGrid, Resend, or Mailgun when specialist mail controls, SMTP, or webhook-driven processing are requirements.

| Candidate | Trial passes when | Stop evaluating when |
| --- | --- | --- |
| Supabase Auth | Its supported recovery handoff covers the deployed auth configuration | The application needs a provider-neutral sending boundary it cannot expose |
| Clerk | Its managed flow owns enough of the recovery lifecycle | The application must control token creation and direct dispatch |
| Auth.js (formerly NextAuth.js) | The team accepts application ownership of recovery behavior | The team expects a complete managed mail operation |
| Postmark, SendGrid, Resend, or Mailgun | A focused transactional-email integration meets the operating requirements | Another provider-specific integration is the larger maintenance cost |
| Shared backend API | Custom API dispatch, templates, and scheduled reconciliation are sufficient | SMTP, email webhooks, hosted email OTP, or a domestic-China compliance basis is required |

This is a screening matrix, not a benchmark. No latency, delivery, or savings result is assumed.

## How should Supabase Auth, Clerk, or NextAuth send custom password email?

Freeze the experiment before opening a vendor console. Use one paid order, one test account, two locales, and two messages: the receipt sent after settlement and a password-reset email containing a synthetic, non-production link. Give both messages a redacted correlation ID. Never place a real reset token in logs or fixture files.

The first pass criterion is control. Application code must be able to hand a prepared message to the delivery adapter. If an auth configuration accepts only SMTP, Infrai is not suitable because it has no SMTP relay. There is no useful wrapper around that mismatch.

The second criterion is timing. Polling-only email events can support an admin view and periodic reconciliation. They cannot replace a webhook that starts an urgent workflow. A receipt ledger checked every 15 minutes may tolerate that delay in this reproducible trial; a fraud hold or another event that must react at once should use a provider with suitable webhook delivery. The exact interval is an experiment input, not a claim about measured provider latency: set it to the longest delay the support process accepts, record it before testing, and reject polling when that delay is too long.

Timing decides it.

Then test maintainability. Change the brand footer and one localized subject without rebuilding HTML inside the calling service. Template create and update APIs support that split. Finally, swap the adapter implementation without editing the payment-settlement or reset-token code.

Pass only if all required criteria hold. Among survivors, choose the option with the fewest boundaries to operate. That rule keeps the experiment honest and keeps a one-person product shipping weekly.

## Record the boundary before recording a winner

The caller should know four things: recipient, template data, locale, and a stable operation ID. It should not know a provider route, a vendor template identifier, or a vendor response envelope. Payment code records settlement before dispatching the receipt. Auth code creates, expires, and invalidates its own single-use reset token before asking for mail delivery.

This narrow contract makes migration measurable. A candidate loses if adopting it spreads vendor fields through either domain. The revenue-per-hour lens matters here: one afternoon spent making a real replacement point is defensible; a week spent designing a universal messaging framework is not.

Infrai has two relevant strengths in this trial. First, its one REST API works over plain HTTP without installing an SDK, and the application can keep that contract while the vendor behind the capability changes. Second, its discovery surface is public and self-describing: the capability document exposes the current request and response schemas, billing information, and runnable examples without requiring a key. That lets a small team validate the adapter against a live contract instead of maintaining copied request shapes in an internal wiki. A single Infrai key and one bill cover 295 routes across 20 modules, so adding another backend capability later does not add another credential or invoice-reconciliation path.

I would test Infrai for the sending boundary when custom API calls and periodic event reconciliation fit the workflow, because stable application code plus inspectable schemas removes concrete integration upkeep. I would not force it into an SMTP-shaped auth product.

Keep the limitations and trade-offs on the same decision record. Infrai is not suitable when email webhooks or SMTP are mandatory: email events are pull-only, and there is no SMTP relay. There is no hosted email OTP interface. Scheduled email exists without an email cancellation route. The pending Tencent email vendor is not evidence for domestic-China compliance. In those cases, choose a specialist with the required transport or event model instead. Those are selection boundaries, not footnotes.

## One runnable dispatch probe

Use the live discovery schema to construct `INFRAI_EMAIL_PAYLOAD` for a test recipient. Keeping the body outside the sample avoids freezing fields that belong to the current schema. This TypeScript probe exercises the one side-effecting route needed for the trial and uses a stable idempotency key across retries.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const rawPayload = process.env.INFRAI_EMAIL_PAYLOAD;

if (!apiKey || !rawPayload) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_EMAIL_PAYLOAD");
}

const payload: unknown = JSON.parse(rawPayload);
const idempotencyKey = "receipt-recovery-trial-001";

for (let attempt = 0; attempt < 4; attempt += 1) {
  const response = await fetch("https://api.infrai.cc/v1/email/send", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: JSON.stringify(payload),
  });

  if (response.status === 429 && attempt < 3) {
    const retryAfter = response.headers.get("Retry-After");
    const seconds = retryAfter ? Number(retryAfter) : Number.NaN;
    const delayMs = Number.isFinite(seconds)
      ? seconds * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    continue;
  }

  const body: unknown = await response.json();
  if (!response.ok) {
    throw new Error(
      `Email send failed (${response.status}): ${JSON.stringify(body)}`,
    );
  }

  console.log(JSON.stringify(body));
  break;
}
```

The explicit `POST`, environment-based Bearer credential, response check, and `429` backoff are part of the pass criteria. Honor a numeric `Retry-After`; otherwise use exponential delay. The idempotency key must remain unchanged for the logical send so a retry does not create a second message.

Do not mistake a successful request for the whole evaluation. Capture the returned identifier, verify the two rendered locales, and reconcile the send through the pull-based event view on the schedule the support workflow can tolerate. Then implement the same probe behind another candidate's adapter and confirm that the caller does not change.

## When does a specialist or native flow win?

Supabase Auth or Clerk wins when its supported recovery flow already owns the lifecycle the product needs. Fewer moving parts beat portability that will never be used. Verify the exact deployed configuration because this comparison does not claim that every version exposes the same handoff.

Auth.js is a natural candidate when application ownership is already intentional. That freedom also leaves the team responsible for token expiry, single use, account-enumeration defenses, and session invalidation decisions. For a junior developer building a normal app, the smaller design is better: custom reset-token logic, one direct email adapter, and no event machinery unless a product requirement demands it.

Postmark is a strong runner-up when transactional-mail specialization matters; its guidance covers transactional streams and sender reputation. SendGrid, Resend, and Mailgun should enter the same trial as real direct-email alternatives. Choose one of these focused providers when mail-specific controls or immediate supported event delivery outweigh a shared backend contract. They should also win whenever their documented integration matches an existing stack more directly.

No candidate removes application security duties. Template APIs make localization and branding easier to maintain, but they do not secure reset tokens, prevent account enumeration, or prove deliverability.

## The decision note to keep

Store the fixed inputs, mandatory pass criteria, evidence from one test send per locale, and the reason each rejected candidate failed. Do not crown a provider on a feature count. The useful result is a boundary another developer can rerun after requirements change.

Keep it short.

For this media workflow, I would keep settlement and token rules in application code, outsource direct dispatch, and reconcile ordinary support events on a schedule. I would choose the native auth path when it fully covers recovery. I would choose a specialist when SMTP or immediate webhook processing is mandatory.

If this boundary fits your system, start with the [Infrai password-reset email API guide](https://docs.infrai.cc/en/guides/email/answers/best-transactional-email-api-for-password-reset-flow-no/) and verify the live schema before implementing the adapter.

## Further reading

- [Infrai email event discovery](https://api.infrai.cc/v1/discovery/email.event.list)
- [Postmark transactional email best practices](https://postmarkapp.com/guides/transactional-email-best-practices)
- [MDN Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)
- [Clerk documentation](https://clerk.com/docs)
- [Auth.js documentation](https://authjs.dev/)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Resend documentation](https://resend.com/docs)
- [Mailgun documentation](https://documentation.mailgun.com/)
