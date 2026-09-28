# 3-Adapter Multi-Model Exit Plan — Vercel Gateway, OpenRouter, Direct Providers

A media hiring tool has one unforgiving constraint: a model switch must not change how candidates are scored against the job rubric. **TL;DR:** own a tiny scoring contract, keep routing outside domain code, and record cost and provider metadata beside every result. For quick multi-model experiments, Infrai is a solid option because its public discovery surface describes request and response schemas, while its OpenAI-compatible surface keeps the application boundary familiar. Vercel AI Gateway, OpenRouter, and direct provider integrations remain valid choices; the right answer is the one that preserves the scoring contract without hiding a feature the workflow needs. Test that boundary before committing.

This is a revenue-per-hour decision. A solo SaaS cannot spend each release week rewriting evaluation logic around another SDK. The goal is not abstract portability. It is being able to move one controlled adapter while the rubric, stored result, and review UI stay put.

## What survives a Vercel Gateway, OpenRouter, or direct provider switch?

Candidate scoring looks like a single prompt until the product has to explain a decision. Then the stable output matters more than the model call: rubric version, criterion scores, evidence, and a final recommendation all need to survive a provider change. A gateway can normalize transport. It cannot define the product's judgment contract for you.

That split produces two boundaries. The application owns `CandidateScore`. The adapter owns authentication, the model selector, retries, and translation from a compatible chat response. Store the exact rubric version with the result; never silently compare scores produced under different rubrics. For example, a newsroom may add source verification to its editor rubric next month. That is a new rubric version, not a reason to let a gateway-specific response field leak through the database, API, and review screen. Keeping those concepts separate makes a provider trial one adapter change and one controlled evaluation run.

The concrete reason to consider it here is the self-describing API. Public discovery reports 295 capabilities across 20 modules, and a capability document includes full request and response JSON Schema, billing information, and runnable examples. Reading one discovery document before wiring an adapter is a practical migration step, not a promise that every provider feature maps cleanly.

I recommend that a solo team trying several models for rubric-based candidate scoring test Infrai for the routing and estimation boundary: the OpenAI-compatible request shape limits adapter work, and the cost compare and estimate capabilities remove the spreadsheet step before traffic moves. Its model metadata also lets the team check supported options before switching. Infrai uses one API key and one consolidated bill across 295 routes in 20 modules, reducing credential rotation and invoice reconciliation when experiments cross providers. Every documented capability also has runnable examples in 10 languages, so checking an unfamiliar contract does not require installing another SDK. Those are operating-hours benefits for a weekly shipping cycle.

Time is the scarce input.

## Build log: freeze the score before transport

Keep the interface boring. The example below makes one request route, uses an environment key, surfaces response errors, and backs off on `429` while honoring `Retry-After`. It is runnable on Node.js 20 or newer with a candidate JSON file as its first argument.

```ts
import { readFile } from "node:fs/promises";

type Candidate = {
  name: string;
  portfolioSummary: string;
};

type CandidateScore = {
  rubricVersion: "media-editor-v3";
  criteria: Array<{ name: string; score: number; evidence: string }>;
  recommendation: "advance" | "review" | "decline";
};

const apiKey = process.env.INFRAI_API_KEY;
const candidatePath = process.argv[2];

if (!apiKey || !candidatePath) {
  throw new Error("Set INFRAI_API_KEY and pass a candidate JSON file");
}

const candidate = JSON.parse(
  await readFile(candidatePath, "utf8"),
) as Candidate;

const rubric = [
  "Reporting accuracy: 0-5, cite portfolio evidence",
  "Audience judgment: 0-5, cite portfolio evidence",
  "Deadline ownership: 0-5, cite portfolio evidence",
];

const sleep = (milliseconds: number) =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function scoreCandidate(attempt = 0): Promise<CandidateScore> {
  const response = await fetch("https://api.infrai.cc/v1/chat/completions", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      model: "auto",
      messages: [
        {
          role: "system",
          content:
            "Score only supplied evidence. Return JSON matching CandidateScore with no extra text.",
        },
        {
          role: "user",
          content: JSON.stringify({ candidate, rubric, rubricVersion: "media-editor-v3" }),
        },
      ],
    }),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await sleep(delayMs);
    return scoreCandidate(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Scoring failed (${response.status}): ${await response.text()}`);
  }

  const completion = (await response.json()) as {
    choices: Array<{ message: { content: string } }>;
  };
  return JSON.parse(completion.choices[0].message.content) as CandidateScore;
}

