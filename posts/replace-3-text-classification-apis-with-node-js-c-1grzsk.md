# Replace 3 Text Classification APIs with Node.js Chat Completions (Tenant Costs Traced)

Short answer: put a tenant-scoped usage record around one OpenAI-compatible chat completions call, keep the classification JSON fixed, and select discovered models through configuration. That gives a solo edtech SaaS one place to replace OpenAI, Claude, or Gemini without mixing infrastructure choices into human-review policy.

| Choice | Application boundary | Per-tenant cost evidence | Pick it when |
| --- | --- | --- | --- |
| Direct OpenAI | OpenAI-specific connection | Attribute calls in your own ledger | OpenAI-specific platform behavior matters |
| Direct Anthropic Claude | Claude-specific connection | Attribute calls in your own ledger | Claude is a deliberate fixed dependency |
| Direct Google Gemini | Gemini-specific connection | Attribute calls in your own ledger | Gemini is already the operational standard |
| A compatible router | One chat contract; model chosen in configuration | Returned cost, vendor, latency, and request metadata can join the tenant ledger | Provider replacement and one integration boundary both matter |

For a one-person product that ships weekly, I would choose the compatible layer only when tenant attribution and provider replacement are real requirements. Infrai fits this narrow case because one API key and one bill reduce reconciliation, while one REST API keeps the application contract stable when the serving vendor changes. Its public, self-describing discovery surface lets the app check model availability before an admin changes configuration. Those conveniences aren't substitutes for a product-owned usage ledger.

## How should one API key route OpenAI, Claude, and Gemini text classification?

Start by deciding what a billable classification means. For this moderation workflow, it is one report submitted for a named tenant, one validated label returned, and one usage entry attached to the same internal operation. The model is a configuration value. The tenant ID is not.

That distinction matters more than the dropdown. Imagine one school submitting 40 reports during a normal week while another imports 40,000 reports after enabling a public class forum. A blended monthly total says almost nothing about which account is expensive to serve. A row containing the tenant ID, selected model, serving vendor, call cost, and request ID gives the founder something actionable: evidence for plan limits, a way to inspect model experiments, and a clean input for margin reporting. It also stops an admin's model switch from silently changing how downstream records are shaped. I don't let the provider response become the database contract — the classifier result and the usage record are separate, narrow objects.

There is no dedicated moderation endpoint in this setup. Use a chat model with a strict JSON schema, then send invalid or low-confidence results to human review. Keep the prompt, label enum, and output fields unchanged while testing candidates. I'm not sure which model will classify a particular school's reports best until the same labeled evaluation set has been run against each one; transport compatibility cannot answer that.

Small boundary. Big payoff.

## Cost evidence begins where the tenant is still known

The safest place to write attribution is next to the classification request, before a queue or analytics export can lose business context. A useful record needs the configured model as well as returned metadata. Those values answer different questions: what the tenant configuration requested, and what the routing layer actually served.

Treat a cost estimate as rollout planning, not accounting. Comparing estimated costs is useful when daily tagging volume may be high, but actual per-call metadata belongs in the tenant ledger. Don't turn an estimate into an invoice.

The revenue-per-hour test is blunt: if reconciling a provider invoice requires joining logs by hand every month, the integration has pushed undifferentiated work back onto the founder. Fix the attribution boundary before tuning prompts.

## A focused TypeScript implementation

This focused example discovers available model IDs through `GET /v1/ai/models`, sends the classification through the OpenAI-compatible client, enforces one JSON shape, retries HTTP `429` with exponential backoff and `Retry-After`, and records returned metadata under the caller's tenant ID. It uses two routes total. The API key stays in the environment.

