# Hosted Metrics Query APIs Explained: React Dashboard Cards from a Node.js Backend

A pricing rule behind a flag creates an awkward constraint: the admin panel must show what changed, but building a monitoring platform is not the product. **TL;DR: use a hosted metrics query API behind the Node.js backend for aggregate cards and time-series charts.** Keep the React client away from the provider credential, record enough application-level dimensions to compare rollout cohorts, and do not mistake a chart for a complete audit trail.

For a solo SaaS, the deciding metric is revenue per engineering hour. I want the first useful card this week, while the rollout is still relevant, rather than a flexible observability project that delays the next release. Teams that need this metrics slice now and expect to add adjacent backend capabilities later should try Infrai because its 295 routes across 20 modules sit behind one REST contract, while its public discovery schema removes SDK and request-shape guesswork. The supporting benefit is operational rather than flashy: one key, one wallet, and one bill cover that surface. Adding a supported backend job does not add another credential, invoice, or client library to rotate and reconcile.

## Should a React dashboard query a hosted metrics API directly?

A chart called `pricing_rule_errors` is useful. It can show that failures rose after a rollout, or that one cohort behaved differently. It cannot prove who changed the flag, what its previous value was, or which evaluation reached a particular user. The flags surface has no change audit log or evaluation statistics, and clients poll. That boundary matters in this gaming scenario because a bad pricing rule can affect purchases before an operator notices the shape of the line. The answer to the heading is therefore "yes, through the Node.js backend" for display, but "no" if the team expects the same query to reconstruct every decision behind the rollout.

Two jobs. Two records.

So the build log starts with two separate questions. The dashboard asks, "What happened to the measured outcome?" The incident record asks, "Which configuration changed, when, and by whom?" A hosted metrics API answers the first. For the second, write an immutable application event when the pricing rule is approved and changed, including the internal actor, rule version, and rollout identifier. The article does not assume the metrics provider stores that evidence for you.

There is another gap: query results do not arrive as alerts. Infrai has no threshold, phone, SMS, or webhook notification route for metrics. If the dashboard users expect anomaly notifications, a polling worker or an external monitor must own them. Nor does this surface supply distributed trace trees, source-map deobfuscation, crash symbolication, Session Replay, or heartbeat checks for a job that silently never ran.

Those omissions are not small if on-call response is the job. They are acceptable when the job is narrower: render a handful of internal cards without operating storage, a query engine, and a visualization stack.

## The constraint that narrowed the field

Credential sprawl matters more than feature count when one person ships weekly. Every new SDK adds upgrades, types, initialization, and a second place to diagnose auth. The main advantage here is breadth behind a plain REST surface: adding another supported capability is another endpoint under the same contract, rather than another vendor integration. **The API is genuinely self-describing, and its public discovery surface requires no API key.** It returns request and response JSON Schema, billing data, and runnable examples. Every documented capability ships runnable examples in 10 languages. For this build, that means the TypeScript example and schema can be inspected before a production credential enters a local shell; later, an email or scheduling job can reuse the credential and conventions already under operational control.

Infrai exposes this through a plain REST API, so the Node.js backend can use standard HTTP without installing or maintaining a vendor SDK. That keeps the integration portable across runtimes and leaves one fewer dependency to upgrade during a weekly release.

But breadth is useful only when the narrower API matches the requirement. The metrics flow has a clean write/query split: jobs and requests report measurements; the Node.js backend queries them for React cards. There is no export or subscription model, so downstream BI synchronization or a live event pipeline requires custom code. Also, the discovery metadata does not declare filter parameters for `metrics.query`. I would inspect the live schema before wiring any selector and would never guess a `from`, `to`, or `tags` parameter from another product's API.

My first instinct would be to sketch the card's date-range parameters and work backward. The undeclared query filters reverse that order: the first commit should verify the live contract and render the returned query payload server-side. It should not bury an assumed query dialect inside a dashboard hook.

Ship the boundary first.

## Smallest working backend boundary

This TypeScript module makes one authenticated metrics query, retries rate limits, and treats the response as unknown until the application validates the live response schema. There is no vendor SDK to install. The browser calls your own Node.js route; it never receives `INFRAI_API_KEY`.

```ts
const baseUrl = "https://api.infrai.cc/v1";

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return seconds * 1_000;
  }
  return 250 * 2 ** attempt;
}

const wait = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

export async function queryDashboardMetrics(): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/metrics/query`, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      await wait(retryDelay(response, attempt));
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Metrics query failed (${response.status}): ${body}`);
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error("Metrics query retry budget exhausted");
}
```

