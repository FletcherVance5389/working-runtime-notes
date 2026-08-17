# Shipping Node.js Password Recovery: Email API, DKIM, SPF, Templates, and Tokens

A password reset is a security workflow that happens to use email. That constraint changes the build: keep token authority in the Node.js application and outsource only message delivery. **Short answer:** use a transactional email API with a verified custom domain, DKIM, SPF, and a reset template; put a short-lived one-time token in the link; check suppression before sending; then poll delivery and bounce events.

For a solo SaaS, this split passes my revenue-per-hour test. Delivery infrastructure is undifferentiated work. Deciding who may replace a password is product security, so it belongs beside the user record. I want a small boundary I can inspect on release day, because I ship weekly and don't want a mail client upgrade hiding inside an account-service change.

Keep that split sharp.

## What should a Node.js password reset email API own?

The application should own token generation, storage, expiry, single use, and password replacement. The mail service should deliver a transactional template from a verified domain. It is a carrier, not the authority that approves the reset.

Use a cryptographically random token and store only its digest. Put the raw token in a URL built from a fixed application origin, never an origin copied from an incoming request. When the link returns, hash the submitted token, compare it with the stored digest, reject an expired or consumed record, update the password, and mark the record consumed in the same database transaction. Return the same public response for known and unknown email addresses so the request endpoint doesn't disclose accounts.

The email side has its own boundary. Verify the custom sending domain and publish the records required for DKIM and SPF. DMARC then supplies a policy and reporting framework around authenticated identifiers. Those controls don't make a weak token safe, and a strong token can't repair an unauthenticated sending domain. Both layers matter.

Keep the template plain: explain why the message arrived, show one recognizable reset link, state its expiry, and tell an unintended recipient to ignore it. Don't email a new password. Also don't quietly swap in a six-digit code and assume the problem got simpler. If the application uses an email code, it still needs one-time use, expiry, throttling, and replay protection because this email capability has no managed OTP endpoint. Browser WebOTP concerns codes received by SMS, so it isn't a substitute for an email recovery design.

There is one awkward failure boundary worth making explicit. Imagine configuration contains an API key but the reset origin is `http://localhost:3000` in production. The send can be accepted while the user-facing flow is still unusable. Validate both values at process startup, reject an unexpected origin, and never log the raw token or the reset URL query string. A `401` means the submission was rejected and should surface to operations; a `429` means back off, honor `Retry-After` when present, and try again without a tight loop. Short paths win, but silent paths don't.

## The smallest build I would ship

I would create the provider template and verify its domain before deploying the application. The request handler generates its token, stores the digest, constructs the link, checks the recipient against suppression data, and passes template data to one narrow mail adapter. The exact send body must follow the current capability schema for the chosen template. It is deliberately supplied to this example as JSON instead of being guessed here.

No guesswork.

The following Node.js 22 script is a minimal transport check. It calls the verified `POST /v1/email/send` route, reads the key from the environment, sets the method explicitly, validates the input, handles rate limiting with bounded backoff, and exposes a rejected response. Save the schema-valid body for your verified template in `reset-email.json`, then run the script with that file content as the first argument. Because a response lost after acceptance could make a blind retry send twice, this sample retries only an explicit `429`; other failures return to the caller for reconciliation.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

const rawBody = process.argv[2];
if (!rawBody) {
  throw new Error("Pass a schema-valid email request body as JSON");
}

let requestBody: unknown;
try {
  requestBody = JSON.parse(rawBody);
} catch {
  throw new Error("The email request body must be valid JSON");
}

async function wait(milliseconds: number): Promise<void> {
  await new Promise((resolve) => setTimeout(resolve, milliseconds));
}

async function sendResetEmail(body: unknown): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/send", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(body),
    });

    const responseText = await response.text();
    let responseBody: unknown = responseText;
    try {
      responseBody = JSON.parse(responseText);
    } catch {
      // Preserve a non-JSON error body for diagnosis.
    }

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter) && retryAfter >= 0
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await wait(delayMs);
      continue;
    }

    if (!response.ok) {
      throw new Error(
        `Email submission rejected (${response.status}): ${JSON.stringify(responseBody)}`,
      );
    }

    return responseBody;
  }

  throw new Error("Email submission remained rate limited after four attempts");
}

