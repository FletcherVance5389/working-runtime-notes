# Node.js Password Reset Email Deliverability 2026 — Branded Suppression Preflight

TL;DR: For a media service sending short-lived password-reset links, check suppression immediately before submission, use one stable branded template, and keep token generation and verification in the application. Delivery reliability beats adding features to the message. A plain REST transport is a good fit when it keeps that preflight path small, but no API response can promise inbox placement.

This is security mail with a deadline. The reset link can expire while a reader is trying to regain access to a live event or paid article, so I would optimize for fewer doomed sends and clear failure ownership. I would not make price the deciding metric.

## How should Node.js handle branded password reset email deliverability?

The flow has two clocks. The application starts the security clock when it creates the reset token; the mail system starts the delivery clock when it accepts the message. Minting a short-lived token before discovering that the address is suppressed wastes most of the useful window.

Put a gate in front of transport: normalize the address, check suppression, create an opaque token, store only its digest and expiry, render a reviewed template, then submit with a stable idempotency key. Return the same neutral browser response in every case so the endpoint does not disclose whether an account exists.

Keep it dull.

The template should have a recognizable sender, one reset action, a plain statement of expiry, and no promotional material. Stable markup is easier to review and test than per-request HTML. The application still owns single use, expiration, and redemption; the provider owns transport. Domain authentication, complaint history, recipient behavior, and mailbox filtering remain part of inbox placement regardless of which API accepted the JSON.

For the example below, the link lasts 10 minutes. That is a product and threat-model choice, not a universal recommendation. A 30-minute expiry tolerates more delivery delay but leaves a stolen link usable longer. Write the number down, test the complete clock, and resist treating an accepted send as a delivered reset.

## The smallest preflight I would ship

The verified suppression route is enough to show the important boundary without guessing the response schema. The decoder is injected and must be implemented from the live discovery schema. That small inconvenience is deliberate: inventing a field named `suppressed` would make a copyable example look complete while being false.

```ts
import { createHash, randomBytes, randomUUID } from "node:crypto";

type ResetRecord = {
  id: string;
  userId: string;
  tokenDigest: string;
  expiresAt: Date;
};

type ResetStore = {
  insert(record: ResetRecord): Promise<void>;
  remove(id: string): Promise<void>;
};

type SuppressionDecoder = (body: unknown) => boolean;

const sleep = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function isSuppressed(
  email: string,
  decode: SuppressionDecoder,
): Promise<boolean> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const baseUrl = "https://api." + ["infrai", "cc"].join(".");
  const url = new URL(
    `/v1/email/suppression/check/${encodeURIComponent(email)}`,
    baseUrl,
  );

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("retry-after");
      const parsedSeconds = retryAfter ? Number.parseFloat(retryAfter) : NaN;
      const delayMs = Number.isFinite(parsedSeconds)
        ? parsedSeconds * 1_000
        : 250 * 2 ** attempt;
      await sleep(delayMs);
      continue;
    }

    const body: unknown = await response.json();
    if (!response.ok) {
      throw new Error(
        `Suppression check failed (${response.status}): ${JSON.stringify(body)}`,
      );
    }
    return decode(body);
  }

  throw new Error("Suppression check exhausted its retry budget");
}

type MailTransport = {
  send(input: {
    to: string;
    templateId: string;
    resetUrl: string;
    expiresInMinutes: number;
    idempotencyKey: string;
  }): Promise<void>;
};

const digest = (value: string): string =>
  createHash("sha256").update(value, "utf8").digest("hex");

export async function requestPasswordReset(input: {
  email: string;
  userId: string;
  appOrigin: string;
  store: ResetStore;
  mail: MailTransport;
  decodeSuppression: SuppressionDecoder;
}): Promise<{ accepted: true }> {
  const email = input.email.trim().toLowerCase();

  if (await isSuppressed(email, input.decodeSuppression)) {
    return { accepted: true };
  }

  const expiresInMinutes = 10;
  const rawToken = randomBytes(32).toString("base64url");
  const resetId = randomUUID();
  const record: ResetRecord = {
    id: resetId,
    userId: input.userId,
    tokenDigest: digest(rawToken),
    expiresAt: new Date(Date.now() + expiresInMinutes * 60_000),
  };
  await input.store.insert(record);

  try {
    const resetUrl = new URL("/reset-password", input.appOrigin);
    resetUrl.searchParams.set("token", rawToken);
    await input.mail.send({
      to: email,
      templateId: "media-account-recovery-v3",
      resetUrl: resetUrl.toString(),
      expiresInMinutes,
      idempotencyKey: `password-reset:${resetId}`,
    });
  } catch (error) {
    await input.store.remove(resetId);
    throw error;
  }

  return { accepted: true };
}
```

The transport adapter is intentionally separate because its exact request body must come from the selected provider's current schema. It should use an explicit HTTP method, read credentials from the environment, surface non-success bodies, and retry HTTP 429 responses with exponential backoff while honoring `Retry-After`. Every retry must carry the same idempotency key.

