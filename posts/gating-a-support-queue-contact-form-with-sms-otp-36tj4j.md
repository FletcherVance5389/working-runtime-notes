# Gating a support-queue contact form with SMS OTP phone verification in Next.js

A public contact form on a B2B SaaS site is a queue router with a text box in front of it. Mine fans out three ways — billing, technical, and account lockout — and only the lockout path is worth paying an SMS bill for, because that is the queue where a stranger's request burns the most support time before anyone can act on it. The constraint that decided the design had nothing to do with the form: a Next.js route handler cannot assume that two requests from the same visitor land on the same instance, so a phone verification countdown living in component state is decoration.

Bottom line: keep the code, the cooldown clock, the attempt budget and the single-use flag in the backend, hand the browser one absolute timestamp, and treat "this number answered a challenge" as an input to the routing decision rather than as a login. The button renders what the server already decided.

The economics for a one-person product are boring. Delivery gets outsourced — carriers, sender IDs and international routing are somebody else's full-time job. The hundred-odd lines that decide who gets a code, how often, and which queue the resulting ticket lands in stay in the repo, under test, because that is the part my triage depends on.

## A verified number is a routing input, not a login trophy

An SMS OTP proves one thing: somebody could read a message sent to that number, once, a moment ago. NIST classifies PSTN-delivered out-of-band authenticators as restricted precisely because number control drifts — porting, recycling, and SIM swap all move it — and OWASP's guidance treats interception as a live threat rather than an edge case. So the claim my backend writes down is narrow and dated: `{ e164, verifiedAt, challengeId }`, nothing more.

That narrowness is what makes the routing rule easy to write.

| Form input | Queue | Verification it needs |
| --- | --- | --- |
| topic=billing, any plan | billing | none; the account already pays by card |
| topic=technical, plan=free | community | none; cost of a wrong route is low |
| topic=technical, plan=team | technical | none, but rate-limit the form |
| topic=lockout, phone on file | account-recovery | phone challenge answered, number matches the record |
| topic=lockout, phone absent or mismatched | identity-hold | human review, no automated recovery |

Normalize to E.164 before anything else reads the number. The ITU numbering plan is the only shared shape you get across US and EU submitters, and if you skip normalization your per-number counters are trivially bypassed: `+1 (555) 010-0199`, `001555010199` and `5550100199` are three keys in the same store and one phone in reality.

## How should the Next.js backend own the SMS OTP countdown instead of the button?

One row per challenge, mutated only through guarded transitions. The store below exposes exactly three operations, and `swap` is the whole concurrency story — it applies the patch only if the guard still holds on the stored row, which is a conditional update in Postgres, a Lua script in Redis, or a condition expression in DynamoDB.

```ts
import { createHmac, randomInt, randomUUID, timingSafeEqual } from "node:crypto";

const POLICY = { ttlMs: 300_000, cooldownMs: 60_000, maxSends: 3, maxFailures: 5 };

type Challenge = {
  id: string;
  e164: string;
  codeHash: string;
  expiresAt: number;
  nextSendAt: number;
  sends: number;
  failures: number;
  open: boolean;
};

type Store = {
  insert(row: Challenge): Promise<void>;
  read(id: string): Promise<Challenge | null>;
  /** Applies patch only while guard still holds on the stored row. */
  swap(
    id: string,
    guard: (row: Challenge) => boolean,
    patch: (row: Challenge) => Challenge,
  ): Promise<Challenge | null>;
};

type Sms = { deliver(e164: string, body: string, idempotencyKey: string): Promise<void> };

const freshCode = () => String(randomInt(0, 1_000_000)).padStart(6, "0");
const digest = (code: string, e164: string) =>
  createHmac("sha256", process.env.OTP_PEPPER!).update(`${e164}:${code}`).digest("hex");

export async function openChallenge(e164: string, store: Store, sms: Sms, now = Date.now()) {
  const code = freshCode();
  const row: Challenge = {
    id: randomUUID(),
    e164,
    codeHash: digest(code, e164),
    expiresAt: now + POLICY.ttlMs,
    nextSendAt: now + POLICY.cooldownMs,
    sends: 1,
    failures: 0,
    open: true,
  };
  await store.insert(row);
  await sms.deliver(e164, `${code} is your support verification code`, `${row.id}:1`);
  return { challengeId: row.id, nextSendAt: row.nextSendAt };
}

export async function sendAgain(id: string, store: Store, sms: Sms, now = Date.now()) {
  const code = freshCode();
  const row = await store.swap(
    id,
    (r) => r.open && now < r.expiresAt && now >= r.nextSendAt && r.sends < POLICY.maxSends,
    (r) => ({
      ...r,
      codeHash: digest(code, r.e164),
      sends: r.sends + 1,
      nextSendAt: now + POLICY.cooldownMs,
    }),
  );
  if (!row) {
    const current = await store.read(id);
    return { sent: false as const, nextSendAt: current?.nextSendAt ?? now + POLICY.cooldownMs };
  }
  await sms.deliver(row.e164, `${code} is your support verification code`, `${row.id}:${row.sends}`);
  return { sent: true as const, nextSendAt: row.nextSendAt };
}

export async function submitCode(id: string, code: string, store: Store, now = Date.now()) {
  const row = await store.read(id);
  if (!row || !row.open || now >= row.expiresAt) return { verified: false as const, reason: "closed" };

  const want = Buffer.from(row.codeHash, "hex");
  const got = Buffer.from(digest(code, row.e164), "hex");
  if (want.length !== got.length || !timingSafeEqual(want, got)) {
    await store.swap(
      id,
      (r) => r.open,
      (r) => ({ ...r, failures: r.failures + 1, open: r.failures + 1 < POLICY.maxFailures }),
    );
    return { verified: false as const, reason: "mismatch" };
  }

  const closed = await store.swap(id, (r) => r.open, (r) => ({ ...r, open: false }));
  if (!closed) return { verified: false as const, reason: "spent" };
  return { verified: true as const, e164: row.e164, verifiedAt: now };
}
```

