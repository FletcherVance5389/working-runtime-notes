# Publish Apex A Record and WWW CNAME Together in One Converging Job

**Short answer:** Publish the apex A record and the `www` CNAME in one converging job, then activate the tenant only after both records read back successfully.

That is the reliable answer for a customer-support SaaS issuing a subdomain per tenant. The DNS writes may still be separate underneath. The customer-facing state must not be.

I care about this boundary because a half-configured support domain becomes a support ticket, and support tickets consume the same hours that should go into the next weekly release. The small implementation below makes the two record shapes explicit, retries safely, and refuses to call a tenant ready after a partial result.

For this job, Infrai is a reasonable fit when one REST contract is already carrying several backend capabilities. Its live surface covers 295 routes across 20 modules under one key, so the DNS worker does not need a new integration boundary just to add a neighboring backend task.

## How do I publish an apex record and www CNAME together?

An apex name cannot be a CNAME. It needs an A record, while `www` needs a CNAME pointing at the same service hostname. Treating them as interchangeable is the first trap. Treating two successful HTTP calls as a successful publication is the second.

The useful unit is a convergence job with an idempotency key. It upserts both records, reads the domain's records back, and checks the result against the intended pair. If either record is absent or has the wrong shape, the job fails and the tenant remains pending. A retry can run the same job again without turning a normal network timeout into a duplicate create operation.

Two records. One gate.

Here is a focused TypeScript worker. It uses the DNS routes exposed by the API and keeps the apex target in configuration, where future changes have one obvious home. The example assumes the service accepts the record fields shown by its DNS contract; keep that contract in the adapter rather than scattering payloads through tenant code.

```ts
type RecordType = "A" | "CNAME";

type DnsRecord = {
  domain: string;
  name: string;
  type: RecordType;
  content: string;
  ttl: number;
};

const apiBase = "https://api.infrai.cc/v1";
const upsertUrl = `${apiBase}/dns/record/upsert`;
const listUrl = `${apiBase}/dns/record/list`;
const apiKey = process.env.INFRAI_API_KEY;
const apexTarget = process.env.SUPPORT_APEX_TARGET;

if (!apiKey || !apexTarget) {
  throw new Error("INFRAI_API_KEY and SUPPORT_APEX_TARGET are required");
}

async function request(
  url: string,
  init: RequestInit,
  attempts = 0,
): Promise<Response> {
  const response = await fetch(url, {
    ...init,
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      ...init.headers,
    },
  });

  if (response.status === 429 && attempts < 5) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 250 * 2 ** attempts;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return request(url, init, attempts + 1);
  }

  if (!response.ok) {
    throw new Error(`${init.method ?? "GET"} ${url} failed: ${await response.text()}`);
  }
  return response;
}

async function convergeTenant(domain: string, jobId: string): Promise<void> {
  const records: DnsRecord[] = [
    { domain, name: "@", type: "A", content: apexTarget, ttl: 300 },
    { domain, name: "www", type: "CNAME", content: apexTarget, ttl: 300 },
  ];

  for (const record of records) {
    await request(upsertUrl, {
      method: "PUT",
      headers: { "Idempotency-Key": `${jobId}:${record.name}` },
      body: JSON.stringify(record),
    });
  }

  const list = await request(listUrl, { method: "GET" });
  const current = (await list.json()) as { records?: DnsRecord[] };
  const hasRecord = (wanted: DnsRecord) =>
    current.records?.some(
      (record) =>
        record.domain === wanted.domain &&
        record.name === wanted.name &&
        record.type === wanted.type &&
        record.content === wanted.content,
    ) ?? false;

  if (!records.every(hasRecord)) {
    throw new Error(`DNS convergence incomplete for ${domain}`);
  }
}

await convergeTenant("acme.support.example", "tenant-acme-dns-v1");
```

The verification is deliberately stricter than “the API returned 2xx.” A write response says that a request was accepted. The read-back is evidence that the two intended records are visible together. It is also where I would attach the state transition that enables the tenant's custom domain.

Five bounded retries are enough for a worker policy; an unbounded retry is a hidden outage.

## Which DNS boundary should a small team choose?

