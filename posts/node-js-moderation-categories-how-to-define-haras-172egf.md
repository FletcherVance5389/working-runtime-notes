# Node.js Moderation Categories: How to Define Harassment and Violence for Startup Apps

TL;DR: Define moderation labels around the action your app will take, then map every classifier or rules engine into that small internal contract. For a game catalog, use seven top-level risks: harassment, sexual content, self-harm, violence, illegal activity, spam, and PII. Do not treat them as seven equivalent booleans. Context, severity, confidence, and the affected field decide whether a listing is published, held, redacted, or escalated.

The deciding constraint is operator time. In a one-person SaaS, a label that does not change an action creates review work without protecting a user. I want a taxonomy I can inspect in one minute, ship this week, and keep when the underlying model changes.

## How should a startup app define moderation categories for harassment?

A moderation category describes content. An action resolves a product decision. Those are related, but they are not interchangeable. A fantasy game may mention a sword fight in its description; a seller may paste a phone number into support text; a spammer may repeat a harmless-looking download link across 200 listings. The words alone do not determine the response.

Start with four outcomes: `allow`, `redact`, `review`, and `block`. Keep the irreversible outcome rare. A low-confidence signal should usually enter a review queue, while an exact PII detector can redact the matched span without hiding an otherwise valid listing. Immediate danger language needs an escalation path outside ordinary catalog review. Ranking it beside promotional spam loses the urgency encoded by the context.

This leads to a compact record rather than a flat list of flags:

```ts
type Category =
  | "harassment"
  | "sexual"
  | "self_harm"
  | "violence"
  | "illegal"
  | "spam"
  | "pii";

type Severity = "low" | "medium" | "high" | "critical";
type Action = "allow" | "redact" | "review" | "block";

interface Finding {
  category: Category;
  severity: Severity;
  confidence: number;
  field: "title" | "description" | "supportText";
  spans: Array<{ start: number; end: number }>;
  reasonCode: string;
}

interface Decision {
  action: Action;
  findings: Finding[];
  taxonomyVersion: "catalog-v1";
}
```

The `reasonCode` is intentionally machine-sized, such as `PII_EMAIL` or `SPAM_REPEATED_LINK`. Put reviewer prose in a separate explanation field if needed. Stable codes make dashboards and appeals less fragile than free-form model text.

This approach has limits. An action-centered taxonomy compresses nuance, and a single final action can hide conflicts between categories. It is a poor fit when a qualified specialist must make an immediate welfare or legal judgment; route those cases to the relevant emergency, safety, or legal process instead. Rules-only detection is easier to audit but misses contextual language. Model-based detection handles more context but adds nondeterminism and needs regression testing. That trade-off is why the adapter preserves findings instead of storing only the final action.

## Build the smallest useful taxonomy

Write one sentence for inclusion and one for exclusion before writing prompts. That exercise exposes overlaps early. Harassment covers targeted abuse or degradation; it excludes ordinary criticism of a game. Sexual covers explicit sexual material or sexual exploitation; it should not automatically absorb non-explicit romance. Self-harm covers encouragement, instructions, intent, or a credible threat involving harm to oneself. Violence covers threats, praise, or graphic depiction of harm to others, while fictional combat still needs contextual treatment.

Illegal activity is the easiest bucket to make useless. Narrow it to content that facilitates a prohibited transaction or gives operational instructions for wrongdoing under the rules that apply to your service. Do not ask a text classifier to settle jurisdiction. Route uncertain legal-policy matches to a person. Spam covers deceptive or repetitive promotion, manipulation, and bulk abuse patterns; it often depends on account and repetition signals absent from one description. PII covers personal data your product has decided should not appear publicly, with exact subtypes such as email, phone, address, or account identifier. NIST's definition of PII is a sound starting point, but your data inventory and exposure policy must make it concrete.

The category is only the first coordinate. Add severity criteria that reviewers can observe:

| Risk | Lower-severity example | High or critical trigger | Default action |
|---|---|---|---|
| Harassment | Insult without a credible threat | Targeted threat or sustained degrading attack | Review |
| Sexual | Non-explicit adult reference | Exploitation or explicit prohibited material | Block and escalate |
| Self-harm | Fictional mention | Encouragement, instructions, or credible imminent intent | Review or escalate |
| Violence | Fictional combat | Credible threat or graphic real-world harm | Review or block |
| Illegal | Vague reference | Actionable facilitation under service policy | Review |
| Spam | One promotional phrase | Repeated deceptive campaign pattern | Review or block |
| PII | Approved business contact field | Personal data in a public free-text field | Redact or review |

These examples are policy scaffolding, not universal law. The important part is the decision boundary. Store it in a versioned policy document so an old decision can be interpreted after `catalog-v2` ships.

## Implement a neutral adapter in Node.js

Keep external output at the edge. The rest of the application should receive only `Finding[]`. A model or rules service can do undifferentiated classification work, while the product owns the consequential decision.

The following TypeScript runs on Node.js 20 or later. It validates untrusted JSON, rejects unknown labels, clamps confidence, and applies deterministic action rules. Replace `classify` with any HTTP client, local model, or rules pipeline that returns the same unknown value.