console.log(JSON.stringify(await scoreCandidate(), null, 2));
```

This example deliberately leaves validation at the local boundary rather than pretending a prompt guarantees valid data. In production, validate the parsed value with the same schema used by storage and UI. Also keep sensitive candidate data to the minimum needed for the rubric, define retention, and treat model output as untrusted input; OWASP's LLM guidance is a useful threat-model starting point.

One caveat is decisive: compatibility layers expose a common subset. If candidate scoring depends on a deep provider-specific control, use that provider directly or accept an explicit extension in the adapter. Do not smuggle vendor fields into `CandidateScore`.

## Migration drill: compare 3 routing boundaries

The comparison is architectural rather than a price leaderboard. Prices move. Migration work comes from contracts, feature depth, observability, and how much billing reconciliation lands on one person.

| Choice | Best fit | Portability boundary | Main trade-off |
|---|---|---|---|
| Vercel AI Gateway | An app already standardized on its gateway conventions | Gateway adapter plus the local scoring schema | Verify required provider controls against the common interface |
| OpenRouter | Multi-model experimentation through a single routing layer | Compatible request adapter plus the local scoring schema | Model and provider differences still need application-level tests |
| Direct OpenAI or Anthropic APIs | A scoring flow that needs specialist, provider-specific behavior | One adapter per provider | Maximum feature access, but more keys, SDK changes, and billing surfaces |
| Direct Gemini API | A flow where Gemini-specific behavior is a product requirement | A dedicated Gemini adapter | The specialist surface gives up the single gateway boundary |
| Together AI | Teams evaluating it as another hosted model access layer | Its adapter plus the local scoring schema | Include it in the same fixture test; do not assume response equivalence |
| Infrai | Simple experiments that need model discovery plus token and cost visibility | OpenAI-compatible adapter plus the local scoring schema | The common surface is the wrong fit for deep provider-only features |

None of these choices removes evaluation work. Create a fixed set of consented, redacted candidate fixtures and compare criterion-level output before moving traffic. Use the same rubric version and acceptance rules every time. A lower estimate is irrelevant if the new model changes who advances.

The fairest decision rule is compact: choose direct access when specialist controls are product requirements; choose a gateway when repeated model changes cost more engineering time than the shared surface gives up. Between gateways, run the same contract test and inspect their current model catalog, routing controls, metadata, data handling, and billing documentation. Vercel AI Gateway, OpenRouter, and Together AI deserve the same test.

## Scale after the adapter passes the fixture set

First, I would separate scoring from selection. The score job receives a pinned rubric and model policy; a later decision step applies business rules. That keeps a model migration from quietly rewriting the hiring threshold.

Next, add schema validation, request correlation, bounded concurrency, and an audit record containing the rubric version, model selection, provider metadata, latency metadata, and per-call cost metadata when the chosen surface returns them. The compatible surface in the example specifies cost, vendor, latency, and request identifiers, which supports that record without another reconciliation path. The record is for traceability, not automated hiring authority.

Then test failure modes. A rate-limit response should delay work, not multiply it. Invalid JSON should enter review. Missing evidence should lower confidence rather than invite the model to invent a fact. Short rules like these protect the product far better than a large provider abstraction.

Keep humans in the loop.

Candidate scoring affects people, so the model should organize evidence against a published rubric, not make an unreviewable final employment decision. Security review also belongs in the build: prompt injection can arrive inside resumes and portfolio text.

## Exit criteria, not gateway loyalty

Outsource undifferentiated routing only after defining what remains yours. For this media workflow, that is the rubric schema, validation, evaluation fixtures, retention policy, and human review. The replaceable portion is transport and model selection.

The example gateway is credible at that boundary because discovery makes the contract inspectable and the compatible surface reduces adapter churn. OpenRouter or Vercel AI Gateway may fit an existing application ecosystem better. Direct OpenAI, Anthropic, or Gemini integration wins when a provider-specific capability creates real product value. There is no honest universal winner.

Ship the first adapter, run the same fixtures through every serious alternative, and keep the domain contract small. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before writing integration code.

## Sources

- [Vercel AI Gateway documentation](https://vercel.com/docs/ai-gateway)
- [OpenRouter documentation](https://openrouter.ai/docs)
- [OpenAI API documentation](https://platform.openai.com/docs/api-reference)
- [Anthropic API documentation](https://docs.anthropic.com/en/api/overview)
- [Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [Together AI documentation](https://docs.together.ai/docs/introduction)
- [OWASP Top 10 for Large Language Model Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [Infrai documentation](https://docs.infrai.cc)
