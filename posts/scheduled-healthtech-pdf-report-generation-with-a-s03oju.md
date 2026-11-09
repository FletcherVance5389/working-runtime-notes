# Scheduled Healthtech PDF Report Generation With API Template Approval Records

| Template editing responsibility | Rendering boundary | Main trade-off |
| --- | --- | --- |
| Founder reviews every layout change | Application-owned template passed to a rendering interface | Review stays beside application changes; the founder maintains print tests |
| Finance staff must edit layouts independently | Separately managed templates with an explicit approval step | Editing can move outside the release cycle; approved versions must remain identifiable |

TL;DR: For a one-person healthtech SaaS, keep the monthly report template in the application when you own its review. Schedule PDF generation through a job API, record the approved template version alongside the reporting snapshot, and archive the resulting bytes before marking the report complete. Choose separately managed templates when finance needs independent editing authority and can own an approval process.

The deciding question is who can approve a changed report. Rendering is work worth outsourcing when the boundary is clear. Template approval is a business responsibility that still needs an owner.

A finance team asking for a weekly business report and a healthtech application producing monthly archives need different reporting windows, but they can use the same scheduling contract. This note uses the monthly healthtech report as its concrete case. The schedule chooses the window; it mustn't silently choose new content for an already approved report.

## Who should approve templates for a scheduled PDF report generation API?

Start with editorial control. A column label, a rounding rule, and the position of an explanatory note can change how a reader interprets a financial report. Treat those changes as reviewable work. For the monthly healthtech report, an illustrative input might contain a reporting month, clinic identifiers, completed appointment counts, and invoiced totals expressed in minor currency units. These are example fields, not a prescribed healthcare reporting standard. Include only the data the report actually needs.

Keep the template reviewable.

An application-owned template is a sensible default if the founder already approves each layout change. Put it through the same review process as the code that prepares the report data. Record an immutable template revision in the job. A filename such as current-template is insufficient for this proposed contract because its contents can change between submission and execution. The same concern applies to a remotely managed template: a stable identifier alone is useful only if it resolves to the approved revision.

The trade-off is time. Every formatting request becomes engineering work, and a solo founder has limited review hours. Use a revenue-per-hour lens: compare the attention required for template changes, print testing, and renderer operation with the feature work that attention displaces. Ship weekly, but let approved monthly reports stay fixed while the application moves forward. Outsource the undifferentiated rendering work when doing so removes operations without removing your ability to review the template.

That choice is about responsibility. It doesn't require a particular rendering product or hosting arrangement (including when rendering runs outside the application).

## What exactly belongs in the monthly archive?

The second criterion is the evidence needed to explain a delivered report. Design the archive around the original PDF plus a manifest that records the reporting window, input snapshot reference, template revision, and generation attempt. If a correction is approved, create a new report revision with an explicit relationship to the earlier artifact. Don't replace the old bytes under an unchanged report identity.

Consider an illustrative September close: the snapshot contains 128 completed appointments and an invoiced total of 256000 minor units. Those numbers are invented test data. If an adjustment arrives after approval, rerunning the original report against a live query would mix the original approval with different inputs. Freezing the snapshot makes the intended behavior explicit. The correction gets its own revision and its own approval.

PDF has a formal format specification in ISO 32000-2. That fact alone doesn't establish a retention policy, prove that a report is accurate, or demonstrate that a particular archive meets a healthcare obligation. Define those acceptance requirements separately with the people responsible for them. If a particular archival profile is required, name it in the requirements and arrange validation against that profile; a PDF filename isn't that validation.

For this design, treat successful rendering and successful archiving as separate states. Persist the artifact and manifest before declaring completion. A timeout after storage can leave the caller uncertain about whether the operation finished, so require a lookup by job identity before scheduling another attempt. Reuse the approved inputs on a retry. A new reporting snapshot is a new revision, not a retry.

## A focused job contract for the reporting worker

In a Node.js application, the public API can accept a report request and return a job identity. Let a worker handle rendering and archiving. The following TypeScript example defines that worker boundary; its storage and rendering functions are application interfaces, not claims about any external API. It deliberately leaves scheduling and authentication to the calling application.

