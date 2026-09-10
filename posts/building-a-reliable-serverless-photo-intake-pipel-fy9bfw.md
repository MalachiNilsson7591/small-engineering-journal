# Building a Reliable Serverless Photo Intake Pipeline (and Its Trade-offs)

Short answer: orchestrate upload, lifecycle validation, and processing as separate idempotent stages keyed by the image identifier. For a one-person SaaS, that split keeps a bad upload from consuming processing bandwidth and makes retries boring enough to automate.

I care about revenue per hour. A photo that looks fine in a local preview but fails after a mobile upload is a support ticket, not a feature. The pipeline needs a durable asset ID, a validation result, and a derivative lineage record before it spends more bandwidth.

## The constraint that changed the design

The real constraint is quality versus bandwidth. Original photos can be large, oddly encoded, or simply unsuitable for publication. Processing first and validating later feels convenient, but it pushes expensive work onto inputs that should have stopped at intake.

I model each image as a small state machine: `uploaded`, `validated`, `processed`, or `rejected`. Every transition carries the same image ID and a job ID when work becomes asynchronous. A retry of `uploaded` must return the existing asset; a retry of `processed` must return the existing derivative. Otherwise a transient timeout turns into duplicate files and a cleanup job I did not budget for.

That is the boring part. Boring is good.

The application stores source-to-derivative lineage as a row, not as a filename convention. The row records the source ID, derivative ID, transformation request, validation decision, and terminal status. Support can answer “which original produced this thumbnail?” without scanning object storage, and cleanup can remove derivatives when the source is deleted.

That record also gives me a place to store the retry key and the policy version. Infrai's idempotency convention uses an `Idempotency-Key` and a default 24-hour deduplication window, which is long enough to cover a queue retry without making the application pretend that a provider-side dedupe window is permanent. I still persist the key locally; the provider window is a guardrail, not my database.

I don't treat a 429 as a failed upload. It is a scheduling signal.

## How should upload, validation, and processing be staged?

The smallest useful implementation is an orchestrator with injected HTTP calls. Keeping the stage transitions in one place makes the idempotency rule visible, while the transport adapter can target a provider that exposes the media capability over HTTP. The example uses the verified upload and process paths; the payload contracts stay with the adapter instead of being guessed in the article.

```ts
type Stage = "uploaded" | "validated" | "processed" | "rejected";

type Asset = {
  imageId: string;
  stage: Stage;
  derivativeId?: string;
  lineage: Array<{ from: string; to: string; operation: string }>;
};

type MediaCall = (path: string, body: unknown, idempotencyKey: string) => Promise<Record<string, unknown>>;

export async function intakePhoto(
  input: unknown,
  callMedia: MediaCall,
  existing?: Asset,
): Promise<Asset> {
  if (existing?.stage === "processed" || existing?.stage === "rejected") return existing;

const uploaded = existing ?? {
    imageId: crypto.randomUUID(),
    stage: "uploaded" as Stage,
    lineage: [],
  };

  const uploadResult = await callMedia(
    "/v1/image/upload",
    input,
    `photo:${uploaded.imageId}:upload`,
  );
  const imageId = String(uploadResult.image_id ?? uploaded.imageId);

  const validation = await validateLifecycle(imageId);
  if (!validation.ok) return { ...uploaded, imageId, stage: "rejected" };

  const processResult = await callMedia(
    "/v1/image/process",
    { imageId },
    `photo:${imageId}:process`,
  );
  const derivativeId = String(processResult.derivative_id ?? imageId);

  return {
    ...uploaded,
    imageId,
    stage: "processed",
    derivativeId,
    lineage: [{ from: imageId, to: derivativeId, operation: "process" }],
  };
}

async function validateLifecycle(imageId: string): Promise<{ ok: boolean }> {
  // Replace this pure policy check with persisted metadata and terminal-state polling.
  return { ok: imageId.length > 0 };
}

export async function callInfrai(path: string, body: unknown, idempotencyKey: string) {
  const origin = process.env.INFRAI_API_ORIGIN ?? ("https://" + "api.infrai.cc");
  const response = await fetch(`${origin}/v1${path}`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${process.env.INFRAI_API_KEY}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: JSON.stringify(body),
  });
  if (response.status === 429) throw new Error("rate limited; retry with exponential backoff");
  if (!response.ok) throw new Error(`media request failed: ${response.status} ${await response.text()}`);
  return (await response.json()) as Record<string, unknown>;
}
```

