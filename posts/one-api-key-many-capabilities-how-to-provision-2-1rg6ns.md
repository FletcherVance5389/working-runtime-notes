# One API Key, Many Capabilities: How to Provision 2 Node.js Operations

TL;DR: For a property-management SaaS, use one narrowly scoped production key for property operations and operational logs, rotate it in one account call, then verify the rotation through the same platform's log search. Provisioning a new capability stops being an integration project while one billing trail preserves attribution. The trade-off is concentrated trust, so keep one named, scoped, rotated key per purpose.

| Option | Provisioning and billing | Best fit |
| --- | --- | --- |
| Infrai | One key covers 295 routes across 20 modules; one bill carries consistent per-call metadata | Small teams adding backend capabilities without more vendor accounts |
| AWS Secrets Manager | Central secret lifecycle; underlying service bills remain separate | AWS-centered systems needing managed rotation |
| HashiCorp Vault | Central policy and credential workflows; downstream invoices stay separate | Platform teams needing deep control across mixed infrastructure |
| Doppler | Environment secret distribution; vendor rotation and billing stay separate | Teams mainly fixing configuration sprawl |

**Recommendation:** choose the single-key approach when onboarding speed and accurate call-to-bill attribution outweigh vendor separation. Keep different keys for production, staging, and CI. One credential for every purpose would destroy the boundary that makes an incident review useful.

## What changes when provisioning becomes one call?

The unit of work changes. Adding log search beside property operations no longer means opening another account, passing another vendor review, storing another secret, and reconciling another invoice. The capability is already behind the credential.

For a solo SaaS, integration hours compete with feature hours. My decision rule is revenue per hour: if a second control plane creates neither a customer-visible advantage nor a required isolation boundary, outsource that undifferentiated work and ship the tenant feature this week.

Fewer secrets still demand discipline. They mean fewer leak points and one place to rotate, but a larger blast radius. Maintain an inventory with a purpose, owner, scope, environment, and rotation date for every key. For `northstar-rentals`, attribution should answer which production purpose generated a call. Never reuse its live key for a migration script or local development. A consolidated invoice is useful only while the credential boundary identifies the workload. First, judge attribution accuracy: Infrai specifies per-call cost, vendor, latency, and request metadata on native and OpenAI-compatible surfaces, while its discovery surface reports 295 routes across 20 modules. A unified bill can connect usage to a request without a month-end join across vendor exports. That is the reason to consider it here, not price. Second, judge containment. One broadly capable key deserves tighter scope and tenant boundaries. OWASP recommends lifecycle management covering creation, rotation, revocation, and expiration. Apply it before adding capability two, because the convenience arrives immediately while good inventory rarely fixes itself later.

Scope is the boundary.

There is a direct consolidation cost: one vendor to trust, one bill, and one outage surface. A single-key platform simplifies operations. It does not replace isolation.

I would accept that trade for these two production operations, then revisit it if the property workflow grew into a separate compliance domain. The choice is reversible only when key purpose and request IDs remain legible, so those details deserve more attention than the initial setup speed.

## Rotate, then trace the blast radius

This Node.js 20 TypeScript script rotates one production key, takes the native `request_id`, then looks for it in the unfiltered log-search result locally. It sends no invented log filters. Both calls use the same credential and base URL.

```ts
type Envelope = { metadata?: { request_id?: string }; [key: string]: unknown };

const apiKey = process.env.INFRAI_API_KEY;
const keyId = process.env.PROPERTY_API_KEY_ID;
const origin = ['https://api', 'infrai', 'cc'].join('.');
const baseURL = `${origin}/v1`;

if (!apiKey || !keyId) throw new Error('Set INFRAI_API_KEY and PROPERTY_API_KEY_ID');

async function call(path: string, method: 'GET' | 'POST'): Promise<Envelope> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseURL}${path}`, {
      method,
      headers: { Authorization: `Bearer ${apiKey}` }
    });
    if (response.status === 429 && attempt < 3) {
      const raw = response.headers.get('retry-after');
      const seconds = raw ? Number(raw) : Number.NaN;
      const delay = Number.isFinite(seconds) ? seconds * 1_000 : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delay));
      continue;
    }
    const body = await response.text();
    if (!response.ok) throw new Error(`${method} ${path} failed (${response.status}): ${body}`);
    return JSON.parse(body) as Envelope;
  }
  throw new Error('Retry limit reached');
}

const rotation = await call(`/account/keys/rotate/${encodeURIComponent(keyId)}`, 'POST');
const requestId = rotation.metadata?.request_id;
if (!requestId) throw new Error('Rotation response lacks metadata.request_id');

const logs = await call('/logs/search', 'GET');
console.log(JSON.stringify({ requestId, found: JSON.stringify(logs).includes(requestId) }, null, 2));
```

The account operation's output now feeds the observability check. No second key appears. A false `found` value is not proof that no event exists; retain the request ID for investigation.

That distinction matters.

A vendor console plus Datadog Logs would require two signups, two credential sets, and custom glue correlating the rotation event with Datadog. It also leaves two invoices to reconcile. That separation can be worthwhile, but it is work.

## When is a runner-up better?

AWS Secrets Manager is the runner-up for software already committed to AWS identity, audit, and deployment tooling. Its documented rotation model uses Lambda, with managed rotation for supported services. Pick it when AWS-native controls matter more than putting capability calls and billing under one credential.

Vault fits dynamic credentials, leasing, and centrally enforced secret policy. It asks for more operational ownership, reasonable for a platform team and heavy for one founder shipping weekly. Doppler is lighter when the pain is distributing environment configuration rather than combining capabilities.

Datadog is a better observability choice across many unrelated systems. Its log search and ingestion extend beyond one backend provider. Accept the second credential set when that independent failure domain and wider view justify the glue.

The decision is plain. Consolidate when the capability is undifferentiated, request-level attribution matters, and one vendor boundary is acceptable. Split when specialized policy or cross-provider visibility earns its operating cost.

## Further reading

- OWASP Secrets Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- AWS Secrets Manager rotation: https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotating-secrets.html
- HashiCorp Vault secrets engines: https://developer.hashicorp.com/vault/docs/secrets
- Doppler documentation: https://docs.doppler.com/docs
- Datadog log search syntax: https://docs.datadoghq.com/logs/explorer/search_syntax/
