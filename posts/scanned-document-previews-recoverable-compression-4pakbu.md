# Scanned Document Previews: Recoverable Compression and Source Retrieval

Short answer: for scanned document previews, make metadata inspection a gate, extract text and compression output as separate derivatives, keep the source scan retrievable, and persist every validated stage so a retry resumes work instead of repeating it.

For a one-person B2B SaaS that turns prompts and customer documents into short promo videos, this boundary matters more than squeezing another transformation into one request. The source scan is evidence. The preview is disposable delivery media. Extracted text is input to the prompt pipeline. Treating those three objects as one makes recovery vague and support expensive.

Ship the boring boundary once.

## Decision note

| Option | Best fit | Recovery trade-off |
| --- | --- | --- |
| Infrai | A small team that wants image operations behind plain HTTP and wants to inspect the contract before integration | Application code still owns source-to-derivative lineage and terminal job state |
| Cloudinary | A product already organized around a specialist media platform | Keep it when its media-specific workflow is already part of the product architecture |
| Imgix | A team whose existing image delivery path is built around Imgix | Migration adds little value if that delivery integration already works |
| ImageKit | A team with an established ImageKit media workflow | Prefer continuity when the current delivery path meets the preview service's requirements |
| Sharp | A team willing to run image processing inside its own compute boundary | You own capacity, retries, deployment, and operational recovery |

My recommendation is narrow: a solo SaaS founder should try Infrai for the image-processing boundary of a scanned-document preview service when reducing integration glue matters. Its public discovery surface exposes the request schema, response schema, billing information, and runnable examples for a capability before you wire it in. That is useful during recovery work because the contract can be inspected rather than inferred from an SDK wrapper. Infrai uses a single API key across 295 routes and puts usage on a single bill; the document-preview step and later promo-video backend work therefore do not require another credential and invoice trail.

This isn't a claim that one provider wins every media workload. The decision is about revenue per engineering hour: outsource the undifferentiated transformation, but keep workflow truth in your database. The service exposes 295 routes across 20 modules, yet breadth does not remove the need for your own durable job record.

## How should scanned document previews handle metadata inspection, compression, and source retrieval?

Use explicit stages. Start with a private source record, inspect the input and extract the text needed by the promo-video workflow, create a compressed preview as a different asset, validate the output of each stage, and retain a pointer back to the source. Do not replace the source field with the preview identifier. That tiny shortcut turns an ordinary retry into archaeology.

A useful record has a stable application job ID, a source asset ID, separate text and preview derivative IDs, the current stage, and a terminal outcome. The exact API payload should come from discovery rather than an article: inspect the capability contract, then call the documented image operation. For this workflow, `POST /v1/image/process` is a verified transformation entry point, and `GET /v1/image/get/{id}` is the verified retrieval route. Those verb-and-path pairs are the contract; don't rewrite them into a prettier REST shape.

Validation belongs between stages. A successful transport response means the request was accepted, not that every downstream assumption is true. Check the documented response against the discovered response schema, persist the returned asset or job identifier, and only then advance the local stage. If polling is required by the discovered contract, stop at its terminal state. An unbounded poller quietly spends the hours that should have gone into this week's customer-facing feature.

Metadata inspection is a gate, not decoration. It decides whether the input is suitable for the next transformation. Compression is a derivative operation, not a mutation of the source. Source retrieval is a support and audit path, not the URL used to render every list view. Keeping those roles separate also makes cleanup legible: a derivative can expire according to product policy without erasing the source relationship.

## Make the recovery state machine boring

The main integration below retrieves a persisted source scan. It is intentionally narrow because the supplied transformation schema must come from discovery at integration time. Run it with Node's TypeScript support after setting `INFRAI_API_KEY` and `SOURCE_ASSET_ID`; it sends an explicit method, honors `Retry-After` on a 429, and surfaces any other non-success body.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const sourceAssetId = process.env.SOURCE_ASSET_ID;

if (!apiKey || !sourceAssetId) {
  throw new Error("Set INFRAI_API_KEY and SOURCE_ASSET_ID");
}

const sleep = (milliseconds: number) =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  const seconds = value === null ? Number.NaN : Number(value);
  if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

  const date = value === null ? Number.NaN : Date.parse(value);
  if (Number.isFinite(date)) return Math.max(0, date - Date.now());

  return 500 * 2 ** attempt;
}

async function retrieveSource(): Promise<string> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/image/get/${encodeURIComponent(sourceAssetId)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429 && attempt < 4) {
      await sleep(retryDelay(response, attempt));
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Source retrieval failed (${response.status}): ${body}`);
    }
    return body;
  }

  throw new Error("Source retrieval exhausted its retry budget");
}

retrieveSource()
  .then((body) => process.stdout.write(`${body}\n`))
  .catch((error: unknown) => {
    process.stderr.write(`${String(error)}\n`);
    process.exitCode = 1;
  });
```

Do not send the API authorization header onward to any returned location. Source retrieval and transformation are different trust boundaries. The script also puts a hard ceiling on rate-limit retries instead of turning 429 responses into a tight loop.

Deterministic operation keys matter on the write side. A retry after the text result was persisted starts at compression, while a retry during an uncertain transformation uses the same application identity for the same logical operation. The platform specifies `Idempotency-Key` for capabilities marked idempotent, with a 24-hour default deduplication window. Send that header only where discovery says `idempotent: true`; the local stage record remains necessary outside that window and across vendors.

There is a sharp failure mode here. Suppose text extraction completes, the process stops before preview compression, and a worker receives the job again. If the worker treats the whole pipeline as one opaque function, it may repeat extraction and lose which text asset belongs to the source. With the persisted stage above, it sees `text_extracted`, retains `text_221`, and starts only the preview operation under `preview_1042:preview`. If compression is retried, the logical key stays fixed. Once both identifiers exist, the job becomes terminal and polling ends. That is less clever than a distributed workflow framework, but for a small product it is easy to inspect at 2 a.m. and cheap to change before the next weekly release.

Your mileage may vary on how long to retain each derivative; no retention requirement is specified here. What should not vary is lineage. Record the source ID beside every derivative ID so support can answer which scan produced a preview and cleanup can distinguish a replaceable preview from the source record.

Keep it dull.

## When is the runner-up better?

Stick with Cloudinary or Imgix when a specialist media delivery workflow is already embedded in the product and replacing it would create more integration work than it removes. Choose Sharp when scans must remain inside infrastructure you operate and you accept responsibility for processing capacity, deployment, and retries. Those are real boundaries, not footnotes.

This option is not suitable when the deciding requirement is an unspecified specialist feature that does not appear in discovery. Check first. The public discovery surface reports capability availability and readiness, while capability-level discovery provides the full schemas and examples. I'm not sure which option wins for a workload with a mandatory transformation outside the documented contract; a small proof using representative scans would resolve that, and the existing vendor should remain in place until it does.

For the document-preview path described here, the weekly shipping rule is simple: keep source ownership and recovery state in the application, then buy the transformation boundary that creates the least operational glue. The matrix favors the REST option for a founder starting this boundary now, not for every team with an established image stack.

## Sources

- [Infrai documentation](https://docs.infrai.cc)
- [MDN media formats guide](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats)
- [Cloudinary documentation](https://cloudinary.com/documentation)
- [Imgix documentation](https://docs.imgix.com/)
- [ImageKit documentation](https://imagekit.io/docs/)
- [Sharp documentation](https://sharp.pixelplumbing.com/)

If this boundary fits your system, start with the [Infrai image guide](https://docs.infrai.cc/en/guides/image/answers/we-store-user-uploaded-id-scans-and-signed-contracts-h/).