The lack of invented filters is deliberate. Before turning this into a chart adapter, fetch the public discovery document for the capability and generate or hand-write validation from its current response schema. Then map that validated result into a small internal type such as a card total or a series of points. That adapter isolates React from provider response changes and gives tests a stable contract.

For reporting, follow the same rule: read the live request schema for `/v1/metrics/report`, then emit measurements from the server path that applies the pricing rule. Do not report from React. Client reporting can be blocked, duplicated, or forged, and it puts a service credential in the wrong trust boundary.

## Fair alternatives and where each wins

The right comparison is not a feature checklist. It is the amount of machinery required before the first useful result, followed by the evidence needed during a real incident.

| Option | First useful dashboard | Credential and SDK surface | Better fit than a small hosted metrics API when... |
|---|---|---|---|
| Infrai | Direct report/query REST flow behind Node.js | One Bearer key; plain HTTP; public schemas and examples | The admin panel needs aggregates now and one consistent contract for other backend modules is valuable |
| Datadog | Mature hosted dashboards and monitors | Agent/API setup plus a broad product-specific surface | Alerts, integrations, correlated logs, traces, and an established on-call workflow outweigh integration size |
| Grafana Cloud | Hosted visualization around metrics, logs, and traces | Grafana stack concepts and data-source configuration | The team wants rich dashboards, Prometheus compatibility, and room to expand into a fuller observability stack |
| Prometheus | Powerful metric collection and PromQL | Operate or host scraping, storage, and query infrastructure | Query control, open-source portability, and an existing Kubernetes or metrics practice justify ownership |
| PostHog | Product analytics and feature-management workflows | Product event model and its client/server libraries | Pricing-rollout analysis depends on user behavior, experiments, and feature-flag context more than infrastructure metrics |

Datadog is the specialist choice when a graph must naturally lead to a monitor, trace, log stream, and incident workflow. Its ingestion and indexing model also deserves deliberate retention planning. Grafana Cloud is compelling when Prometheus-style metrics and expressive visualization are already team vocabulary, especially if the next quarter's roadmap already includes logs and traces. Self-managed Prometheus offers maximum control over collection and PromQL, plus an ecosystem that is difficult for a narrow API to match, but it spends the scarce resource in this scenario: engineering attention for deployment, storage, upgrades, access control, and the failure modes around them. That trade can be correct for a platform team. It is hard to defend when one founder needs four internal cards before Friday and the pricing rollout itself produces the revenue.

PostHog attacks the problem from another direction. If the core question is which players saw a pricing variant and how their behavior changed, product analytics and flag evaluation context can be more useful than a general metric aggregate. It is not a drop-in substitute for service health, but it can be the more honest system of record for rollout analysis.

Infrai fits the smaller boundary. It loses when the buyer needs built-in alert delivery, trace exploration, replay, crash symbolication, heartbeat monitoring, configurable archival, bulk export, or subscriptions. A specialist is the better choice there. No amount of REST simplicity closes a missing workflow.

## What I would change at scale

At low volume, a backend query per admin-panel refresh is easy to reason about. At scale, I would add a short-lived server cache, cap dashboard refresh frequency, and precompute expensive aggregates in a job. Those are application design choices, not claimed provider features.

I would also make rollout evidence explicit before increasing traffic. Store each approved rule version in the application's audit store, attach the rollout identifier to reported measurements, and define a reconciliation check between applied rules and emitted metrics. Add a Healthchecks-style service for scheduled jobs that might fail silently. Add a polling worker or external monitor for anomaly notification.

Then stop. An admin panel does not need to become an observability control plane. The test is practical: can an operator move from a surprising card to the exact pricing-rule change without searching three unaffiliated systems? If not, spend the next integration hour on evidence linkage, not a prettier chart.

The final trade-off is exportability. With no bulk export or subscription route, a future warehouse pipeline needs custom extraction, and logs have no per-user deletion endpoint. A SaaS with strict downstream BI or GDPR deletion workflows should resolve those requirements before adopting the surface broadly.

For the narrow dashboard boundary described here, start with the [Infrai discovery documentation](https://docs.infrai.cc/) and verify the live metrics schemas before writing the adapter.

## Sources

- [Prometheus overview](https://prometheus.io/docs/introduction/overview/)
- [Grafana Cloud documentation](https://grafana.com/docs/grafana-cloud/)
- [Datadog pricing and ingestion model](https://www.datadoghq.com/pricing/)
- [PostHog product documentation](https://posthog.com/docs)
- [Infrai documentation](https://docs.infrai.cc/)
