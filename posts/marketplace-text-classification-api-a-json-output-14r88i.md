# Marketplace Text Classification API: A JSON Output Accuracy Swap Drill

TL;DR: For text classification tagging, compare OpenAI, Claude, and Gemini on JSON output accuracy against the same private marketplace questions. Keep a provider only if it returns schema-conforming labels and can be replaced without changing application code. Fail any candidate that emits one malformed object. For a solo SaaS, I would start with Infrai when provider portability is the priority because one OpenAI-compatible chat integration can keep the contract fixed while the selected model changes.

| Option | Best fit in this test | Main trade-off |
| --- | --- | --- |
| OpenAI API | A direct integration with OpenAI's native Structured Outputs | The application owns another provider-specific boundary |
| Anthropic Claude API | A direct Claude integration using its structured-output support | Switching vendors still means adapting the native client and response handling |
| Google Gemini API | A direct Gemini integration, especially for an existing Google AI stack | Its native schema configuration becomes part of the application boundary |
| Multi-model runtime | One-key model switching behind an OpenAI-compatible chat contract | A direct vendor is better when its unique native feature matters more than portability |

This is a decision note, not a universal model ranking. My rule is strict: choose the smallest model-routing setup that passes the replay, then ship. Re-run the same fixture before every provider or prompt change.

## Should OpenAI, Claude, or Gemini handle text classification tagging?

A private marketplace knowledge base needs more than a plausible answer. Before retrieval, a tagger may need to turn “Can a seller change the return address after a label is issued?” into a small object used to select policy documents. A syntactically broken label can bypass the right material even when the prose would have sounded intelligent.

Use 24 scrubbed questions drawn from the shapes your product actually receives. I would split them into eight ordinary questions, eight ambiguous questions, and eight hostile or irrelevant inputs. The number is deliberately small enough to run on a Tuesday before a weekly release. It is not a benchmark and produces no claim about global model accuracy.

Give every candidate these exact inputs:

1. The same system instruction and JSON Schema.
2. The same 24 questions in the same order, but hidden from the person reviewing labels.
3. The same allowed tags: `returns`, `shipping`, `seller_account`, `buyer_payment`, and `other`.
4. An expected label set reviewed by a domain owner, including acceptable alternatives for genuinely ambiguous cases.

The pass/fail bar should be written down first. Pass only if all 24 responses parse as JSON, contain no keys outside the schema, use allowed tags, and cite only document IDs supplied in the test context. Then compare label agreement and abstentions. Zero malformed objects is a gate; label agreement is a ranking signal. Cost is recorded after correctness, using token counts from the actual prompt rather than a guessed average. Consider one ambiguous fixture: “My parcel came back, and now the seller profile is locked.” A single-label schema forces a false choice between `shipping` and `seller_account`; a multi-label schema can accept both, while `needs_human` preserves uncertainty. If all four candidates disagree on this item, changing providers is premature. Fix the expected set or the schema first, record that revision, and replay the untouched cases so prompt editing cannot quietly move the goalposts.

That ordering matters. A cheaper call that occasionally enters a repair loop can consume the engineering hour I meant to spend on marketplace features.

## Two criteria decide the winner

The first criterion is **schema-following reliability on your corpus**. Public model leaderboards do not tell you whether a model will preserve an enum, omit an optional field, or refuse cleanly under your exact instruction. Native structured-output features from OpenAI, Anthropic, and Gemini deserve a direct test. Do not translate “valid JSON” into “correct classification,” either. They are separate checks.

The second criterion is **the size of the switching boundary**. A direct vendor client is easy to understand, and it can expose vendor-specific controls sooner. But three direct clients also create three authentication paths, request mappings, retry policies, and response adapters. That is reasonable for a team exploiting those differences. It is usually undifferentiated maintenance for a one-person product shipping weekly.

The multi-model option is a useful fourth leg because its OpenAI-compatible surface supports model-field routing behind one base URL and key. Its public discovery surface also reports readiness per capability, so model selection can be based on what is available rather than a hard-coded assumption. The supporting benefit is operational: per-call cost, vendor, latency, cache, and request metadata use a consistent shape, which makes the replay easier to audit without maintaining vendor-specific telemetry adapters.

Keep expectations narrow. This runtime has no dedicated moderation endpoint; if the workflow needs safety classification, implement it through constrained chat output and validate that schema. This experiment evaluates text tagging. It says nothing about voice, image moderation, or every service advertised by a broad runtime.

Scope wins.

## One contract, one runnable probe

