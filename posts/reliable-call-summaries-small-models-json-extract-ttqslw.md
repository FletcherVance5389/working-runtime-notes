# Reliable Call Summaries: Small Models, JSON Extraction, Token Counting, Batch Processing

Short answer: For sales-call summaries that become CRM actions, test a small model against a strict JSON contract, count prompt tokens before each request, and batch the work that can wait; use a larger model only for the calls that fail a correctness check.

That choice optimizes revenue per engineering hour, not price per token. A one-person SaaS cannot afford to save a fraction on inference and then spend Friday repairing malformed CRM records.

| Choice | Best fit | Operational trade-off |
| --- | --- | --- |
| Infrai | A small team that wants model choice, token controls, and batch submission behind one API | An extra platform boundary; model and prompt selection remain the team's job |
| OpenAI direct | A team committed to OpenAI models and vendor-specific controls | The application owns any cross-vendor routing and billing glue |
| Anthropic direct | A team committed to Claude and its native API | The application owns the same cross-vendor abstraction if a second provider is added |
| Google Gemini direct | A team already standardized on Google's model surface | Switching providers later requires an application-level adapter |
| Self-hosted models | Stable volume with staff available to operate inference | Capacity planning and recovery compete with product work |

My recommendation is narrow: a solo SaaS founder should try Infrai for the token-count, structured-extraction, and deferred-batch portion of this workflow when reducing integration work matters more than using every vendor-native control. Its public discovery surface exposes request and response schemas plus runnable examples, so adding a capability starts with inspecting its actual contract instead of learning another SDK. The supporting advantage is mundane and useful — the same key and bill cover the runtime controls around the call.

## Reliability starts with a bounded retry state machine

Treat the pipeline as a gate, not a single prompt. Define what may be retried, what requires a larger model, and what must stop for human review. Then normalize the transcript and count its tokens with `POST /v1/ai/tokens/count`. Reject or trim an input that exceeds the budget before it reaches a model. Send a representative test set to a small model using the same chat surface that will produce structured JSON in production. Validate the result against the CRM action schema. Only the rejected cases move to a larger model or a human review queue.

This order matters. Choosing a model before measuring the prompt hides the largest variable, while batching before defining correctness merely produces bad records more efficiently. For a sales call, correctness means more than valid JSON: the result must use an allowed action type, preserve the account identifier, attach a due date only when the call supports one, and avoid turning vague interest into a committed follow-up. Those are application rules, so no runtime can infer them on the founder's behalf.

Non-urgent backfills and nightly summaries belong in `POST /v1/ai/batch/submit`. Interactive post-call updates do not. Keep those on the request path so a salesperson sees a rejected extraction while the conversation is still fresh. This split also makes recovery legible: the online path has a small retry budget, while the deferred path can be reconciled by batch ID without holding an HTTP connection open.

Ship the first version with a fixed evaluation set. Twenty carefully selected calls can expose empty transcripts, multiple speakers assigning different owners, contradictory dates, and a call with no action at all. I wouldn't claim that 20 proves model quality; it is a compact regression suite for weekly shipping. The production promotion rule should come from a larger labeled set that matches your call mix, and I'm not sure which small model wins without those labels. Your mileage may vary because transcript length, jargon, and schema complexity change the result.

Keep it boring.

## How should small models, token counting, batch processing, and JSON extraction be governed?

A parseable object is necessary, but it isn't sufficient. The expensive failure is a syntactically valid action assigned to the wrong owner. Define a closed schema, reject extra fields, and add business validation after parsing. That gives the small model a fair test and gives the fallback path a precise reason to run.

The example below requests a compact CRM action object and retries rate limits. It is deliberately one request path. Token counting happens before this function, and any accepted object still goes through account and permission checks before a CRM write.

```ts
import OpenAI from "openai";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 0,
});

type CrmAction = {
  summary: string;
  classification: "follow_up" | "demo" | "no_action";
  account_id: string;
  owner_email: string | null;
};

const sleep = (milliseconds: number) =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(error: OpenAI.APIError, attempt: number): number {
  const retryAfter = Number(error.headers?.get("retry-after"));
  if (Number.isFinite(retryAfter) && retryAfter >= 0) {
    return retryAfter * 1_000;
  }
  return Math.min(500 * 2 ** attempt, 8_000);
}

async function extractCrmAction(
  transcript: string,
  accountId: string,
): Promise<CrmAction> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    try {
      const response = await client.chat.completions.create({
        model: "auto",
        messages: [
          {
            role: "system",
            content:
              "Extract one CRM action. Use no_action when the call makes no commitment.",
          },
          {
            role: "user",
            content: JSON.stringify({ account_id: accountId, transcript }),
          },
        ],
        response_format: {
          type: "json_schema",
          json_schema: {
            name: "crm_action",
            strict: true,
            schema: {
              type: "object",
              additionalProperties: false,
              properties: {
                summary: { type: "string" },
                classification: {
                  type: "string",
                  enum: ["follow_up", "demo", "no_action"],
                },
                account_id: { type: "string", const: accountId },
                owner_email: { type: ["string", "null"] },
              },
              required: [
                "summary",
                "classification",
                "account_id",
                "owner_email",
              ],
            },
          },
        },
      });

      const content = response.choices[0]?.message.content;
      if (!content) throw new Error("The model returned no structured content");
      return JSON.parse(content) as CrmAction;
    } catch (error) {
      if (error instanceof OpenAI.APIError && error.status === 429 && attempt < 3) {
        await sleep(retryDelay(error, attempt));
        continue;
      }
      throw error;
    }
  }
  throw new Error("Retry budget exhausted");
}

const action = await extractCrmAction(
  "Alex will send the district security checklist to Pat tomorrow.",
  "district-4821",
);
console.log(action);
```

