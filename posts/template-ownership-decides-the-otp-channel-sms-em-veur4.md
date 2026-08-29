# Template Ownership Decides the OTP Channel: SMS, Email, 2FA Latency and Audit Trails

Pick the channel whose message body you can still reproduce, byte for byte, two years after you sent it. In a property-management SaaS that pushes 2FA login codes to staff and statutory compliance notices to residents through the same messaging stack, template ownership is the axis that quietly settles the other three arguments — deliverability, latency, and what a delivery record can prove in a dispute. SMS is usually faster. Email carries better evidence. Whoever owns the template decides which of those two you actually get to keep.

| Option | Who owns the message body | What the record proves | Where it hurts |
|---|---|---|---|
| Short code / OTP sent from a provider-owned template | The provider and the carrier registry | Carrier delivery receipt to a handset, no read proof | Copy edits go through re-registration, on someone else's clock |
| SMS code from a template you author and register | You | Same carrier receipt, plus your own version history | You carry campaign registration and per-country sender rules |
| Email code rendered from a template in your repo | You | Transport-level delivery status, plus your render history | Inbox placement and privacy proxies blur what "delivered" means |
| TOTP app or passkey, no message at all | Nobody — nothing is sent | No delivery to prove, and none needed | Enrollment and account recovery become your problem |

My rule for this system: own the template for anything that might end up as evidence, and treat the login OTP as the one place where renting a template is acceptable. A compliance notice and a login code look like the same API call. They are not the same artifact. One needs to survive a hearing; the other needs to arrive in ten seconds and then be forgotten.

## Should a property-management SaaS send 2FA login codes over SMS or email?

Send the primary login code over SMS, keep email as a second factor path the application fully owns, and get both off the critical path for staff accounts within a release cycle or two.

That last clause matters more than the channel argument. NIST's digital identity guidelines classify out-of-band authentication over the public switched telephone network — SMS and voice — as a restricted authenticator: you're expected to assess the risk, tell users about it, and offer an alternative that isn't restricted. So the honest answer to "SMS or email for 2FA" is that both are transitional. A TOTP authenticator built on RFC 6238, with its 30-second step, or a passkey, ends the deliverability conversation entirely because nothing has to be delivered.

Until every property manager on your platform has enrolled one, you're shipping codes.

Latency is where SMS earns its place. A carrier-delivered code typically lands while the user is still looking at the login form; email goes through queueing, greylisting and retry logic on the receiving side, and a retry schedule measured in minutes is a normal, correct SMTP behavior rather than a failure. A login code with a 10-minute expiry can and does expire in transit. That's not a deliverability bug — it's a queue doing its job on a message class that has no business being latency-sensitive.

Security cuts the other way, though. An SMS code is bound to a phone number, and phone numbers move; the account-recovery path around a lost number is where most real 2FA compromises start. Email codes inherit whatever protection the mailbox has, which in this industry is frequently a shared `office@` or `leasing@` inbox that four people read. Neither channel is strong. Both are better than a password alone.

## Who owns the template, and how long that data must live

For application-to-person SMS in the US, the campaign registry model means the sample message bodies are part of a filing, not part of your codebase. That's the trade-off people underestimate. It's fine when the copy is stable — a six-digit code and a brand name rarely change. It's painful for compliance notices, where the body carries statutory language that varies by state, by notice type, and occasionally by a court decision handed down last quarter. Localization for EU recipients multiplies the same problem: alphanumeric sender IDs and message content are governed per country, so a template that ships in one market may need registration or rework in the next. If the copy is going to change under legal pressure, the template belongs in your repository, under version control, rendered by your code, with the rendered output hashed and stored next to the send record. If the copy will never change, renting the template is a fair way to buy someone else's carrier relationships.

Retention is the other half of ownership. A rented template leaves the rendered body on someone else's infrastructure under their deletion schedule, while resident phone numbers and unit addresses inside that body are personal data with a defensible retention period of their own — and for EU residents, a storage region you may have to name. Owning the render step lets you keep the notice archive for as long as the statute requires and drop the transport copy early.

Template ownership also decides your test story. A template you own can be snapshot-tested in CI, rendered into a fixture, diffed on every pull request, and rolled back like any other code. A template that lives in a provider console is tested by sending real messages and reading them on a real phone — which is a manual step, and manual steps in the notice path are where a solo operator's week disappears.

I'd rather spend that week on features.

## What a delivery receipt proves, and how to measure it

Delivery evidence for email is defined by the DSN format in RFC 3464: it reports what happened between mail transfer agents. Delivered means the receiving system accepted the message for the mailbox. It does not mean a person opened it, and it never did.

Open tracking is worse than useless here, and Apple's Mail Privacy Protection is the reason. When it's on, remote content is fetched through a proxy regardless of whether the recipient read anything, so an "open" event is evidence of a proxy, not of a resident. Any compliance dashboard that counts opens as receipt is reporting a number it cannot defend, and any risk engine that treats an OTP email open as a security signal is reading noise. Delete the pixel from the notice template. Keep it out of the login template too.

Domain authentication is the part that genuinely moves deliverability. DMARC, specified in RFC 7489, ties SPF and DKIM results to the visible From domain and gives you aggregate reports from receivers — which is the only cheap, standards-based visibility you get into how your notice traffic is being judged at the border. A management company sending from a domain with no DMARC policy, through a provider it changed twice, is a spoofing target and a spam-folder resident.

