# Password Reset Email Timeout: Classifying Hanging API Requests Before Delayed Sends

Short answer: make the business event your evidence authority, give the outbound `fetch` or Axios call a deadline, and classify a timeout as unknown until later evidence resolves it. For a marketplace, that means a seller's new-order notice and a password reset email can use the same audit model without treating a hanging API request as proof of failure.

| Evidence authority | What a timeout means | Audit value | Operating cost | Best fit |
| --- | --- | --- | --- | --- |
| Business event plus notification ledger | Acceptance is still unknown | One chronology under your control | Moderate | Compliance evidence matters |
| Provider receipt and event history | Local request outcome is incomplete | Useful when export and retention fit policy | Low | Outsourced operations matter most |
| Application log line | The call did not finish locally | Weak and easy to fragment | Low until an investigation | Non-sensitive, disposable messages |

Choose the first model for seller order alerts and password resets. It answers the question compliance actually asks: what did the application intend, what did it observe, and why did it send again? It also fits one-person SaaS economics. Keep the small policy layer in your code, outsource the undifferentiated transport, and spend revenue-producing hours on the marketplace.

Don't retry yet.

## Which evidence can prove a password reset email API request timed out?

A client deadline proves one narrow fact: your process stopped waiting. It does not prove that the transactional email provider rejected the message, and it does not prove that the message was delayed or delivered. This distinction is the center of the design. `fetch` and Axios are messengers here; neither library can turn a missing response into a remote delivery fact.

Start by writing the claims your system must support. For a marketplace seller notification, a useful set is: order `ord_4821` created notification intent `ntf_913`; attempt `att_01` began under policy revision `notify-v4`; the client deadline elapsed; a transport acknowledgment later matched that attempt; and the final channel event was attached without rewriting prior history. The reset path uses the same fields, but stores neither the reset token nor a secret-bearing message body. Keep a recipient hash for correlation, plus the template revision, purpose, timestamps, actor, and retention class.

This is deliberately more precise than a single `status` column. If one mutable row moves from `pending` to `failed` to `delivered`, an investigator can't tell which observation caused each change. An append-only transition trail preserves that causality. It also separates three clocks that teams often blur together: the business event time, the local transport deadline, and the remote event time. A seller may receive a new-order alert after the HTTP caller has stopped waiting. The audit record should say exactly that — no drama, no invented certainty.

The practical test is simple. Ask support to explain one notification without opening raw process logs. If they need to correlate timestamps across containers by hand, the application doesn't yet own enough evidence.

I'm not sure every provider offers the same receipt lookup, event retention, or export contract. Verify those points during selection and record the result in the policy. Your mileage may vary by recipient domain, so delivery-time alert thresholds also need evidence from your own traffic rather than a universal number.

## Model uncertainty before choosing a retry

Use explicit states instead of mapping every exception to `failed`. The important branch occurs after intent exists but before acceptance is known.

| State | What is known | Permitted next action |
| --- | --- | --- |
| `ready` | Intent is durable; no attempt has started | Start one attempt |
| `sending` | Transport work started | Wait until the client deadline |
| `uncertain` | The deadline elapsed without acknowledgment | Reconcile; do not automatically resend |
| `accepted` | A transport receipt is attached | Wait for later channel evidence |
| `rejected` | A definite rejection is recorded | Correct the cause or apply retry policy |
| `delivered` | A delivery event is attached | Close the notification |

Unknown is real state.

This state machine changes how incidents are diagnosed. First find the business intent, then inspect its attempts, then reconcile receipts and channel events. Only after that should an operator decide whether another attempt is permitted. The order matters because an eager retry after an ambiguous timeout can create two legitimate sends, each accepted independently. For a password reset, that confuses the user; for a seller's new order, it can make a single purchase look like two operational events.

Classification belongs in policy, not in an Axios interceptor tucked away from the business record. A definite payload or authentication rejection can become `rejected`. A local deadline with no acknowledgment becomes `uncertain`. A later receipt can advance that same attempt to `accepted`. Preserve who initiated a manual retry and which policy revision allowed it. There is no honest exactly-once claim at an external message boundary, so make duplicate suppression and reconciliation visible application decisions.

Email purpose classification deserves the same care. RFC 8058 defines one-click unsubscribe signaling for list email. Don't paste that mechanism into account-recovery or transactional order messages without first deciding what kind of mail you are sending. If SMS is a fallback channel, test the rendered body: GSM-7 and UCS-2 have different character limits and therefore different segmentation behavior. A template edit can change the encoding and the number of segments even when the workflow code stays untouched.