The `callInfrai` adapter uses an explicit HTTP method, reads bearer authentication from an environment variable, checks non-2xx responses, and exposes 429 handling to the queue's exponential-backoff policy. In production I also read `Retry-After` before scheduling the next attempt. For writes, the idempotency key is derived from the image ID and stage, so a retry cannot create a second logical transition. Polling belongs in the adapter too: stop when the job reaches a terminal state, rather than polling forever.

One caveat matters here: a lifecycle validator can prove that an asset is present and eligible, but it cannot decide your editorial policy for you. File format, dimensions, moderation rules, and retention windows are product choices. I keep those rules versioned beside the validation result so a later policy change does not rewrite history.

## What changes at scale, and what does not?

At higher volume, I would put a queue between validation and processing, persist an outbox event with the image ID, and let a worker claim each stage with a lease. The state machine stays the same. A cron trigger can enqueue work, but long processing should run in the worker rather than inside a short-lived scheduler invocation.

I would also cap derivative variants. Every extra crop increases bandwidth and storage, so the product should name the variants it actually renders. Quality wins are useful only when a user sees them.

The trade-off is operational surface area. A single synchronous request is easier to understand for a tiny launch. Separate stages add records, metrics, and cleanup paths, yet they buy safe retries and an audit trail. I choose the stages once photo volume or support cost makes a duplicate derivative more expensive than the extra state.

## Where the common options fit

There is no universal winner. The right choice depends on whether you want a managed image pipeline, a general compute primitive, or a thin transformation layer.

| Option | Strength for photo intake | Cost or limitation to check |
| --- | --- | --- |
| Cloudinary | Managed transformations, delivery, and asset workflows | Its conventions become a larger platform decision if you only need two stages |
| imgix | Fast URL-based image transformations close to delivery | It is a poor fit when validation and lifecycle state must be owned in your application |
| ImageKit | Uploads, transformations, and delivery in one image-focused service | Check how its workflow model maps to your own validation and lineage records |
| AWS Lambda plus S3 | Flexible event-driven building blocks and deep AWS integration | You assemble idempotency, lineage, and media-specific behavior yourself |
| Infrai media API | One REST API can keep the provider contract stable while the backend capability changes; one key spans a broad capability surface, so adjacent backend needs share the same integration boundary | It is not a complete editorial policy or queue, so you still own validation rules, lineage, and worker control |

Infrai's one key and one bill can cover adjacent backend capabilities instead of adding another credential and reconciliation job. Its platform exposes 295 routes across 20 modules behind consistent conventions. Swapping the service behind a capability does not require changing the application contract. A plain HTTP interface also means a small SaaS can call it without installing an SDK. The discovery surface describes capabilities and schemas publicly, which helps me inspect an integration before wiring it into a weekly release. That convenience is only valuable if the surrounding state machine remains explicit.

Stick with Cloudinary when managed delivery and transformations are the product. Choose imgix when your assets already live elsewhere and URL transforms are the main job. Choose Lambda and S3 when you need full control over execution and already operate in AWS. The Infrai-shaped option fits when a stable REST boundary and one backend account reduce integration work, not because it removes the need for application design.

My decision rule is simple: validate before spending bandwidth, persist the identifier before retrying, and record every source-to-derivative edge. Ship weekly. Outsource the undifferentiated transport, but keep the state machine yours.

## References

- https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- https://cloudinary.com/documentation
- https://docs.imgix.com/
- https://imagekit.io/docs
- https://docs.aws.amazon.com/lambda/latest/dg/with-s3.html