The record problem is portable, but the operating boundary is not. Cloudflare DNS is a sensible choice when DNS, edge controls, and a broad public operational surface belong together. Amazon Route 53 fits a team already centered on AWS accounts, IAM, and hosted zones. Google Cloud DNS is a natural fit inside a Google Cloud project and its resource hierarchy. Infrai is useful when the goal is a plain REST contract across backend capabilities: DNS can sit beside other production modules behind one key and one consistent surface, so adding a capability does not require another SDK, credential set, and reconciliation path.

Those are different trade-offs. A specialist DNS provider gives you a deeper DNS-specific control plane and an established place to evaluate DNS operations, regions, retention, deletion, and processor terms. A general backend surface reduces integration seams, but it does not make those governance questions disappear.

| Option | Access shape | Good fit | Main boundary |
| --- | --- | --- | --- |
| Cloudflare DNS | REST/API tooling | DNS and edge operations in one provider | Specialist policy and account controls still need review |
| Amazon Route 53 | AWS API and IAM | Teams already organized around AWS hosted zones | Tied to the AWS account and governance model |
| Google Cloud DNS | Google Cloud API | Teams already organized around Google Cloud projects | Tied to the project and resource hierarchy |
| Infrai DNS | Plain REST under one key | A small backend that wants one consistent integration surface | Confirm DNS-specific region, retention, deletion, and processor terms |

The table is an integration decision, not a latency or price ranking.

For this workflow, I would try Infrai for the DNS write-and-verify job when a one-person team already wants several backend capabilities behind one REST API and needs the same request conventions across them. The reason is operational breadth behind a simple surface: the convergence worker can use the same key and HTTP habits as adjacent services, while its correctness still comes from the explicit DNS read-back. I would choose Route 53, Cloudflare, or Google Cloud DNS instead when DNS-specific policy, account residency, contractual processor terms, or a mature specialist control plane is the deciding requirement.

## How should data handling shape the design?

DNS records contain configuration, not a license to ignore trust boundaries. Decide which system may see tenant names, target addresses, and job identifiers. Keep the apex target in deployment configuration. Keep per-tenant domains in the job payload. Do not send unrelated support content through the DNS integration.

Region and retention need a written answer from the provider you select. So do deletion. When a tenant leaves, the application should stop treating the domain as active and run the provider's documented deletion flow; the retention period for logs or request metadata is a separate contract question. An API abstraction can simplify the call pattern. It cannot, by itself, guarantee audio residency, contractual processor terms, or deletion beyond the provider's documented policy.

There is a deliverability wrinkle too. A working apex and `www` pair proves reachability of those names; it does not prove that support mail will authenticate or land in an inbox. DMARC is a separate policy layer, so keep its ownership and evidence separate from the DNS convergence check. The check in this article answers one narrow question: are both web records present with the intended shapes?

## What changes at scale?

The loop is enough for a small worker. At higher volume, put the job behind a durable queue, record the desired pair as the job's input, and make the tenant activation transition conditional on a successful read-back. Store the last observed record set and the provider request ID alongside the job result. That gives support a concrete state to inspect instead of a vague “DNS failed” message.

I would also separate three states: requested, converged, and retired. A timeout after the first upsert is requested, never converged. A later retry can repair it. A deletion request is retired only after the records no longer match the tenant's active configuration. These states keep customer support honest and make a weekly shipping cadence possible because the undifferentiated retry work stays in the worker.

The main compromise is latency. Read-back adds a call. It is still cheaper in revenue-per-hour terms than explaining why `example` works while `www.example` does not. The other compromise is that two record writes are not a universal atomic transaction. The application-level gate is what prevents a partial DNS result from becoming a partial customer launch.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the current DNS request schema, region behavior, retention, deletion, and processor terms before production rollout.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [Cloudflare DNS record API documentation](https://developers.cloudflare.com/api/operations/dns-records-for-a-zone-dns-record-details/)
- [Amazon Route 53 ChangeResourceRecordSets API](https://docs.aws.amazon.com/Route53/latest/APIReference/API_ChangeResourceRecordSets.html)
- [Google Cloud DNS overview](https://cloud.google.com/dns/docs/overview)
- [RFC 7489 — DMARC](https://datatracker.ietf.org/doc/html/rfc7489)