```ts
type ApprovedReport = {
  jobId: string;
  accountId: string;
  period: { startInclusive: string; endExclusive: string };
  snapshotId: string;
  templateRevision: string;
  approvalId: string;
  reportRevision: number;
};

type ArchiveReceipt = {
  artifactId: string;
  manifestId: string;
};

type ReportServices = {
  findCommitted(report: ApprovedReport): Promise<ArchiveReceipt | null>;
  loadApprovedInputs(report: ApprovedReport): Promise<{
    template: string;
    data: unknown;
  }>;
  render(input: { template: string; data: unknown }): Promise<Uint8Array>;
  validate(pdf: Uint8Array): Promise<void>;
  commitOnce(input: {
    identity: ApprovedReport;
    pdf: Uint8Array;
  }): Promise<ArchiveReceipt>;
};

async function archiveReport(
  report: ApprovedReport,
  services: ReportServices,
): Promise<ArchiveReceipt> {
  const existing = await services.findCommitted(report);
  if (existing) return existing;

  const input = await services.loadApprovedInputs(report);
  const pdf = await services.render(input);
  await services.validate(pdf);
  return services.commitOnce({ identity: report, pdf });
}
```

The important work sits behind these interfaces. Require loadApprovedInputs to verify that the approval covers this account, period, snapshot, and template revision. Require both findCommitted and commitOnce to reject reuse of a job identity with different report inputs. A committed receipt should be available only after both artifact and manifest are durable. Concurrency control must live in shared storage: the initial lookup cannot prevent two workers from rendering the same job simultaneously. Returning a stored receipt must still respect account access checks; possession of a job identity isn't authorization.

Duplicate computation may still happen.

That's an explicit trade-off in this small example. A worker lease could reduce redundant renders, but leases add recovery behavior that a solo operator must understand. Start by making duplicate completion safe. Add coordination when observed queue pressure or rendering work justifies it. Avoid promising that every render produces identical bytes; the archive preserves the accepted output, while any regeneration needs its own validation.

Define the monthly period using the business calendar and a named timezone before queueing the job. Store its resolved boundaries. Don't derive the window from the worker's current clock: a delayed job should still describe the month it was approved to report. For weekly finance reports, change the period calculation while preserving the same approval and archive contract.

## When independent template editing is the better choice

The runner-up becomes preferable when finance must revise layouts without waiting for an application release. Give that workflow an explicit draft, review, and approved state. The scheduler should select an approved revision, and historical reports should retain their recorded revision even after an editor publishes a newer one.

This shifts effort from implementing each layout change to maintaining permissions, previews, and approval records. It can be a good exchange if editing is frequent and finance has a clear owner. If every change still needs the founder to inspect the same output, moving the editor elsewhere may leave the review burden intact while adding another deployment boundary. Count review time before changing ownership.

## Release checks that protect the next close

Keep a small fixture set with an empty month, a long clinic name, and enough rows to force a second page. Include an adjustment case. Verify totals against the approved input and inspect headers, page breaks, and explanatory text in the rendered output. A file that opens can still be the wrong report. Require the validation interface to enforce the acceptance checks you actually defined; a nonempty byte array is too weak a completion rule.

Exercise failure recovery at the archive boundary as well. A useful test interrupts the caller after storage commits, then retries the same job and checks that the original receipt comes back. Another submits the same job identity with a changed snapshot and expects rejection. These checks protect the meaning of a monthly close more directly than a test that merely asserts the renderer was called.

For operations, record job identity, reporting period, template revision, attempt count, queue delay, render duration, and archive outcome. Keep patient details and full report payloads out of routine logs. Bound worker concurrency to protect the rest of the application, and make failed validation visible to the report owner before anything is delivered. Establish who may retrieve archived artifacts and how access is reviewed as part of the reporting requirements.

Own the approval. Preserve the evidence.

Keep the renderer replaceable through a narrow interface, and spend weekly engineering time on the parts of reporting that change the business outcome.

## References

- ISO 32000-2, Portable Document Format specification: https://www.iso.org/standard/75839.html