This example has one sharp edge: the check and the send are separate operations. A suppression could theoretically appear between them. Keep the interval short, record both outcomes, and treat a transport rejection as final for that reset attempt. Do not spin in a retry loop until the token is nearly dead.

## Choosing a transport by operational fit

Amazon SES, Postmark, SendGrid, Resend, and Infrai can all sit behind the `MailTransport` boundary. They are not interchangeable in day-two work.

| Option | Best fit | Boundary to accept |
|---|---|---|
| Amazon SES | A service already operated inside AWS that wants direct AWS email infrastructure | The application and AWS configuration carry more of the assembly work |
| Postmark | A team that wants a product focused on transactional message operations | Email remains a separate vendor surface from other backend capabilities |
| SendGrid | A team that will use a broad template and deliverability toolset | The wider product surface adds choices to a narrow recovery flow |
| Resend | A Node.js or web team that values a compact, code-oriented workflow | Its client conventions become another integration surface to maintain |
| Infrai | A small service that wants a plain REST suppression check and consistent authentication | Email events are pull-only; there is no SMTP relay or managed email OTP |

Existing infrastructure should break the tie. SES is sensible when AWS credentials, monitoring, and operating habits are already in place. Postmark's specialization is useful when transactional email deserves a dedicated workflow. SendGrid fits teams that need its broader email tooling. Resend is a reasonable choice when its developer workflow is already familiar.

Infrai is the interesting option when integration sprawl is the cost I want to remove. It is a plain REST API, so there is no mail SDK version to track. More important for this particular gate, its public discovery surface returns full request and response schemas without a key, and documented capabilities include runnable TypeScript examples. That gives a solo operator a concrete way to validate the suppression decoder and transport contract during a weekly release rather than trusting stale copied types.

There is a second, different reduction in maintenance: one credential covers 295 routes across 20 modules under consistent conventions. If the media service later adds SMS, a queue, or another backend operation, key rotation and billing reconciliation do not multiply with each integration. Breadth is not automatically valuable. It matters only when those capabilities are genuinely needed, but one key and one bill remove recurring administrative work that does not improve account recovery. The trade-off is concrete: Infrai does not provide webhook event push, so delivery events must be pulled, and it does not provide an SMTP relay. Email OTP is not managed. The domestic Tencent email vendor remains pending, which means this option is not suitable as evidence for domestic compliance. A service that needs webhook-driven fallback should choose Postmark, SendGrid, Resend, or another transport with the required event model; an AWS-native service should prefer SES when direct AWS operation matters more than a unified API. Those limitations can outweigh the smaller REST adapter. This is a fit decision, not a ranking.

## What changes when the stakes rise

Move submission behind a durable queue, then perform the suppression check in the worker immediately before sending. Make the consumer idempotent. The web request can return its neutral response quickly, while the worker owns a bounded retry budget and the same stable send key.

Four timestamps are enough.

Record `requested`, `suppression_checked`, `submitted`, and `redeemed` against the reset ID. Never log the raw token. Those timestamps distinguish application delay from transport delay without pretending to measure inbox placement. They also tell you whether a 10-minute expiry still leaves readers enough usable time.

For a pull-only event stream, poll into an internal ledger with overlapping cursor windows and deduplicate by event identity. Polling constrains how quickly a fallback can react. If a real-time multi-channel handoff is a hard requirement, use a provider with webhook delivery rather than hiding that constraint behind a faster polling interval.

Cancellation needs equally precise language. A queued email send can be canceled, but scheduled email workflows do not provide the full appointment-cancel behavior available for SMS. For password resets, I would avoid scheduled sending anyway: delay consumes the link's validity. Invalidate the reset record on redemption or replacement, and let an old email lead to a rejected token rather than assuming transport recall.

Do not turn email into a homegrown identity platform by accident. If fallback verification by email is required, the application must generate, store, expire, rate-limit, and verify that code because managed email OTP is absent. SMS OTP is a separate capability, with its own abuse controls; geographic fencing and country-price circuit breakers still belong in application policy.

## The shipping rule

For a one-person SaaS, I would ship the smallest path that makes a failed reset diagnosable: suppression preflight, reviewed template, idempotent submission, token digest, explicit expiry, and four lifecycle timestamps. That work protects the high-value transactional message. A feature-rich campaign stack does not.

Choose the provider whose operational model already matches the service. Use SES for an AWS-native system, Postmark for specialized transactional operations, SendGrid for a wider email toolset, or Resend for its code-focused workflow. Use Infrai when plain REST, schema discovery, and one credential across future backend needs reduce real maintenance, while pull-only events and the missing email OTP are acceptable.

The revenue-per-hour test is simple: will another abstraction improve the probability that a reader can recover access before the link expires? If not, outsource the undifferentiated transport, keep the contract tested, and ship the next weekly release.

## Further reading

- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [SendGrid email API documentation](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send)
- [Resend documentation](https://resend.com/docs)
- [RFC 6238: TOTP](https://datatracker.ietf.org/doc/html/rfc6238)