```ts
import OpenAI from "openai";

const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.CLASSIFIER_MODEL;

if (!apiKey || !model) {
  throw new Error("INFRAI_API_KEY and CLASSIFIER_MODEL are required");
}

const baseURL = process.env.OPENAI_BASE_URL;
if (!baseURL) throw new Error("OPENAI_BASE_URL is required");
const client = new OpenAI({ apiKey, baseURL });

type Classification = {
  label: "abuse" | "spam" | "other";
  confidence: number;
  reason: string;
};

type UsageEntry = {
  tenantId: string;
  requestedModel: string;
  vendor: string | null;
  costUsd: number | null;
  requestId: string | null;
};

async function availableModelIds(): Promise<Set<string>> {
  const response = await fetch(`${baseURL}/ai/models`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (!response.ok) {
    throw new Error(`Model discovery failed (${response.status}): ${await response.text()}`);
  }

  const body = (await response.json()) as {
    data: Array<{ id: string; available: boolean }>;
  };
  return new Set(body.data.filter((item) => item.available).map((item) => item.id));
}

function retryDelayMs(headers: Headers | undefined, attempt: number): number {
  const retryAfter = headers?.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }
  return 500 * 2 ** attempt;
}

async function classifyReport(
  tenantId: string,
  reportText: string,
): Promise<{ result: Classification; usage: UsageEntry }> {
  const models = await availableModelIds();
  if (!models.has(model)) throw new Error(`Configured model is unavailable: ${model}`);

  for (let attempt = 0; attempt < 4; attempt += 1) {
    try {
      const { data, response } = await client.chat.completions
        .create({
          model,
          messages: [
            {
              role: "system",
              content: "Classify an edtech moderation report. Return JSON matching the schema.",
            },
            { role: "user", content: reportText },
          ],
          response_format: {
            type: "json_schema",
            json_schema: {
              name: "moderation_report_classification",
              strict: true,
              schema: {
                type: "object",
                additionalProperties: false,
                properties: {
                  label: { type: "string", enum: ["abuse", "spam", "other"] },
                  confidence: { type: "number", minimum: 0, maximum: 1 },
                  reason: { type: "string" },
                },
                required: ["label", "confidence", "reason"],
              },
            },
          },
        })
        .withResponse();

      const content = data.choices[0]?.message.content;
      if (!content) throw new Error("Classification response contained no content");

      const result = JSON.parse(content) as Classification;
      if (
        !["abuse", "spam", "other"].includes(result.label) ||
        typeof result.confidence !== "number" ||
        typeof result.reason !== "string"
      ) {
        throw new Error("Classification response did not match the schema");
      }

      const metadata = (
        data as typeof data & {
          infrai?: { vendor?: string; cost_usd?: number; request_id?: string };
        }
      ).infrai;

      return {
        result,
        usage: {
          tenantId,
          requestedModel: model,
          vendor: metadata?.vendor ?? null,
          costUsd: metadata?.cost_usd ?? null,
          requestId: metadata?.request_id ?? null,
        },
      };
    } catch (error) {
      if (!(error instanceof OpenAI.APIError)) throw error;
      if (error.status !== 429 || attempt === 3) {
        throw new Error(`Classification request failed (${error.status})`, {
          cause: error,
        });
      }
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelayMs(error.headers, attempt)),
      );
    }
  }

  throw new Error("Classification retry limit reached");
}

const output = await classifyReport(
  "school_42",
  "A learner reports repeated promotional links in a class discussion.",
);

process.stdout.write(`${JSON.stringify(output, null, 2)}\n`);
```

The sample deliberately leaves persistence outside the transport function. In production, write the returned usage object with the moderation job's internal ID so replay behavior is explicit. The chat call does not publish a product-side state change, while your ledger write does; those are different retry decisions.

Also keep model discovery out of the hot path once the choice is validated. Refresh the available choices for admin configuration, reject unavailable selections, and cache the approved ID in tenant settings. The classification function should not invent a fallback model during an incident because that would make label-quality changes invisible to the operator.

## Replacing a model is a controlled rollout

A shared endpoint is useful only if the rest of the application remains ignorant of the serving provider. Store `label`, `confidence`, and `reason`; do not scatter vendor response types through the review queue. Run the same prompt and schema against every candidate, compare estimated costs before a high-volume rollout, and promote a model by changing tenant configuration after its evaluation passes.

This is where the routing approach earns its keep. The provider behind the capability can change without a code rewrite, while discovery exposes which model IDs are available. Its broader REST surface is useful for a solo product for a second reason — plain HTTP and consistent conventions reduce SDK and credential sprawl when adjacent backend work appears — but breadth should not decide the classifier evaluation. Label quality still does.

Ship the policy weekly. Outsource the plumbing.

## Reliability starts with honest boundaries

The catch is that an extra routing boundary has no revenue-per-hour value when switching is hypothetical. Stick with direct OpenAI when OpenAI-specific platform behavior is part of the product. Choose direct Anthropic Claude when the team has intentionally standardized on Claude, or direct Google Gemini when Gemini is already the fixed operational choice. Fewer layers make sense when the dependency itself is the decision.

This pattern is also not suitable when the requirement is a dedicated moderation service. It uses a general chat model plus `json_schema`, with human review retaining final authority. For large asynchronous datasets, evaluate a batch workflow instead of stretching a synchronous loop; OpenAI documents its Batch API separately.

I've made the decision rule narrow: choose a compatible router when provider replacement, stable parsing, and per-tenant attribution all belong in the acceptance test. Choose a direct provider when its unique integration matters more than portability. Either way, keep the review taxonomy, labeled evaluation set, and usage ledger inside the product. Those are differentiated work.

## Further reading

- [OpenAI Structured Outputs guide](https://platform.openai.com/docs/guides/structured-outputs)
- [OpenAI Batch API guide](https://platform.openai.com/docs/guides/batch)