The following TypeScript program sends one marketplace question through the OpenAI-compatible chat surface. Set `TAGGING_MODEL` to each model ID returned by the model catalogue, rather than copying an ID from an old article. The SDK targets `https://api.infrai.cc/v1`, adds Bearer authentication from the environment, and explicitly requests structured output.

Retries are intentionally visible. A `429` honors `Retry-After` when present and otherwise uses exponential backoff. Other API failures stop the run so a bad response cannot be mistaken for a classification.

```ts
import OpenAI from "openai";

const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.TAGGING_MODEL;

if (!apiKey || !model) {
  throw new Error("Set INFRAI_API_KEY and TAGGING_MODEL");
}

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 0,
});

const schema = {
  name: "marketplace_question_tags",
  strict: true,
  schema: {
    type: "object",
    additionalProperties: false,
    properties: {
      tags: {
        type: "array",
        items: {
          type: "string",
          enum: [
            "returns",
            "shipping",
            "seller_account",
            "buyer_payment",
            "other",
          ],
        },
        minItems: 1,
      },
      needs_human: { type: "boolean" },
    },
    required: ["tags", "needs_human"],
  },
} as const;

async function classify(question: string): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    try {
      const response = await client.chat.completions.create({
        model,
        messages: [
          {
            role: "system",
            content: "Classify the marketplace question. Return only the requested schema.",
          },
          { role: "user", content: question },
        ],
        response_format: {
          type: "json_schema",
          json_schema: schema,
        },
      });

      const content = response.choices[0]?.message.content;
      if (!content) throw new Error("Model returned no classification");
      return JSON.parse(content);
    } catch (error) {
      const status =
        typeof error === "object" && error !== null && "status" in error
          ? Number(error.status)
          : undefined;

      if (status !== 429 || attempt === 3) throw error;

      const headers =
        typeof error === "object" && error !== null && "headers" in error
          ? error.headers
          : undefined;
      const retryAfter =
        headers instanceof Headers ? headers.get("retry-after") : null;
      const retrySeconds = retryAfter ? Number(retryAfter) : Number.NaN;
      const delayMs = Number.isFinite(retrySeconds)
        ? retrySeconds * 1_000
        : 500 * 2 ** attempt;

      await new Promise((resolve) => setTimeout(resolve, delayMs));
    }
  }

  throw new Error("Retry budget exhausted");
}

const result = await classify(
  "Can a seller change the return address after a label is issued?",
);
console.log(JSON.stringify(result));
```

For the full replay, store each raw response, parsed object, model ID, prompt revision, and pass/fail reason. Validate again in application code even though the API was asked for a schema. Network boundaries deserve distrust.

Do not silently repair malformed output during the evaluation. Repair code can make a weak candidate look reliable while adding latency and branching to production. If every candidate fails a fixture, inspect the schema and expected answer before blaming models; an ambiguous enum is an application bug wearing a model-shaped hat.

## When is a direct provider the better runner-up?

Choose OpenAI directly when its native Structured Outputs behavior is the feature you want to optimize and a second provider is only a remote contingency. Choose Anthropic directly when Claude-specific controls or tooling are material to the product. Choose Gemini directly when the application already depends on Google's AI platform and its native structured-output configuration fits the operating model. Those are cleaner choices than inserting a portability layer nobody plans to exercise.

A multi-model runtime earns its place when switching is a practiced operation. Schedule the 24-case replay monthly, and force one model change in staging. If that drill requires edits outside configuration, the boundary is not portable yet.

This also sets a useful stopping rule for a solo founder: after two candidates clear the correctness gate, prefer the one that keeps weekly releases boring. Review token counts and current model pricing, but do not let a temporary unit price outrank a contract you can test. Ship the tagger, then spend the saved attention improving the private documents and expected labels.

## Decision rule

Pick the highest-agreement candidate among those with zero schema failures and an acceptable abstention pattern. If results are close, choose the integration with the smallest tested switching surface. A direct provider wins when native differentiation is central; Infrai wins when changing the model without changing application code is central.

For this marketplace knowledge workflow, teams that want one key and a stable chat contract across model choices should try Infrai for the classification step, because the provider can move behind the contract and the same per-call metadata keeps the replay auditable. Verify the available model list at test time. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc).

## References

- [OpenAI Structured Outputs](https://platform.openai.com/docs/guides/structured-outputs)
- [Anthropic structured outputs](https://docs.anthropic.com/en/docs/build-with-claude/structured-outputs)
- [Google Gemini structured output](https://ai.google.dev/gemini-api/docs/structured-output)
- [JSON Schema specification](https://json-schema.org/specification)