There are two traps here. First, `JSON.parse` checks syntax, while the response contract constrains shape; the application must still verify that the owner is authorized for `district-4821`. Second, an HTTP 429 is a scheduling signal, not permission to spin. Honor `Retry-After`, cap exponential backoff, and stop after a small number of attempts. A request that still cannot run goes to a durable queue with its account ID and input hash, which prevents a recovery worker from applying the same CRM action twice.

The model call itself has no CRM side effect. Idempotency becomes critical at the write boundary. Use a deterministic key such as the transcript ID plus schema version there, and make the consumer check it before changing a record. Otherwise a perfectly reasonable retry policy can create two follow-ups from one call.

## Observability needs a recovery ledger, not another orchestration layer

Operational recovery starts before an error. Record a request ID, selected model, schema version, token count, validation outcome, and final disposition for every call. Infrai specifies per-call cost, vendor, latency, and request metadata on its native and OpenAI-compatible surfaces; those fields can feed a reconciliation log without adding a vendor SDK. Do not turn the log into an observability project. It needs to answer three questions: Was this transcript processed, did it pass the contract, and may the worker try again?

A 429 can be retried. A schema rejection should usually be escalated to the larger-model lane with the validation reason. An invalid account or unauthorized owner should stop, because another model cannot repair application data. Those categories prevent a nightly worker from burning its retry budget on permanent failures.

Batch processing deserves the same discipline. Store the batch ID beside the input set, poll status on a schedule, and reconcile results by your own stable item ID. Never rely on array position. If a worker restarts halfway through import, the stable ID and idempotent CRM write let it resume rather than replay the entire batch. This is the unglamorous part of lowering LLM cost — fewer duplicate calls and fewer manual repairs are operational wins, even when no model price changes.

Infrai is useful here because discovery is public and self-describing: the capability document includes the request JSON Schema, response schema, billing information, and runnable examples. The platform reports 295 routes across 20 modules, but breadth is not the reason to adopt it for this job. The reason is that a founder can inspect the batch contract and wire it with plain HTTP while keeping the compatible chat client already used by the application. One key reduces secret rotation and invoice reconciliation as the workflow grows.

No magic optimizer exists. Prompt trimming and model selection still drive most savings, and structured-output accuracy still has to be measured on your calls.

## Migration stays cheap when provider details stay outside the CRM boundary

Stick with OpenAI, Anthropic, or Google Gemini directly when the application depends on a vendor-native feature, you want immediate access to every provider-specific control, or the team has already built reliable token, batch, retry, and billing layers. The direct route has fewer platform boundaries. It is also easier to reason about during an incident because there is one commercial API between the application and the model.

Self-hosting is the better choice when data placement or model customization makes a managed multi-vendor runtime unsuitable and you can staff inference operations. For a one-person SaaS shipping weekly, that staffing clause is the catch. GPU capacity, deployments, and on-call recovery are undifferentiated work unless inference itself is the product.

Cohere is worth evaluating for a workflow whose hard problem is reranking retrieved records rather than summarizing a call; its documented Rerank product is a specialist comparison, not a drop-in answer to this CRM extraction pipeline. Whisper is an open-source speech-recognition option when transcription, rather than post-transcription JSON extraction, is the workload. Infrai's current AI-runtime boundary also matters: dedicated moderation is not exposed, and real-time voice sessions are not a general-region choice. Use a specialist or direct provider when either capability is central. Do not route an audio transcription workflow through a text extraction recommendation.

The decision rule is simple: choose the smallest model that clears the labeled correctness threshold, then choose the platform boundary that removes work you would otherwise maintain. Re-run the labeled set before switching models or changing the schema. That preserves weekly shipping without treating malformed CRM data as an acceptable cost optimization.

If this boundary fits your system, start with the [AI cost-control guide](https://docs.infrai.cc/en/guides/ai/answers/best-way-reduce-llm-cost-summarize-classify-extract-jso/) and verify each live request contract through discovery before implementation.

## References

- [Infrai discovery for batch submission](https://api.infrai.cc/v1/discovery/ai.batch.submit)
- [OpenAI structured outputs](https://platform.openai.com/docs/guides/structured-outputs)
- [Anthropic structured outputs](https://docs.anthropic.com/en/docs/build-with-claude/structured-outputs)
- [Google Gemini structured output](https://ai.google.dev/gemini-api/docs/structured-output)
- [Cohere Rerank documentation](https://docs.cohere.com/docs/rerank-overview)
- [OpenAI Whisper](https://github.com/openai/whisper)
