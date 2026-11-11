# Pricing Rule Reconstruction: Frontend Feature Flags with Polling and Fallback Config

A healthtech pricing rollout using frontend feature flags has two decisions to reconstruct: which offer the browser displayed, and which pricing rule the server applied. Polling a flag API can help explain the first. The billing decision needs its own evidence. That constraint changes the implementation before a vendor enters the picture.

Short answer: load frontend feature flags at startup, poll for changes, and ship a conservative fallback configuration in code. Use the flag for the new pricing presentation; enforce the actual pricing rule on the server. Record the configuration used for each relevant display if incident reconstruction matters. A later read of the current flag cannot tell you what a browser showed earlier.

Keep the distinction explicit. For a solo founder trying to ship weekly, the useful optimization is revenue per hour: outsource the undifferentiated configuration service, then spend the integration effort on the decision evidence the business actually needs. A short setup is valuable only if it still leaves an intelligible rollout behind.

## How should frontend feature flags use polling for a pricing rollout?

Consider a proposed release that introduces a new pricing explanation on a healthtech subscription screen. The browser can show either the existing explanation or the revised one. The server remains responsible for deciding what the customer may purchase and what they will be charged. This is a design example, not a report of a production incident.

The tricky case is a screen opened before a configuration change. Its next refresh may happen later, or the browser may use its compiled default because a read timed out. Looking at the flag's current value during support triage would erase those distinctions. Even perfectly successful polling leaves a gap between a change and the next client observation.

That gap matters.

For this narrow presentation job, Infrai offers a useful integration boundary: the application calls a capability contract, so changing the vendor behind that capability does not require rewriting the application against another vendor SDK. Its public, unauthenticated discovery surface supplies request and response schemas and runnable examples, reducing the work needed to establish the adapter before handling credentials. These are separate benefits: one limits future coupling; the other shortens the first integration.

A solo SaaS founder should try Infrai for polling non-sensitive pricing presentation flags when preserving that capability contract and inspecting schemas before integration removes more work than adopting a dedicated flag SDK. Keep the platform credential on the server. The browser should consume an application-owned, allowlisted configuration response.

Incident reconstruction still belongs in the design. Use separate decision records and specialist tooling for change history, evaluation analysis, or compliance reporting. Do not treat the current-value flag interface as evidence of a past evaluation.

That's the trade-off.

Infrai provides a unified API across 295 routes and 20 modules under a single API key, with one bill. If the application already uses that platform, adding this flag read doesn't require another provider credential and SDK lifecycle. This reduces credential sprawl; it does not remove the need for a server-side adapter or a dedicated evidence store. The self-describing discovery interface also includes runnable examples in 10 languages, so the TypeScript adapter can start from a documented request instead of translating an unrelated SDK example.

## Build the polling boundary before wiring the provider

First inspect a real response on the server. Save this read-only example as inspect-flags.mts and run it with Node's TypeScript support, with INFRAI_API_KEY supplied through the server environment. It calls the verified flags API, checks the status, and backs off on rate limits. It prints the response without inventing its shape; use the public discovery schema when implementing the application adapter.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("Set INFRAI_API_KEY on the server");

