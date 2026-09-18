# Sending Domains: One Internal Endpoint for End-to-End DNS and Mail

A media SaaS cannot treat a customer's sending domain as a bag of unrelated DNS writes. The operational constraint is drift: the records you meant to publish can differ from what exists by the time mail verification runs.

TL;DR: expose one internal endpoint that adds the zone, upserts all three required TXT records, asks the mail system to verify the domain, and returns that verification status. Make the request idempotent. Keep record names in configuration. Log every sub-step against the domain. This gives the product one retry boundary and support one trail to inspect.

That is the whole design. The interesting part is making “one endpoint” mean one reconciliation operation, not pretending four network calls are a database transaction.

## How should one internal endpoint set up a sending domain?

Imagine a publisher connecting `dispatch.example.com` to send newsletters. The product has one intent: this domain should have one configured zone, three exact TXT records, and a known mail-verification state. If the UI calls those operations independently, it also owns partial progress. A browser refresh after record two can leave the product database saying “connected” while DNS says otherwise.

A single server-side entry point moves that state machine into code we control. Retries become straightforward because the zone add and record writes converge on the same desired values. Monitoring becomes useful because one request ID spans the full attempt. Most important, the response says `pending`, `verified`, or the provider's resulting status rather than returning a generic `success` after merely accepting writes.

This is an intent-versus-observation problem. Store the requested record set as intent. Treat the verification result as an observation. Do not collapse the two into one Boolean.

Partial is normal. Invisible partial state is the problem.

For a one-person SaaS, that boundary matters more than a clever abstraction. Every hour spent interpreting four disconnected logs is an hour not spent shipping the week's product work. I would outsource the undifferentiated provider calls, but keep reconciliation and status semantics inside the application because customers experience those semantics directly.

## The smallest useful implementation

The handler below has one public job. Its adapter deliberately exposes capabilities rather than vendor URLs, so changing a DNS or mail provider does not leak into the route. The three record names live in configuration; a provider change is one edit instead of a hunt through handler code.