```ts
const categories = [
  "harassment", "sexual", "self_harm", "violence",
  "illegal", "spam", "pii",
] as const;
const severities = ["low", "medium", "high", "critical"] as const;

type Category = (typeof categories)[number];
type Severity = (typeof severities)[number];
type Action = "allow" | "redact" | "review" | "block";

type Finding = {
  category: Category;
  severity: Severity;
  confidence: number;
  field: "title" | "description" | "supportText";
  spans: Array<{ start: number; end: number }>;
  reasonCode: string;
};

const isOneOf = <T extends readonly string[]>(
  values: T,
  value: unknown,
): value is T[number] => typeof value === "string" && values.includes(value);

function parseFindings(value: unknown): Finding[] {
  if (!Array.isArray(value)) throw new Error("Classifier output must be an array");

  return value.map((item): Finding => {
    if (typeof item !== "object" || item === null) throw new Error("Invalid finding");
    const row = item as Record<string, unknown>;
    if (!isOneOf(categories, row.category)) throw new Error("Unknown category");
    if (!isOneOf(severities, row.severity)) throw new Error("Unknown severity");
    if (!isOneOf(["title", "description", "supportText"] as const, row.field)) {
      throw new Error("Unknown field");
    }
    if (typeof row.confidence !== "number" || !Number.isFinite(row.confidence)) {
      throw new Error("Invalid confidence");
    }
    if (typeof row.reasonCode !== "string") throw new Error("Invalid reason code");

    const spans = Array.isArray(row.spans)
      ? row.spans.flatMap((span) => {
          if (typeof span !== "object" || span === null) return [];
          const pair = span as Record<string, unknown>;
          return Number.isInteger(pair.start) && Number.isInteger(pair.end)
            ? [{ start: pair.start as number, end: pair.end as number }]
            : [];
        })
      : [];

    return {
      category: row.category,
      severity: row.severity,
      confidence: Math.max(0, Math.min(1, row.confidence)),
      field: row.field,
      spans,
      reasonCode: row.reasonCode,
    };
  });
}

function decide(findings: Finding[]): Action {
  const credible = findings.filter((finding) => finding.confidence >= 0.7);
  if (credible.some((finding) =>
    finding.severity === "critical" && finding.category !== "pii"
  )) return "block";
  if (credible.some((finding) => finding.category === "pii" && finding.spans.length > 0)) {
    return "redact";
  }
  if (credible.some((finding) =>
    finding.severity === "high" || finding.severity === "critical"
  )) return "review";
  return "allow";
}

async function classify(_text: string): Promise<unknown> {
  return [{
    category: "pii",
    severity: "high",
    confidence: 0.98,
    field: "supportText",
    spans: [{ start: 15, end: 33 }],
    reasonCode: "PII_EMAIL",
  }];
}

const raw = await classify("Questions? Try player@example.com");
const findings = parseFindings(raw);
console.log({ action: decide(findings), findings });
```

A threshold of `0.7` is a placeholder, not a universal operating point. Choose thresholds from labeled catalog data and the cost of each error. Missing a credible self-harm threat and delaying an innocuous listing do not have the same impact, so one global threshold is usually a poor final policy.

Fail into review when the response is malformed. Do not convert a parser exception into `allow`. Preserve the original text, normalized text, taxonomy version, detector version, and final human action in an access-controlled audit record. Keep raw PII out of logs; store spans or irreversible tokens when those are enough for debugging.

## Test decisions, then ship in shadow mode

A useful test set contains hard boundaries from the actual catalog: fictional violence versus a credible threat, a romance plot versus explicit sexual content, a player-support address versus somebody else's private address, and one promotion versus a repeated campaign. Include clean listings. Otherwise a system can look accurate by finding risks in a risk-only fixture set.

For each case, assert the final action, required category, and forbidden category. Run the same fixtures through every candidate backend. That makes provider portability an ordinary contract test rather than a rewrite. Track false-negative and false-positive rates per category and severity; one aggregate accuracy number hides the error that matters.

Ship weekly, but use a shadow phase. Record proposed decisions without enforcing them, sample disagreements for human review, then enable redaction or blocking category by category. Monitor category rate, review-queue age, override rate, malformed responses, and the share of traffic receiving `allow`. Sudden movement is a release signal, not proof that users changed overnight.

Appeals belong in the data loop. An overturned block should become a labeled boundary case after sensitive material is removed. A confirmed miss should too. This produces a small, relevant regression suite instead of a giant generic benchmark that never resembles game listings.

## What I would change at scale

At low volume, one synchronous classification call plus deterministic rules is easy to reason about. At higher volume, separate ingestion from enforcement with an idempotent job identifier, bounded retries, and a dead-letter path. Batch processing can reduce request overhead for work that does not need an immediate answer, but it changes latency and recovery behavior; the OpenAI Batch guide is one public example of that interface pattern, not a reason to couple the taxonomy to it.

Split detectors by evidence type. Regex and checksums suit structured identifiers. Account velocity and duplicate hashes help with spam. A language model can classify contextual threats or fictional violence. Audio listings need transcription before the same text contract can apply; Whisper is one open-source reference implementation for speech recognition. Each component should emit the internal finding shape, with provenance attached.

More machinery has a cost. Do not add a queue, a feature store, and three classifiers because a diagram looks mature. Add a component when it removes a measured review bottleneck or catches a documented miss. Revenue per engineering hour still counts: the right moderation system for a solo SaaS is the smallest one whose failures are visible, reversible where possible, and routed to a human when context carries the decision.

## References

- NIST, Personally Identifiable Information: https://csrc.nist.gov/glossary/term/personally_identifiable_information
- NIST, AI Risk Management Framework: https://www.nist.gov/itl/ai-risk-management-framework
- OWASP, Logging Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- OpenAI, Batch API guide: https://platform.openai.com/docs/guides/batch
- openai/whisper, open-source speech recognition: https://github.com/openai/whisper