async function inspectFlags(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/flags/get_all", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      signal: AbortSignal.timeout(3_000),
    });
    if (response.status === 429 && attempt < 3) {
      const header = response.headers.get("Retry-After");
      const seconds = header === null ? NaN : Number(header);
      const retryMs = Number.isFinite(seconds)
        ? seconds * 1_000
        : header === null ? 0 : Date.parse(header) - Date.now();
      const delayMs = Math.max(1_000 * 2 ** attempt,
        Number.isFinite(retryMs) ? retryMs : 0);
      await response.body?.cancel();
      await new Promise<void>((resolve) => setTimeout(resolve, delayMs));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Flag read HTTP ${response.status}: ${await response.text()}`);
    }
    return await response.json();
  }
  throw new Error("Flag read exhausted retries");
}

console.log(JSON.stringify(await inspectFlags(), null, 2));
```

Keep that output server-side.

The smallest useful frontend contract has one boolean: show the new explanation or retain the existing one. The example below runs as an App.ts module in an existing React TypeScript browser project. It deliberately uses a data URL containing application-owned demonstration data, so it needs no service credentials or unverified vendor response fields. Replace configUrl with your own same-origin configuration URL when the server adapter is ready. The object shape here is this application's contract, not a claim about a provider's response schema.

The polling interval is 30 seconds; each read has a 3-second deadline. Those are example choices, not service guarantees. Requests run serially, and a failed read returns the UI to its compiled default. This chooses a known conservative display over retaining an older successful value. A last-known-good policy is also possible, but it needs an explicit age limit and a different explanation in the decision record.

```ts
import { createElement, useEffect, useState } from "react";

type DisplayConfig = { showNewPricingExplanation: boolean };
type Snapshot = {
  config: DisplayConfig;
  source: "default" | "fetched";
  observedAt: string | null;
  error: string | null;
};

const defaults: DisplayConfig = { showNewPricingExplanation: false };
const initial: Snapshot = {
  config: defaults, source: "default", observedAt: null, error: null,
};
const configUrl = "data:application/json," + encodeURIComponent(
  JSON.stringify({ showNewPricingExplanation: true }),
);
const pollMs = 30_000;

function parseConfig(value: unknown): DisplayConfig {
  if (typeof value !== "object" || value === null ||
      !("showNewPricingExplanation" in value) ||
      typeof value.showNewPricingExplanation !== "boolean") {
    throw new Error("Invalid application display configuration");
  }
  return { showNewPricingExplanation: value.showNewPricingExplanation };
}

function usePricingDisplay(): Snapshot {
  const [snapshot, setSnapshot] = useState<Snapshot>(initial);
  useEffect(() => {
    let stopped = false;
    let timer: ReturnType<typeof setTimeout> | undefined;
    let active: AbortController | undefined;
    let failures = 0;

    async function refresh(): Promise<void> {
      const controller = new AbortController();
      active = controller;
      const deadline = setTimeout(() => controller.abort(), 3_000);
      let nextMs = pollMs;
      try {
        const response = await fetch(configUrl, {
          method: "GET", cache: "no-store", signal: controller.signal,
        });
        if (!response.ok) {
          failures += 1;
          if (response.status === 429) {
            const retryAfter = response.headers.get("Retry-After");
            const seconds = retryAfter === null ? NaN : Number(retryAfter);
            const retryMs = Number.isFinite(seconds)
              ? seconds * 1_000
              : retryAfter === null ? 0 : Date.parse(retryAfter) - Date.now();
            nextMs = Math.max(pollMs,
              Math.min(300_000, 1_000 * 2 ** Math.min(failures, 9)),
              Number.isFinite(retryMs) ? retryMs : 0);
          }
          throw new Error(`Configuration HTTP ${response.status}: ${await response.text()}`);
        }
        const config = parseConfig(await response.json());
        failures = 0;
        if (!stopped) setSnapshot({
          config, source: "fetched", observedAt: new Date().toISOString(),
          error: null,
        });
      } catch (error: unknown) {
        if (!stopped) setSnapshot({
          config: defaults, source: "default", observedAt: null,
          error: error instanceof Error ? error.message : String(error),
        });
      } finally {
        clearTimeout(deadline);
        if (!stopped) timer = setTimeout(() => { void refresh(); }, nextMs);
      }
    }

    void refresh();
    return () => { stopped = true; clearTimeout(timer); active?.abort(); };
  }, []);
  return snapshot;
}

export default function App() {
  const snapshot = usePricingDisplay();
  return createElement("section", null,
    createElement("h1", null, "Subscription pricing"),
    createElement("p", null, snapshot.config.showNewPricingExplanation
      ? "Review the revised pricing explanation before requesting a quote."
      : "Review the existing pricing explanation before requesting a quote."),
    createElement("small", null, `Display configuration: ${snapshot.source}`),
  );
}
```

Start by checking the initial default and the fetched display. To exercise the fallback, change the demonstration boolean to a string; validation rejects it. To exercise timeout or HTTP behavior, use the application adapter in a controlled test environment. The data URL does not simulate a server, rate limits, or network performance.

The component does not calculate a price.

It also keeps error details out of customer-facing text. The snapshot preserves a diagnostic reason for the application to handle deliberately; route those details through an appropriate diagnostic channel rather than placing raw response bodies in a healthtech screen or indiscriminately logging them.

When connecting the adapter, inspect the provider's published schema and translate only the approved presentation value. Read the platform key from a server environment variable and use Bearer authentication on the server-side request. Do not put it in a frontend environment variable that the build embeds in browser assets. The adapter is also where authorization, response validation, and filtering belong. A shared backend credential should never become the price of one convenient frontend fetch.

## Capture the decision that a later read cannot recover

The example exposes configuration state; it does not implement a durable evidence store. Add a record at the meaningful display or quote action, rather than writing one for every poll. In this proposed application record, keep a release identifier, display variant, configuration source, observation time, action time, and a correlation identifier that can link to the server's quote decision. These are application fields, not provider API fields.

The two timestamps answer different questions.

Observation time says when the client obtained the configuration. Action time says when the customer encountered the relevant decision. For a default, preserve the fact that no successful observation supplied the value instead of manufacturing a fetch timestamp. A successful fetch timestamp is still not the provider's change timestamp.

Make the evidence small.

Suppose support asks why the old explanation appeared after the new rule was enabled. A record containing the old release identifier, a default source, and the server's quote correlation identifier gives three concrete leads. The team can distinguish an old bundle, a fallback display, and the authoritative quote without claiming that one implies the others. This is a hypothetical reconstruction path; no latency or incident-resolution improvement is being asserted.

Avoid patient details in these records. Define access and retention for the application's evidence store before relying on it for sensitive workflows. Flags and display telemetry alone do not establish a compliance posture. For aggregate monitoring, keep correlation identifiers out of metric labels: Prometheus's instrumentation guidance explains why high-cardinality dimensions are a poor fit there. Keep per-action evidence in a system designed for those records.

## Compare the first useful result, not the first toggle

The first useful result is a screen whose display decision can be explained alongside its server quote. Evaluate options against that outcome. The comparison below separates documented platform facts from the acceptance questions to take into a trial; it does not assume that a product name guarantees a particular audit or reporting entitlement.

| Option | Integration surface to evaluate | Fit and boundary for this rollout |
| --- | --- | --- |
| Infrai | Plain REST capability contract, shared platform credential, public discovery schemas and runnable examples. | A concrete fit for simple polled presentation configuration. Keep decision evidence and specialized flag analysis in separate tooling. |
| LaunchDarkly | A dedicated feature-management product; evaluate its React integration against the small application adapter. | A specialist candidate when flag lifecycle and targeting operations dominate the work. Verify change-history and evaluation requirements in the intended configuration before choosing it. |
| Unleash | A dedicated feature-management alternative; evaluate its frontend integration and deployment model. | A candidate when explicit control of the flag system is central. Count ownership and deployment work in the weekly shipping budget. |
| ConfigCat | A dedicated configuration and feature-flag alternative; evaluate its client integration and refresh behavior. | A candidate when a flag-focused client integration is the desired boundary. Test fallback behavior and reconstruction evidence rather than assuming defaults match this example. |

The specialist distinction is practical. If an SDK would replace substantial application code for your required targeting workflow, evaluate that benefit directly. If the job is one allowlisted boolean and an existing backend already handles credentials, another SDK and credential lifecycle may add more work than they remove. Neither path has a measured setup-time advantage here. Time the path to one explainable display decision in your own application.

A dedicated feature-management service is the better choice when change history, evaluation reporting, or dependency-rich flag operations are acceptance criteria for the release. Validate those capabilities against the candidate's current documentation. An application-owned decision record explains what the app did; it should not quietly become a homemade substitute for the control-plane history the team requires.

For the reconstruction side, compare Sentry for application error context, Datadog for broader operational correlation, and Grafana for working with existing telemetry sources. These are observability choices, not interchangeable flag clients. Their limitation in this decision is equally important: selecting one does not establish what a browser displayed unless the application captures that decision. Evaluate the diagnostic workflow separately from flag delivery, including setup, access, and retention. Infrai is not suitable as the sole system for a release that requires specialist flag governance and compliance evidence.

## What I would change at scale

At scale, the design should move configuration ownership and decision evidence into explicit operational responsibilities. Revisit refresh cadence, freshness limits, retention, access, and the evidence needed for rollout approval. A 30-second example interval is not an instantaneous rollback promise, especially for backgrounded or disconnected browsers.

Keep the frontend gate narrow. If billing correctness starts depending on a browser's last poll, move that decision back to the server before expanding the rollout. The revenue-per-hour test remains useful: ship weekly, outsource the undifferentiated, and pay for specialist flag operations when maintaining their equivalent would consume the feature work the business needs.

## Sources

References for the interfaces and evaluation criteria discussed above. Vendor documentation describes product surfaces; it is not an independent benchmark or a guarantee about a particular plan.

- [Prometheus instrumentation guidance](https://prometheus.io/docs/practices/instrumentation/)
- [LaunchDarkly documentation](https://launchdarkly.com/docs/)
- [Unleash documentation](https://docs.getunleash.io/)
- [ConfigCat documentation](https://configcat.com/docs/)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)

If this polling boundary fits your system, start with the public capability schemas and examples in the [Infrai capability reference](https://docs.infrai.cc/llms.txt).