## One TypeScript boundary for fetch, Axios, and delayed provider evidence

Keep deadline ownership above the transport adapters. Both adapters receive an `AbortSignal`; the orchestrator records observations and returns an explicit outcome. It does not silently issue attempt two.

```ts
type Notification = {
  notificationId: string;
  attemptId: string;
  kind: "password-reset" | "seller-new-order";
  recipientHash: string;
  templateRevision: string;
};

type Receipt = {
  providerMessageId: string;
  acceptedAt: string;
};

type SendOutcome =
  | { kind: "accepted"; receipt: Receipt }
  | { kind: "uncertain"; deadlineAt: string };

interface Transport {
  send(notification: Notification, signal: AbortSignal): Promise<Receipt>;
}

interface EvidenceStore {
  start(
    notification: Notification,
    startedAt: string,
    deadlineAt: string,
  ): Promise<void>;
  accept(attemptId: string, receipt: Receipt): Promise<void>;
  markUncertain(attemptId: string, observedAt: string): Promise<void>;
}

async function attemptOnce(
  transport: Transport,
  evidence: EvidenceStore,
  notification: Notification,
  timeoutMs: number,
): Promise<SendOutcome> {
  const controller = new AbortController();
  const startedAt = new Date();
  const deadlineAt = new Date(startedAt.getTime() + timeoutMs);
  const timer = setTimeout(() => controller.abort(), timeoutMs);

  await evidence.start(
    notification,
    startedAt.toISOString(),
    deadlineAt.toISOString(),
  );

  try {
    const receipt = await transport.send(notification, controller.signal);
    await evidence.accept(notification.attemptId, receipt);
    return { kind: "accepted", receipt };
  } catch (error: unknown) {
    if (controller.signal.aborted) {
      const observedAt = new Date().toISOString();
      await evidence.markUncertain(notification.attemptId, observedAt);
      return { kind: "uncertain", deadlineAt: deadlineAt.toISOString() };
    }

    throw error;
  } finally {
    clearTimeout(timer);
  }
}
```

The transport implementation may use native `fetch` or Axios, but its contract stays boring: one attempt in, one receipt out, with cancellation supplied by its caller. That makes provider migration smaller because retention, retry permission, and compliance vocabulary do not live inside a vendor adapter. It also makes tests fast. Use a controlled promise and a fake clock to cover acceptance before deadline, deadline before acknowledgment, definite rejection, a late receipt attached during reconciliation, and two workers contending for the same logical notification. No wall-clock sleeps.

Deployment needs an equally plain rule: persist intent before starting network work. A durable queue is useful when the notification must survive a web-process restart or a burst, but the queue does not replace the evidence model. It adds worker leases, redelivery, poison-message handling, and its own retention surface. Ship the ledger and state transitions as one thin vertical slice, exercise them with a seller order in staging, then add channel adapters. Weekly shipping is easier when the irreversible policy is separate from replaceable plumbing.

Observe gaps, not one blended latency percentile. Count intents without attempts, attempts that enter `uncertain`, accepted messages without terminal channel evidence, and duplicate attempts per logical notification. An incident view should place business intent, transport start, deadline, receipt, and later channel event on one timeline. That's enough to diagnose a delayed send without pretending the Node.js exception knows what happened downstream.

## When should the runner-up own the compliance trail?

Let provider receipts and event history be the primary record when notification evidence is low-risk, the provider's export and retention terms satisfy policy, and maintaining a ledger would steal more feature time than it returns. The catch is dependence on an external evidence schema and access window. This option is not suitable when your application must join notification history to order-level actor, policy, and retention decisions after provider data is no longer available.

Stick with a durable queue before transport when work must outlive the request, absorb bursts, or continue across deployments. Accept its operational surface on purpose. For modest traffic with prompt reconciliation, a bounded synchronous attempt can be the smaller choice; for sustained bursts, the queue is usually the more legible runner-up. Neither changes the meaning of a timeout.

Application logs alone are appropriate only when losing or duplicating the message has little consequence. Password resets and seller order notices don't fit that category. The decision isn't about which HTTP client has the nicer timeout option. It is about who can produce a coherent, retained explanation after the client stopped waiting.

## References

- RFC 8058, “Signaling One-Click Functionality for List Email Headers”: https://datatracker.ietf.org/doc/html/rfc8058
- Twilio, “What is the SMS character limit?”: https://www.twilio.com/docs/glossary/what-sms-character-limit

## Further reading

- https://datatracker.ietf.org/doc/html/rfc8058
- https://www.twilio.com/docs/glossary/what-sms-character-limit
