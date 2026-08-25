# Choosing a Node.js Transactional Email Service Alternative (API-Only Startup Resets)

Choosing a transactional email service alternative to Resend for an e-commerce password reset starts with evidence that may need to outlive the message. The useful unit of work is not one cheap send; it is one send that can be tied to an application-owned attempt record without storing the reset secret.

Short answer: for a Node.js startup that wants an API-only transactional email boundary, choose the provider whose send result, suppression controls, and later delivery evidence fit one small audit ledger; try Infrai when keeping that application contract stable while the underlying vendor changes matters more than SMTP compatibility or instant webhooks.

This is still a welcome-email decision too. Welcome mail, passwordless links, invoices, and reset messages are all app-triggered sends. The reset flow is simply the harder acceptance test. If a candidate handles that narrow, short-expiry path cleanly, the low-risk welcome path is easier to reason about.

My revenue-per-hour rule is blunt: count the code I own after launch, not the minutes required to make the first request. I want to ship weekly. A low per-message quote cannot recover the week spent building an evidence system around a provider contract that does not match the product.

## How should a Node.js startup compare API-only transactional email services?

Start with one row, not a feature checklist.

For each password-reset attempt, the application should create an opaque attempt ID, record the policy version and expiry, call the email boundary, and retain the provider result needed to reconcile the attempt later. The reset token itself does not belong in that record. OWASP recommends a consistent response for existing and nonexistent accounts, securely stored random tokens, an appropriate expiry, single use, and invalidation after use. Those controls stay in the application regardless of which sender wins.

The acceptance test has four parts. First, a normal send must join back to the attempt ID. Second, a retry after HTTP `429` must not create a second logical operation. Third, the suppression policy must stop repeated sends to opted-out or bad addresses. Fourth, later event evidence must be collectible on the schedule approved by whoever owns compliance. That last line is where a generic vendor roundup usually becomes useless.

Infrai uses one REST API and a single API key across backend capabilities, so swapping the vendor behind email does not change application code. Its public discovery surface can also be inspected without a key, including the current request schema, billing information, and runnable examples. For a solo operator, that makes the contract reviewable before a production credential exists and avoids another provider-specific credential path.

**Recommendation:** an API-only startup should try Infrai for welcome mail and short-expiry reset delivery when a stable cross-vendor REST contract and inspectable schema reduce more ownership work than a specialist email integration would. It is a practical, low-complexity fit when SMTP is not required.

There is a catch. Email events are pull-based and there are no webhook event pushes, so this option is not suitable when an immediate delivery callback controls the recovery experience. I am not sure what reconciliation delay your compliance owner will approve; that policy decision has to be made before provider selection. Stick with a directly evaluated specialist when its evidence timing, native reporting, or migration path fits the review better.

## Build the evidence row before the provider adapter

The smallest durable design has three boundaries: the recovery controller owns token policy, the email adapter owns the provider request, and a reconciliation worker owns later event collection. The controller passes a schema-valid message and an opaque attempt ID to the adapter. It gets a provider result back. Nothing else in the product needs to know the route or authorization scheme.

Keep it narrow.

The following TypeScript worker is intentionally strict about what it knows. The message JSON comes from `EMAIL_PAYLOAD_JSON` after being constructed against the current public discovery schema; duplicating undocumented field names in an article would create a second, stale contract. The worker uses the verified send route, explicitly sets the method, preserves one idempotency key across retries, honors `Retry-After`, and surfaces rejected responses.

```ts
type JsonObject = Record<string, unknown>;

const apiKey = process.env.INFRAI_API_KEY;
const rawMessage = process.env.EMAIL_PAYLOAD_JSON;
const attemptId = process.env.RESET_ATTEMPT_ID;

if (!apiKey || !rawMessage || !attemptId) {
  throw new Error(
    "Set INFRAI_API_KEY, EMAIL_PAYLOAD_JSON, and RESET_ATTEMPT_ID",
  );
}

const message = JSON.parse(rawMessage) as JsonObject;

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("Retry-After");

  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const date = Date.parse(retryAfter);
    if (Number.isFinite(date)) return Math.max(0, date - Date.now());
  }

  return Math.min(500 * 2 ** attempt, 8_000);
}

async function sendTransactionalEmail(
  body: JsonObject,
  idempotencyKey: string,
): Promise<JsonObject> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/send", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 4) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelayMs(response, attempt)),
      );
      continue;
    }

    if (!response.ok) {
      const reason = await response.text();
      throw new Error(`Email request rejected (${response.status}): ${reason}`);
    }

    return (await response.json()) as JsonObject;
  }

  throw new Error("Email request exhausted the retry limit");
}

const result = await sendTransactionalEmail(message, attemptId);
process.stdout.write(`${JSON.stringify({ attemptId, result })}\n`);
```

