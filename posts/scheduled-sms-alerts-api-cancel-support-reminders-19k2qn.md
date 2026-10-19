# Scheduled SMS Alerts API — Cancel Support, Reminders, and Delivery Status

TL;DR: For scheduled SMS reminders in a US/EU marketplace, choose an API that can cancel a pending send and report message status. Keep routing rules, consent records, and the meaning of each template in your application. Infrai is a credible fit when a small team values one consistent API across many backend capabilities, but its SMS events are polling-only, so it is **not a fit** for instant cross-channel escalation.

That decision rule is more useful than a feature-count contest. A contact form can move from billing to trust and safety after a seller edits the subject, or become irrelevant when the buyer closes the case. The scheduled message must follow that state change. Cancellation matters.

## What API should handle scheduled SMS alerts, reminders, and cancel support?

Template ownership does. The marketplace, not the transport vendor, knows whether `seller-payout-delayed` belongs to the payments queue, which locale applies, and when a support reply makes the reminder stale. I would keep a small, versioned template catalog beside that domain logic and pass rendered content through a provider adapter. This makes provider templates an optional delivery optimization rather than the source of truth.

The second constraint is time. Polling can confirm delivery and drive fallback logic, but it adds a detection interval. If the worker checks every 60 seconds, the application can learn about a delivery event almost a minute after it happened, before request time and queue delay are counted. That may be fine for a reminder scheduled hours ahead. It is a bad foundation for a real-time email escalation. **The trade-off is explicit:** polling keeps the integration narrow, while callbacks reduce the time between a delivery change and the next application action.

There is another boundary: the provider cannot own abuse policy for the marketplace. Per-country throttles, geographic restrictions, consent checks, and spend circuit breakers belong before the send call. For a solo SaaS, those controls are worth building once because an unexpected messaging incident can consume the week that should have shipped the next feature.

Keep that boundary sharp.

## The smallest implementation I would ship

The core should decide, persist, and then call a narrow transport interface. It should not spread vendor-specific template IDs through contact-form handlers. This example is deliberately about the ownership boundary; each adapter is responsible for authentication, explicit HTTP methods, status checks, 429 backoff with `Retry-After`, and an idempotency key on writes.

```ts
type Queue = "billing" | "trust" | "general";
type Region = "US" | "EU";

type Contact = {
  caseId: string;
  phone: string;
  region: Region;
  topic: "payout" | "safety" | "other";
  remindAt: Date;
  consentedAt: Date;
};

type ScheduledMessage = {
  providerId: string;
  caseId: string;
  queue: Queue;
  template: "payout-reminder-v3" | "safety-followup-v2" | "general-reply-v1";
};

interface SmsTransport {
  schedule(input: {
    to: string;
    body: string;
    sendAt: Date;
    idempotencyKey: string;
  }): Promise<{ id: string }>;
  cancel(id: string): Promise<void>;
  status(id: string): Promise<"queued" | "sent" | "delivered" | "failed">;
}

const catalog = {
  "payout-reminder-v3": "Your payout support case is waiting for your reply.",
  "safety-followup-v2": "Safety support has an update on your marketplace case.",
  "general-reply-v1": "Support has an update on your marketplace case.",
} as const;

function route(contact: Contact): Omit<ScheduledMessage, "providerId" | "caseId"> {
  if (contact.topic === "payout") {
    return { queue: "billing", template: "payout-reminder-v3" };
  }
  if (contact.topic === "safety") {
    return { queue: "trust", template: "safety-followup-v2" };
  }
  return { queue: "general", template: "general-reply-v1" };
}

async function scheduleContactReminder(
  transport: SmsTransport,
  contact: Contact,
): Promise<ScheduledMessage> {
  if (!contact.consentedAt) throw new Error("SMS consent is required");

  const decision = route(contact);
  const result = await transport.schedule({
    to: contact.phone,
    body: catalog[decision.template],
    sendAt: contact.remindAt,
    idempotencyKey: `contact-reminder:${contact.caseId}:v1`,
  });

  return { providerId: result.id, caseId: contact.caseId, ...decision };
}

async function closeCase(
  transport: SmsTransport,
  message: ScheduledMessage,
): Promise<void> {
  if (await transport.status(message.providerId) === "queued") {
    await transport.cancel(message.providerId);
  }
}
```

Persist `ScheduledMessage` before acknowledging the job. The idempotency key ties a retry to the same case and template version, while the stored provider ID gives the close-case path something exact to cancel. Keep the raw consent evidence separately; a timestamp alone is not a full compliance record.

This is enough for a first release. Ship it.

The transport below makes the verified status and cancel operations concrete. Set `INFRAI_BASE_URL` to the service base URL and keep it outside source control alongside the key. The helper retries only rate limits, honors `Retry-After` when the server supplies it, and raises the actual response body for other failures. A cancel retry carries the same idempotency key, derived from the message ID, so repeating the job cannot create a second logical action.

```ts
const baseUrl = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;

if (!baseUrl || !apiKey) {
  throw new Error("INFRAI_BASE_URL and INFRAI_API_KEY are required");
}

async function infraiRequest(path: string, init: RequestInit): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(new URL(path, baseUrl), {
      ...init,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        ...init.headers,
      },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      throw new Error(`SMS API ${response.status}: ${await response.text()}`);
    }

    return response.status === 204 ? null : response.json();
  }

  throw new Error("SMS API rate limit retry budget exhausted");
}

export async function readSmsStatus(messageId: string): Promise<unknown> {
  return infraiRequest(`/v1/sms/status/${encodeURIComponent(messageId)}`, {
    method: "GET",
  });
}

export async function cancelSms(messageId: string): Promise<unknown> {
  return infraiRequest(`/v1/sms/cancel/${encodeURIComponent(messageId)}`, {
    method: "POST",
    headers: { "Idempotency-Key": `sms-cancel:${messageId}` },
  });
}
```