Four details in there carry most of the weight. The code is stored as a keyed digest, so a leaked table dump doesn't hand out live codes, and the comparison is constant-time. A re-send mints a new code instead of resurrecting the old one, which means nothing needs to be decryptable. The idempotency key carries the send counter, so a retried write can't bill two messages for one press. And verification closes the row through a guarded swap, so two submissions racing on the same challenge produce one winner — without that, both requests read an open row and both proceed.

The wire format matters as much as the state. The route handler answers with the absolute `nextSendAt`, and when a request arrives early it answers 429 with `Retry-After`, which is the mechanism HTTP already defines for "come back later" — status code from RFC 6585, semantics from RFC 9110. The client renders `nextSendAt - Date.now()` and disables the button until zero. A background tab that gets throttled, or a laptop with a skewed clock, will render a slightly wrong label; the server's answer is still the one that counts, and the client learns the truth on its next call.

Then the routing layer consumes the claim, and only the claim:

```ts
type Submission = {
  topic: "billing" | "technical" | "lockout";
  plan: "free" | "team" | "enterprise";
  phoneOnFile: string | null;
};

export function routeToQueue(s: Submission, verified: { e164: string } | null) {
  if (s.topic === "lockout") {
    return verified && verified.e164 === s.phoneOnFile ? "account-recovery" : "identity-hold";
  }
  if (s.topic === "billing") return "billing";
  return s.plan === "free" ? "community" : "technical";
}
```

Both functions take `now` and a gateway as parameters, which is the only reason this is testable without a network call: a fake `Sms`, a fake clock, and the boundaries write themselves — one millisecond before the cooldown expires, the exact expiry instant, the last allowed failure, two concurrent correct submissions, a delivery call that throws after the row was already patched. That last one is the case teams usually skip, and it's the one that produces a challenge nobody can finish.

## What I would change once the queue gets real volume

Move `POLICY` out of the module and into reviewed configuration, because the cooldown becomes a business argument the moment support asks why a customer waited 60 seconds. Add budgets on two more dimensions — per number per day, and per source address — since a public form is a free SMS button for anyone who finds it, and country routing costs vary enough that an allowlist matched to where paying customers actually live is the cheapest fraud control available.

Log decisions, not codes. Every allow and deny gets a line keyed by challenge id, which is how you later answer why a ticket landed in `identity-hold` at 02:14 without keeping anything sensitive in the log.

The email leg of the same form deserves the same rigor and usually gets none. The acknowledgment message needs DKIM signing so the receiving domain can bind it to yours, and the temptation to measure "did they see it" by open tracking is worth resisting — Apple's Mail Privacy Protection proxies remote content, so image loads say more about a privacy relay than about a human. If you want a signal, ask for a click on something that changes state.

On mobile, `autocomplete="one-time-code"` and the WebOTP API do the boring conversion work of filling the field for the visitor. Nice to have. Not part of the security model, since anything the client fills in is still checked server-side.

## Where phone verification is the wrong call for a contact form

The catch is that every gate you add drops legitimate submissions. If the real goal is spam reduction on a general contact form, a phone challenge is expensive theatre; a cheap bot check plus email confirmation costs less and rejects fewer real customers. Stick with a plain dropdown and manual triage when a human reads every submission anyway — at that volume the integration effort loses to the salary it was meant to save, and I'd rather ship a feature that week.

It's also not suitable when you need identity rather than reachability. Number control isn't proof of who someone is, so anything with a regulatory identity requirement needs a different instrument entirely, and the lockout queue should still end in a human decision for high-value accounts.

One more boundary, and it's the one I see duplicated most: if a managed auth provider already owns your phone factor, adding a parallel challenge store means two systems believe they hold the truth about the same number. Read the claim from the provider and route on it. Two state machines for one fact is how you get a support ticket in the wrong queue and no way to explain it.

Own the state machine, not the delivery.

## References

- ITU-T Recommendation E.164 — international public telecommunication numbering plan: https://www.itu.int/rec/T-REC-E.164
- NIST SP 800-63B — Digital Identity Guidelines, authenticator requirements: https://pages.nist.gov/800-63-3/sp800-63b.html
- OWASP Multifactor Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Multifactor_Authentication_Cheat_Sheet.html
- RFC 6585 — Additional HTTP Status Codes (429 Too Many Requests): https://www.rfc-editor.org/rfc/rfc6585.html
- RFC 9110 — HTTP Semantics (Retry-After): https://www.rfc-editor.org/rfc/rfc9110.html
- RFC 6376 — DomainKeys Identified Mail (DKIM) Signatures: https://datatracker.ietf.org/doc/html/rfc6376
- Apple — Use Mail Privacy Protection: https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
- MDN — WebOTP API: https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API