SMS evidence is thinner than most teams assume. The carrier delivery receipt tells you a message reached a handset; there is no equivalent of a bounce classification, no per-recipient policy report, and the retention window on those receipts is set by someone else. For a US-only notice flow that may be acceptable. In the EU, where an electronic registered delivery service under eIDAS provides evidence of both sending and receipt, an SMS receipt is not a substitute for a registered delivery channel when the statute demands one — the catch is that a fast, cheap channel and a legally-recognized channel are two different products, and a compliance notice sometimes needs the second.

## A minimal challenge record: the code an auditor can follow

The pattern is the same for both message classes: render from an owned template, hash the exact bytes you rendered, store that hash with the challenge, and let the provider status arrive later as an attachment to that record rather than as the record itself.

```ts
import { createHash, createHmac, randomInt, timingSafeEqual } from "node:crypto";

type Channel = "sms" | "email";

interface SendRecord {
  id: string;
  accountId: string;
  channel: Channel;
  templateId: string;        // "notice.entry.v4" or "auth.otp.v2"
  bodySha256: string;        // hash of the exact bytes handed to the provider
  sentAt: string;
  providerMessageId?: string;
  providerStatus?: "queued" | "delivered" | "failed";
  statusAt?: string;
}

const CODE_TTL_MS = 10 * 60 * 1000;
const MAX_ATTEMPTS = 5;

// Templates live in the repo so CI can diff them; the provider only sees rendered text.
function render(template: string, vars: Record<string, string>): string {
  return template.replace(/\{\{(\w+)\}\}/g, (_, k) => vars[k] ?? "");
}

function sha256(s: string): string {
  return createHash("sha256").update(s, "utf8").digest("hex");
}

// Store a keyed digest, never the code itself — the record is evidence, not a secret store.
function digest(code: string, challengeId: string): Buffer {
  return createHmac("sha256", process.env.OTP_PEPPER as string)
    .update(`${challengeId}:${code}`)
    .digest();
}

export function issueCode(challengeId: string, template: string) {
  const code = String(randomInt(0, 1_000_000)).padStart(6, "0");
  const body = render(template, { code });
  return {
    body,
    bodySha256: sha256(body),
    digest: digest(code, challengeId),
    expiresAt: new Date(Date.now() + CODE_TTL_MS).toISOString(),
  };
}

export function verify(input: string, stored: Buffer, challengeId: string, attempts: number, expiresAt: string): boolean {
  if (attempts >= MAX_ATTEMPTS) return false;
  if (Date.parse(expiresAt) < Date.now()) return false;
  const candidate = digest(input.trim(), challengeId);
  return candidate.length === stored.length && timingSafeEqual(candidate, stored);
}
```

Two details carry most of the operational weight. The attempt counter and the expiry are checked before the comparison, so a burned challenge costs nothing to reject, and `timingSafeEqual` keeps the comparison from leaking the code one byte at a time. The `bodySha256` is what turns a log line into an audit record: when a resident disputes the wording of an entry notice, you produce the template version, re-render it, and show the hashes match. Without that hash you have a timestamp and a promise.

Provider status belongs in a separate write path — a callback handler or a poll job that updates `providerStatus` and `statusAt` by `providerMessageId` — because delivery confirmation arrives seconds to hours after the send, and blocking the login response on it would be an outage waiting for a slow carrier. Retention is a design decision too: keep notice records as long as the statute of limitations for that notice type, and expire OTP records aggressively, because a table of phone numbers and challenge times is a liability with no evidentiary upside.

## When email should be the primary, and when nobody should get a code

Email deserves the primary slot in two cases I'd argue for without hesitation. The first is any workflow where the artifact matters more than the arrival time — notices, statements, lease documents — because you get authenticated sending, a delivery status vocabulary, unlimited body length, and attachments. The second is an international resident base where SMS registration overhead per country outruns the value of a few seconds of latency.

Stick with SMS-first only while the login population is phone-primary field staff.

And there's a class of accounts where you should stop shipping codes altogether. Portfolio managers and anyone with delete permissions on the notice archive should be on passkeys or a TOTP app; the practice worth adopting is to make the restricted channels a fallback with a visible warning rather than the default. If your platform can't offer that today, say so in the risk register and put it on the roadmap — I'm not sure any code-over-a-message design survives the next few years of account-takeover tooling, and I wouldn't build a five-year plan around one.

The template ownership question is the one I'd settle first, though. It's the decision that's expensive to reverse, it dictates how much of your delivery record you actually control, and it's the difference between an audit that takes an afternoon and one that takes a lawyer.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
- [Apple: Use Mail Privacy Protection on iPhone](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [NIST SP 800-63B: Digital Identity Guidelines — Authentication and Lifecycle Management](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [RFC 6238: TOTP — Time-Based One-Time Password Algorithm](https://datatracker.ietf.org/doc/html/rfc6238)
- [RFC 3464: An Extensible Message Format for Delivery Status Notifications](https://datatracker.ietf.org/doc/html/rfc3464)
- [Regulation (EU) No 910/2014 (eIDAS), including electronic registered delivery services](https://eur-lex.europa.eu/eli/reg/2014/910/oj)