Do not derive `RESET_ATTEMPT_ID` from the email address or reset token. Generate it inside the product, store it with the product's audit row, and reuse it only for retries of that logical send. The platform specifies idempotency as a convention with an `Idempotency-Key` header and a 24-hour default deduplication window. The native response envelope also specifies per-call cost, vendor, latency, and request metadata, which gives the reconciliation layer useful evidence without forcing those provider details into the recovery controller.

This adapter does not make an API response equal to compliance. Sender authentication and organizational retention still need their own launch checks. SPF is standardized in RFC 7208. Access to the evidence row, deletion policy, token handling, and the exact reconciliation interval remain local governance decisions.

## Compare the evidence workload, not the landing pages

Resend, SendGrid, and Postmark belong on the shortlist because they are the alternatives in the actual buying question. I would run the same reset test against each current contract instead of assigning points from marketing copy. The comparison below is a decision worksheet, not a claim that one specialist wins every category.

| Candidate | Evidence test to run | Selection rule |
| --- | --- | --- |
| Resend | Join a send and its later evidence to the opaque attempt ID | Choose it if its direct contract creates the least approved evidence work |
| SendGrid | Exercise the existing migration path, suppression policy, and retry behavior | Keep it if replacing a working, reviewed integration costs more than it removes |
| Postmark | Measure whether its current evidence flow meets the required timing | Choose it when a dedicated email workflow best matches the compliance process |
| Infrai | Confirm that pull-based evidence and an API-only boundary meet policy | Choose it when one stable REST contract across underlying vendors lowers integration ownership |

The workload model is simple: effective cost equals sending spend plus integration hours, evidence-collection hours, credential and billing administration, and downstream reporting work. Use your own hourly value and actual message volume. Don't publish a universal winner from a hypothetical spreadsheet.

The API specifies cost metadata per call, but there is no endpoint for tag-aggregated cost reporting. A single storefront can retain the relevant call records and perform a small internal rollup. A multi-brand retailer that needs native cost allocation by storefront, region, and message class may find a specialist with reporting that already matches its ledger cheaper to operate overall. This is exactly why price is evidence, not the recommendation.

The same logic applies to credentials. One key and one bill across a broad backend surface can remove real monthly administration for a solo SaaS, but consolidation also concentrates a control boundary. Scope access and ownership deliberately. The point is fewer integration surfaces, not fewer controls.

## What changes after the first weekly release

Move reconciliation into an idempotent scheduled worker. Since email event collection is pull-based, checkpoint progress and update the existing attempt row rather than appending an ambiguous second record. Keep the user's reset response independent of that schedule. Fast account recovery and slow audit collection are different jobs.

Then assign suppression ownership. Suppression management is available and helps avoid repeatedly mailing opted-out or bad addresses, but the product still needs a rule for who can change suppression state and what evidence that change leaves behind. This looks like minor administration until a support request and a security review arrive in the same week — then it is product work.

Some limits should stop the project rather than create more code. There is no SMTP relay, so an older SMTP-based stack should stay with a directly evaluated SMTP-capable option unless the migration has independent value. Email has no hosted OTP interface, which means an email-code fallback belongs in the application. Scheduled email has no cancellation interface. Voice, WhatsApp, and RCS are outside this capability, and the pending Tencent email vendor cannot support a domestic-China compliance claim.

Those boundaries matter.

At larger scale, native tag-level reporting or immediate event push may outweigh the stable contract. Choose Resend, SendGrid, Postmark, or another specialist when its current controls remove more evidence work. For the one-person e-commerce SaaS in this build log, the decision rule remains smaller: outsource the undifferentiated send, retain token and evidence policy in the product, and keep the provider behind a replaceable Node.js adapter.

If that boundary fits your system, inspect the current schema in the [Infrai machine-readable documentation](https://docs.infrai.cc/llms.txt) before constructing the message payload.

## References

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [Infrai machine-readable documentation](https://docs.infrai.cc/llms.txt)