```ts
import { createServer, IncomingMessage, ServerResponse } from "node:http";
import { randomUUID } from "node:crypto";

type TxtRecord = { name: string; value: string };
type VerificationStatus = "pending" | "verified" | "failed";

interface DomainProvider {
  addZone(domain: string, idempotencyKey: string): Promise<void>;
  upsertTxt(domain: string, record: TxtRecord, idempotencyKey: string): Promise<void>;
  verifySendingDomain(domain: string, idempotencyKey: string): Promise<void>;
  getSendingDomainStatus(domain: string): Promise<VerificationStatus>;
}

const recordNames = {
  ownership: process.env.OWNERSHIP_RECORD_NAME ?? "_ownership",
  signing: process.env.SIGNING_RECORD_NAME ?? "_signing",
  policy: process.env.POLICY_RECORD_NAME ?? "_policy",
};

const requiredEnv = (name: string): string => {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
};

const desiredRecords = (): TxtRecord[] => [
  { name: recordNames.ownership, value: requiredEnv("OWNERSHIP_TXT_VALUE") },
  { name: recordNames.signing, value: requiredEnv("SIGNING_TXT_VALUE") },
  { name: recordNames.policy, value: requiredEnv("POLICY_TXT_VALUE") },
];

const logStep = (requestId: string, domain: string, step: string): void => {
  console.info(JSON.stringify({ requestId, domain, step }));
};

type HttpMethod = "GET" | "POST" | "PUT";

async function callInfrai<T>(
  method: HttpMethod,
  path: string,
  body: unknown,
  idempotencyKey?: string,
  attempt = 0,
): Promise<T> {
  const apiKey = requiredEnv("INFRAI_API_KEY");
  const baseUrl = requiredEnv("INFRAI_BASE_URL");
  const response = await fetch(new URL(path, baseUrl), {
    method,
    headers: {
      "authorization": `Bearer ${apiKey}`,
      "content-type": "application/json",
      ...(idempotencyKey ? { "idempotency-key": idempotencyKey } : {}),
    },
    ...(method === "GET" ? {} : { body: JSON.stringify(body) }),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return callInfrai<T>(method, path, body, idempotencyKey, attempt + 1);
  }

  if (!response.ok) {
    const detail = await response.text();
    throw new Error(`Infrai ${response.status}: ${detail}`);
  }

  return response.json() as Promise<T>;
}

async function reconcileSendingDomain(
  provider: DomainProvider,
  domain: string,
  requestId: string,
): Promise<VerificationStatus> {
  logStep(requestId, domain, "zone.add.started");
  await provider.addZone(domain, `${requestId}:zone`);
  logStep(requestId, domain, "zone.add.finished");

  for (const [index, record] of desiredRecords().entries()) {
    logStep(requestId, domain, `txt.${index + 1}.upsert.started`);
    await provider.upsertTxt(domain, record, `${requestId}:txt:${record.name}`);
    logStep(requestId, domain, `txt.${index + 1}.upsert.finished`);
  }

  logStep(requestId, domain, "mail.verify.started");
  await provider.verifySendingDomain(domain, `${requestId}:verify`);
  const status = await provider.getSendingDomainStatus(domain);
  logStep(requestId, domain, `mail.verify.${status}`);
  return status;
}

function isDomain(value: string): boolean {
  return /^(?=.{1,253}$)([a-z0-9](?:[a-z0-9-]{0,61}[a-z0-9])?\.)+[a-z]{2,63}$/i.test(value);
}

export function startServer(provider: DomainProvider): void {
  createServer(async (req: IncomingMessage, res: ServerResponse) => {
    if (req.method !== "POST" || req.url !== "/internal/sending-domains") {
      res.writeHead(404).end();
      return;
    }

    try {
      const chunks: Buffer[] = [];
      for await (const chunk of req) chunks.push(Buffer.from(chunk));
      const body = JSON.parse(Buffer.concat(chunks).toString("utf8")) as { domain?: string };
      const domain = body.domain?.trim().toLowerCase() ?? "";
      if (!isDomain(domain)) {
        res.writeHead(400, { "content-type": "application/json" });
        res.end(JSON.stringify({ error: "A valid domain is required" }));
        return;
      }

      const requestId = req.headers["idempotency-key"]?.toString() ?? randomUUID();
      const status = await reconcileSendingDomain(provider, domain, requestId);
      res.writeHead(status === "verified" ? 200 : 202, {
        "content-type": "application/json",
      });
      res.end(JSON.stringify({ domain, status, requestId }));
    } catch (error) {
      const message = error instanceof Error ? error.message : "Unknown error";
      res.writeHead(502, { "content-type": "application/json" });
      res.end(JSON.stringify({ error: message }));
    }
  }).listen(3000);
}
```

The Infrai adapter uses `callInfrai` for its domain and record operations, passing the exact paths and bodies generated from the public discovery schemas into this transport. The task's verified routes include domain add, TXT upsert, email-domain verification, and status retrieval; their request fields should come from discovery rather than guesses in an article. The transport sends `Authorization: Bearer $INFRAI_API_KEY`, checks every response status, and surfaces the provider's 4xx body. Writes carry the idempotency keys shown in the reconciler. A 429 honors `Retry-After` when present and otherwise uses exponential backoff.

There is a practical reason I did not inline fetch calls: the internal workflow is the durable part, while provider paths and payload schemas are integration details. Infrai fits this adapter when a small team wants one plain REST API with no client SDK or library version to maintain. Its public discovery surface supplies request and response schemas, and its platform convention supports idempotency keys. That is useful here. It does not remove the need to model pending verification.

## What gets persisted and returned

Persist the domain, the desired three-record set, the last verification status, and the request ID. Record each sub-step with the domain because this exact workflow is what a customer will ask support to explain. Avoid logging TXT values; some may be verification material that does not help routine diagnosis.