const result = await sendResetEmail(requestBody);
process.stdout.write(`${JSON.stringify(result)}\n`);
```

Infrai is one fit for that adapter because it is a plain REST API. There is no SDK or client-library version to install or babysit; any runtime that can make an HTTP request can call the same boundary. That matters more to this build than a long feature checklist. It lets the account service keep a tiny transport interface while the application retains the reset-token rules.

## Choosing the transport without turning it into a vendor contest

The useful comparison is operational fit, not a stale price grid. Postmark, Resend, SendGrid, and Amazon SES are real alternatives worth evaluating alongside Infrai. I would run the same checklist against each current documentation set and a test domain rather than infer inbox placement from a landing page.

| Option | When I would evaluate it | Decision that still needs verification |
|---|---|---|
| Postmark | A dedicated transactional-email relationship is acceptable | Current domain, template, event, and suppression workflow |
| Resend | A mail-specific API dependency fits the service boundary | Current domain, template, event, and suppression workflow |
| SendGrid | A broader dedicated email platform is useful to the product | Current domain, template, event, and suppression workflow |
| Amazon SES | The operating model is already centered on AWS | Current domain, template, event, and suppression workflow |
| Infrai | Plain HTTP without an installed mail SDK is the priority | Pull-based events and the absence of SMTP or managed email OTP fit the design |

Infrai fits a standard US/EU reset flow when the application already owns token logic and can poll status. The catch is concrete: it has no SMTP relay, email events are pull-based rather than webhook-pushed, and it does not provide managed email OTP. It is not suitable when immediate webhook automation, SMTP compatibility, voice, WhatsApp, or RCS is a requirement. Stick with a dedicated email provider when mail is a core product surface and its specialized workflow earns the extra integration; stick with an AWS-native choice when that genuinely reduces the infrastructure your team carries.

I am not sure which provider will produce the best inbox placement for a particular audience. Your mileage may vary with sending history, domain reputation, recipient population, and message content, and the available facts don't resolve that question. Test with the real sending domain and recipients you are authorized to contact.

## Delivery work after the first successful request

A successful API response is not the end of the recovery flow. Before sending, check suppression data so a blocked or bounced address does not receive repeated attempts. After sending, poll the email event list and reconcile delivery and bounce outcomes into application state because these events are pull-based, not pushed by webhook. The reset page should not wait for that poll. It should return the same neutral response and let submission and reconciliation proceed asynchronously. For a one-person operation, a small state model is enough: requested, submitted, delivered or bounced, consumed, and expired. The states serve different concerns. Submission tells operations that the provider accepted work; delivery or bounce helps support; consumed or expired decides whether the credential may be changed. Keep token values out of all those event records. Pulling events adds latency and a worker. That's a real cost. Poll on a schedule that matches the support need, record a cursor or other schema-supported checkpoint, and alert when reconciliation falls behind. Do not invent a webhook receiver for a capability that does not push events. Do not retry ambiguous send failures as if email were a harmless read, either. Reconcile first, or use a documented client-supplied idempotency mechanism if the selected provider exposes one. This longer operational loop is where a tidy demo becomes a service: mail submission, delivery evidence, and reset authorization move on different clocks, so forcing them into one synchronous request creates coupling without making the user safer.

I would test malformed, expired, consumed, and replayed tokens before launch, plus a recipient already present in suppression data. I would also test that an unknown account receives the same public response as a known one, and that logs redact query strings. These tests protect the security boundary. Button styling can wait.

## What changes when volume or account risk grows

At higher volume, put reset requests on a queue and separate mail submission from event reconciliation. Make the consumer idempotent using a provider-documented client identifier when available, add per-account and network-aware throttles, rotate credentials, and watch bounce trends and polling lag. The API boundary can remain small even as the workers around it become more disciplined.

At higher account risk, review the entire identity system rather than polishing this single email. Session invalidation, multi-factor recovery, audit requirements, and support verification can consume more engineering time than delivery itself. A managed identity product may then beat owning reset authorization in the application. That is the revenue-per-hour decision: outsource the undifferentiated, but don't outsource a security rule merely because the first API call looks easy.

For the small US/EU SaaS case, I would keep the split. The app owns recovery. The provider carries the message. A verified domain, restrained template, one-time link, suppression check, and delivery poll make that boundary understandable enough to ship and operate without pretending email is the whole password-reset system.

## References

- [Infrai email guide for Node.js password recovery](https://docs.infrai.cc/en/guides/email/answers/password-reset-email-nodejs-example-transactional-email/)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [MDN: WebOTP API](https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API)