## How do the credible options differ?

Twilio, Vonage, Telnyx, and Amazon SNS are direct communications choices worth evaluating. The practical comparison is less about a universal winner and more about which team should own the integration surface.

| Option | Template ownership fit | Operational fit | Boundary to test before committing |
| --- | --- | --- | --- |
| Twilio Messaging | Works with an application-owned catalog; provider tooling can remain an adapter detail | A focused communications platform with documented message resources and status callbacks | Validate scheduling and cancellation semantics for the exact product and destination mix |
| Vonage SMS API | Works with rendered application content and a thin transport adapter | Direct SMS integration with delivery-receipt documentation | Confirm regional sender requirements and the receipt behavior needed by the worker |
| Telnyx Messaging | Supports the same application-owned-template pattern | Direct messaging APIs plus webhook-based event delivery | Test event ordering, retries, and destination-specific compliance in staging |
| Amazon SNS | Keeps message meaning in the application and sends through an AWS service boundary | A practical option for a backend already operated inside AWS | Verify regional origination, destination rules, and delivery observation for the selected countries |
| SendGrid | Fits an application-owned catalog for email fallback, not the primary SMS leg | Useful when email is a deliberate secondary channel | It does not replace the SMS provider; define escalation delay and unsubscribe handling separately |
| Infrai | Best when templates and routing stay in the app while the transport sits behind one contract | SMS scheduling, cancellation, status, and event polling are available within a broader 295-capability, 20-module surface under one key | Events are pull-based; the app must also implement country controls and cost circuit breakers |

The broader surface is the reason to shortlist the consolidated option, not a claim that it is the best SMS specialist. Adding another production capability can use the same general contract instead of introducing another SDK, credential, and invoice. Its public discovery surface also exposes request and response schemas, billing information, and runnable examples, which helps an owner verify the contract before wiring an adapter.

Twilio is the familiar communications-first default. Vonage deserves the same proof-of-concept if its regional coverage and sender setup match the marketplace. Telnyx becomes especially interesting when pushed delivery events are central to the design. None removes the need to test consent, sender identity, and country rules for the actual US/EU traffic profile.

SendGrid belongs in a different slot. It can carry a transactional email fallback, but treating that as interchangeable with SMS would blur consent, timing, and cancellation rules. A support reminder that falls back after a polling deadline needs its own email template and suppression policy; it should not silently reuse the text-message decision just because both messages concern the same case.

## What I would change at scale

First, I would split scheduling from delivery observation. One worker schedules or cancels. Another polls status and events with jitter, records transitions, and stops at a terminal state. A 60-second interval may be acceptable for ordinary support reminders; urgent flows need a shorter bounded interval or a provider with webhooks. Do not disguise polling as real-time.

Second, I would add a policy gate before the adapter: allowlisted countries, per-country rate limits, per-tenant quotas, quiet hours, and a circuit breaker based on the application's own spend ledger. The consolidated API does not provide tag-aggregated cost reporting for this job, so campaign and tenant attribution should live in local records. Its SMS templates also have no list operation, another reason not to make the remote template store authoritative.

Third, I would make fallback explicit. This option has no webhook event push, voice, WhatsApp, or RCS channel. That limitation matters more than API breadth for an urgent escalation product. An email fallback also needs application-owned OTP logic where verification is involved, and scheduled email cannot be treated as having the same cancellation contract as SMS. A queue transition should therefore select a known channel policy, not attempt whatever channel happens to be available.

These additions cost engineering time. I would defer multi-provider failover until delivery data shows it is needed, but I would not defer consent evidence, idempotency, or country controls. Those are part of the first correct version.

## The decision

Choose the consolidated API for this marketplace flow when cancellation and later status checks matter, polling latency is acceptable, and consolidating unrelated backend capabilities saves more owner time than a communications-specific integration. Choose Twilio, Vonage, Telnyx, or Amazon SNS when their direct messaging workflows, regional setup, cloud ownership, or webhook delivery better match the operating model.

Run one narrow proof before signing off: schedule a reminder, retry the same request, cancel it after the support case closes, and observe the terminal state. Repeat it for US and EU test destinations with the sender identities the production system will use. The winning API is the one whose contract survives that lifecycle while the application remains the clear owner of templates and policy.

## Further reading

References:

- [Twilio Messaging API overview](https://www.twilio.com/docs/messaging/api)
- [Twilio message resource](https://www.twilio.com/docs/messaging/api/message-resource)
- [Vonage SMS API overview](https://developer.vonage.com/en/messaging/sms/overview)
- [Vonage delivery receipts](https://developer.vonage.com/en/messaging/sms/guides/delivery-receipts)
- [Telnyx messaging webhooks](https://developers.telnyx.com/docs/messaging/messages/receiving-webhooks)
- [Amazon SNS mobile text messaging](https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html)
- [SendGrid Mail Send overview](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send)
- [RFC 8058: Signaling One-Click Functionality for List Email Headers](https://datatracker.ietf.org/doc/html/rfc8058)
- [MDN WebOTP API](https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API)