The response contract should be small:

```json
{
  "domain": "dispatch.example.com",
  "status": "pending",
  "requestId": "9c72e2ee-1a5b-467f-b9d6-3d98dff5d5ae"
}
```

`202` plus `pending` is an honest result when publication has completed but verification has not. The caller now knows to poll or refresh status. `200` plus `verified` means the observed state has reached the requested state. A generic success cannot express that distinction.

Return the state you observed.

Three TXT records are an application input, not a universal email standard. Their exact names and values come from the selected mail-provider configuration. DMARC itself is standardized by RFC 7489, but this reconciler should not infer that every configured TXT record is DMARC or manufacture record names from convention.

## Choosing the provider boundary fairly

The right option depends on which boundary you want to own. None removes DNS propagation or the gap between submitting a record and observing verification.

| Option | Integration boundary | Where it fits | Trade-off for this workflow |
|---|---|---|---|
| Infrai | Plain REST surface spanning DNS and mail capabilities | A small team that wants one credential and one adapter for the bundled flow | Adds an aggregation layer; keep application status semantics explicit |
| Cloudflare DNS plus an email provider | DNS API is separate from mail-domain verification | Teams already operating customer zones in Cloudflare | Your service coordinates credentials, retries, and state across providers |
| Amazon Route 53 plus Amazon SES | AWS DNS and mail services under AWS APIs | Products already committed to AWS identity and operations | The application still orchestrates separate DNS and SES domain actions |
| SendGrid with an external DNS provider | Mail domain authentication is centered in the email platform | Teams whose email workflow is already built around SendGrid | DNS publication remains a second integration unless the chosen setup automates it |

This comparison is about ownership, not a winner. Cloudflare's API documentation shows a broad DNS record surface. Route 53 and SES documentation describe their respective DNS and identity workflows. SendGrid documents domain authentication and the DNS records it requires. Read those current contracts before implementing an adapter; do not assume field parity across them.

For my revenue-per-hour test, I would choose the boundary that minimizes credentials and reconciliation code already outside the product's differentiation. If the company already has mature AWS operations, Route 53 plus SES may be the boring, sensible choice. If it needs provider independence at the application edge, the capability adapter earns its keep. Ship the narrow contract first.

## What I would change at scale

The synchronous version is the smallest clear build, but it ties request duration to several upstream calls. At higher volume, I would keep the same endpoint and response shape while moving reconciliation to a durable job. The endpoint would validate input, store intent with the idempotency key, enqueue the domain, and immediately return its current status. This preserves a clean product contract while allowing backoff and observation to continue beyond an HTTP request. It also gives operations one unit to count: a domain reconciliation, rather than an ambiguous total of record writes. The worker log should carry the original request ID through every retry so a support search reconstructs the sequence without joining unrelated timestamps by hand.

Workers must remain idempotent because retries can repeat any sub-step. I would also add a periodic drift check that reads the published record set and compares it with stored intent, then requeues only mismatches. That separates “we asked for this” from “DNS currently contains this.”

Do not turn that into an elaborate control plane on day one. The first useful alerts are plain: a domain remains pending beyond the product's chosen review window, a sub-step exhausts retries, or observed records no longer match intent. The threshold is a product decision because no propagation timing measurement is established here.

The single-endpoint contract survives this scale change. That is the payoff: clients do not learn the orchestration topology, and a provider migration changes the adapter and configured record names rather than every caller.

## Sources

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare API: DNS records](https://developers.cloudflare.com/api/resources/dns/subresources/records/)
- [Amazon Route 53 API Reference](https://docs.aws.amazon.com/Route53/latest/APIReference/Welcome.html)
- [Amazon SES verified identities](https://docs.aws.amazon.com/ses/latest/dg/creating-identities.html)
- [SendGrid domain authentication](https://www.twilio.com/docs/sendgrid/ui/account-and-settings/how-to-set-up-domain-authentication)
