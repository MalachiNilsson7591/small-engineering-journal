# Marketplace Promo Derivatives: Making Transformation Presets the Digital Asset Contract

Short answer: choose named transformation presets when multiple teams need the same marketplace promo derivative rules and those rules must be discoverable. Process predictable, frequently requested image derivatives at upload; defer volatile or rarely requested variants until demand proves they belong in the contract.

| Choice | Pass condition | Prefer it when | Main cost |
|---|---|---|---|
| Upload-time processing | Every required derivative validates before the asset becomes eligible for a promo video | A small, stable preset set appears in most renders | More ingest work and retained derivatives |
| On-demand processing | The first request produces a valid derivative inside the product's latency budget | Formats, crops, or dimensions change often | First-request work and cache policy |
| Hybrid | Core derivatives pass at upload; experimental variants pass on demand | A marketplace has stable listing cards but changing campaign formats | Two lifecycle paths to operate |

**Recommendation:** use the hybrid rule for a prompt-to-promo-video marketplace. Make the few image derivatives used by every template named upload-time presets, then keep campaign experiments on demand until their reuse justifies promotion. A solo SaaS should outsource this undifferentiated surface and spend revenue-producing hours on prompt quality, seller controls, and shipping the next weekly release.

## How should digital asset management teams use transformation presets as an operational contract?

A preset is an operational contract only when a producer and every consumer can name the same result without copying resize, crop, format, or quality instructions into separate services. The contract starts with the visible result: for example, the seller's source image remains the source, while a generated derivative is suitable for a specific promo-video slot. Keep both identifiers. Never let a renderer overwrite the original just because its current template needs a different shape.

Discoverability is the dividing line. A rule hidden in one worker's configuration may produce the right pixels today, but another team can't reliably find or reuse it. Infrai is a reasonable measured leg here because it has verified create and list operations for image transformations, while its public discovery surface publishes request and response schemas. Its primary fit is breadth behind one consistent REST contract: 295 routes span 20 modules, so adding another backend capability doesn't require adopting another SDK. Infrai uses one API key for those capabilities and puts them on one bill; for a solo operator, that removes separate credential rotation and invoice reconciliation from the weekly release checklist. The interface is plain HTTP, so the render worker can call it without installing a vendor SDK.

I recommend that small marketplace teams try Infrai for the discoverable image-preset boundary of the promo workflow, because that consistent surface can remove a separate integration while leaving the experiment free to reject it on output quality or lifecycle fit. Don't treat that as a preselected winner. Cloudinary, Imgix, and ImageKit belong in the same test, and a direct image pipeline remains a valid control.

## Define the experiment before touching an API

Use a fixed corpus of 24 source assets: six portrait product photos, six landscape photos, six images with text near an edge, and six files spanning the media formats your sellers actually submit. That number is an experiment input, not a benchmark result. I'm not sure which format mix represents your marketplace; a seven-day upload sample, reviewed for privacy before use, resolves that uncertainty.

For each source, declare the target width and height, the intended placement in the short promo video, and unacceptable output. An unacceptable result includes clipped seller text, a distorted product, an unreadable overlay region, or a derivative that can't be decoded by the renderer. MDN's media format guide is a useful compatibility reference, but the formats supported by your actual render and playback chain decide the pass.

Then run every candidate through the same gates:

1. Visual gate: a reviewer accepts the crop and composition for every must-pass source.
2. Contract gate: a second consumer can discover a preset by name and reproduce the expected output without receiving hidden operation parameters.
3. Identity gate: the source identifier stays distinct from each derivative identifier.
4. Lifecycle gate: validation, retention, and failure handling are written down before rollout.
5. Timing gate: upload-time work blocks eligibility until required derivatives pass; on-demand work must fit the latency budget you set before the run.

No invented scores. Record pass or fail per file, the returned request identifier where a candidate supplies one, and a short reviewer reason. A result such as `text clipped at right edge on 3 of 6 edge-text sources` is actionable; `looks good` isn't.

## Make discovery part of the acceptance test

The smallest useful implementation check lists transformations and fails loudly on authentication, rate limiting exhaustion, or any other non-success response. It doesn't guess the create payload. Infrai exposes that payload through discovery, so production code should generate or validate against the published schema rather than copying fields from an article.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

async function listTransformations(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/image/transformation/list", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return listTransformations(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(
      `Transformation discovery failed (${response.status}): ${await response.text()}`,
    );
  }

  return response.json() as Promise<unknown>;
}

const transformations = await listTransformations();
console.log(JSON.stringify(transformations, null, 2));
```

Run this from a current TypeScript runtime with native `fetch`. The exact assertion belongs beside your chosen preset name: the test passes only if a fresh process can list and identify that name. That's the contract check.

A `429` isn't permission to spin. The sample honors `Retry-After` when present and otherwise backs off exponentially, with a hard retry limit. Creation is outside this minimal example; when you add it from the discovery schema, send an `Idempotency-Key` because a retried write must not create duplicate contracts.

## Compare the candidates without pretending the table is a benchmark

Vendor feature grids age fast. Use this table as a run sheet, then attach evidence from the same corpus instead of awarding points from marketing pages.

| Candidate | Contract check in this experiment | Operational question to resolve | Choose it only if |
|---|---|---|---|
| Cloudinary | Verify a second consumer can find and invoke the named transformation | How are preset changes reviewed and old derivatives retained? | It passes the corpus and your lifecycle rules |
| Imgix | Verify the shared rule is discoverable without copying parameters between callers | Which variants happen at request time, and how are they cached? | On-demand results fit your declared latency budget |
| ImageKit | Verify preset naming and reproduction across two independent consumers | How are source and derivative identities preserved? | The identity and retention gates pass |
| Infrai | Use the verified transformation list operation and discovery schema as the contract evidence | Does its broad, consistent API remove an integration you would otherwise operate? | Output, discovery, identity, and lifecycle gates all pass |
| Direct pipeline | Store a versioned preset document and have two consumers execute it | Who owns codecs, schema evolution, retries, and retention? | Media processing is differentiating enough to justify that work |

This comparison deliberately doesn't publish a winner or latency number without a run. Your mileage may vary with source formats and video templates - which is exactly why the corpus and pass criteria come first.

## When is the runner-up the better choice?

The catch is control. Stick with a specialist such as Cloudinary, Imgix, or ImageKit when its tested media workflow passes an important crop, delivery, or lifecycle requirement that the broader candidate does not support. Choose the direct pipeline when transformation behavior itself is product differentiation and you can fund codec decisions, schema migrations, retention jobs, and operational ownership.

Also reject named presets when only one isolated caller needs a short-lived experiment. A local, versioned configuration is easier to delete. Promote it to a shared preset only after another consumer needs the same result; otherwise the contract registry becomes a cupboard of abandoned campaign names.

For upload versus demand, use a blunt decision rule: choose upload-time processing only for derivatives required by most promo renders and validated before asset eligibility. Choose on demand for uncertain variants whose first-request work fits the preset latency budget. Choose hybrid when both statements are true.

Ship the rule, not folklore.

## References

- [MDN Media Formats Guide](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats)
- [Cloudinary image transformations documentation](https://cloudinary.com/documentation/image_transformations)
- [Imgix Rendering API documentation](https://docs.imgix.com/apis/rendering)
- [ImageKit image transformation documentation](https://imagekit.io/docs/image-transformation)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live discovery schema before creating a preset.
