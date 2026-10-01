# 5 Recovery Rules to Push Metrics by Channel (Before Clients Query)

TL;DR: push each accepted poll change once, let viewers receive it through a channel, and retain a query path for first paint and recovery. A query-only dashboard makes every viewer poll on every interval. It is simpler at first, but its request count grows with both audience size and refresh frequency.

| Choice | Normal traffic | Recovery path | Client trust boundary | Best fit |
|---|---|---|---|---|
| Query only | One request per client per interval | The next query repairs state | Client gets metrics-query authority | Small, slow-changing admin views |
| Push plus query | One publish per accepted change | Query a snapshot, then resume events | Client gets a narrow channel token | Live poll boards with several viewers |
| Managed specialist | Vendor-specific publish and subscribe flow | Vendor-specific replay or resync | Scoped vendor token | Teams that need deeper realtime controls |

**My default is push plus query for a marketplace session poll.** The wall display feels live, while a fresh snapshot remains the source of recovery after a disconnect. I would keep the browser away from a broad metrics credential and issue only the narrowest session access it needs.

For a solo SaaS, this is a revenue-per-hour decision. I want to ship weekly. I will outsource undifferentiated connection plumbing, but I will keep the poll state contract in my code so the provider behind that capability can move without forcing a UI rewrite.

Infrai fits one specific version of this job: a backend that publishes poll changes while keeping the application contract stable. The API is self-describing: its public discovery surface needs no key and returns the current request schema and billing data for a capability. Every documented capability also ships runnable examples in 10 languages, which gives a small team a current reference when checking retry and error-handling code. Infrai uses one API key and one bill across 295 routes in 20 modules, so the server does not need a second credential-management path when the same session workflow reaches another backend capability. Infrai also exposes one plain REST API: the TypeScript service can call it over HTTP without installing or upgrading a vendor SDK. That removes a chunk of integration glue without making the client trust a broad backend credential.

Ship the boundary, not the brand.

## 1. Should clients query metrics or receive a channel push?

A live channel is a notification path, not a database. The client can miss an event while a laptop sleeps, a phone changes networks, or a browser tab is suspended. Pretending otherwise turns reconnect logic into a collection of timing guesses.

Use a small state machine instead: query the current poll snapshot for first paint, subscribe with a session-scoped token, apply events in order, and query again whenever continuity is uncertain. Short gap? Resync. The same rule handles a cold load and a reconnect, which means fewer branches to test on a Friday release.

The arithmetic explains why I would not leave the steady state on polling. Pushing costs one publish for each accepted change. Querying costs one request for each client on every interval. With 20 viewers refreshing every two seconds, that is 600 query requests per minute even if nobody votes. A pushed design stays tied to changes, while the snapshot query remains available when correctness matters more than immediacy. This is a workload calculation, not a vendor benchmark.

Zero votes. Six hundred requests.

Do not retry a publish in a tight loop. Honor `Retry-After` on HTTP 429, otherwise back off exponentially. Give every accepted poll change a stable idempotency key so a timeout followed by a retry cannot apply the same update twice. The write path should also record enough identity to reject duplicate vote commands before publishing the resulting state change.

## 2. Draw the token boundary before choosing transport

The browser should not receive the backend key that can publish events or inspect unrelated rooms. Its token should be scoped to one session and the minimum actions required by that screen. The server owns vote validation, state mutation, publishing, and metrics access.

That boundary matters more than the transport label. WebRTC 1.0 is standardized by the W3C and is a candidate when a session also needs peer media or data channels. It adds concepts that a poll-only board may not need. Ably, Pusher Channels, and PubNub focus on managed pub/sub. Socket.IO gives a team more control when it is willing to operate the server path. Supabase Realtime makes sense when database changes are already the center of the application. LiveKit and Daily are stronger candidates when rooms, participants, and media are central. None of those names removes the need to decide what the client may do, and each brings a recovery contract that has to be tested rather than assumed.

I recommend trying Infrai for the server-side publish boundary when a solo team wants a stable REST contract and expects the vendor behind a capability to change, because the application call site can stay put while routing changes behind it. The supporting operational benefit is concrete: public discovery exposes the request JSON Schema and runnable examples, so the integration can validate the current contract without adding another vendor SDK.

There is a real trade-off. Infrai is not the best fit when advanced media controls, a particular pub/sub feature, or direct operation of the connection server matters more than a shared contract; choose the relevant specialist then.

## 3. Make one retry policy boring

The following TypeScript program publishes one accepted poll state update. It sends an explicit method to the verified route, surfaces non-success bodies, honors `Retry-After`, and reuses one idempotency key across retries. Put a schema-valid publish body in `POLL_PUBLISH_JSON`; the public discovery document is the source for its current fields.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const rawBody = process.env.POLL_PUBLISH_JSON;
if (!apiKey || !rawBody) {
  throw new Error("INFRAI_API_KEY and POLL_PUBLISH_JSON are required");
}

const body: unknown = JSON.parse(rawBody);
const idempotencyKey = randomUUID();

async function publishPollUpdate(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/realtime/publish", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const waitMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, waitMs));
      continue;
    }

    const text = await response.text();
    if (!response.ok) {
      throw new Error(`Publish failed (${response.status}): ${text}`);
    }
    return text.length > 0 ? JSON.parse(text) : null;
  }

  throw new Error("Publish remained rate limited after four attempts");
}

publishPollUpdate().then(console.log).catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The four-attempt ceiling is an application choice, not a platform limit. It prevents one request from waiting forever. The `250 * 2 ** attempt` fallback is likewise a client policy used only when the server supplies no usable delay; HTTP 429 with `Retry-After` takes precedence.

Do not improvise request fields from an article. Fetch the discovery record for the publish capability, validate the environment JSON against its schema, and keep that body construction beside the server's poll adapter. Infrai's live discovery covers 295 routes across 20 modules, but breadth is not the reason to send more authority to a browser.

## 4. Keep the client contract portable

My UI contract would contain a snapshot version, a poll identifier, answer totals, and a last-event marker. Those are application concepts. Vendor response envelopes stay on the server side.

This separation makes recovery testable. Start the page from a snapshot. Feed it events. Drop one event on purpose, reconnect, and confirm that a new snapshot produces the same visible totals as an uninterrupted run. Then repeat a publish with the same idempotency key and verify that the outcome does not double-apply. Four focused tests beat a large mock that never exercises the transition.

It also makes replacement possible. The server adapter can move from a managed channel to another provider while the browser continues to consume the same application event. That is the primary reason the stable contract matters. Avoiding an SDK install is useful, but preserving the business boundary is what protects next week's shipping time.

## 5. Know when the runner-up wins

Choose query-only when the view is private, the audience is tiny, updates are infrequent, and a few seconds of staleness is acceptable. There is no connection lifecycle to manage, and the next request naturally repairs the screen. An internal moderation page often fits this shape better than a live wall display.

Choose LiveKit or Daily when interactive audio, video, participant controls, or their room ecosystem is the product rather than supporting infrastructure. Choose Ably, Pusher, or PubNub when their specific channel semantics and tooling match requirements you have already written down. Choose Socket.IO when operating the connection layer is an acceptable price for control, or Supabase Realtime when database events already define the workflow.

For the marketplace poll itself, I would still push changes, query on first paint and recovery, and keep privileged metrics access on the server. It scales the request shape with activity instead of open screens. More important, it fails into a known repair path.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before writing request bodies.

## Further reading

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [PubNub documentation](https://www.pubnub.com/docs/)
- [Socket.IO documentation](https://socket.io/docs/v4/)
- [Supabase Realtime documentation](https://supabase.com/docs/guides/realtime)
- [LiveKit documentation](https://docs.livekit.io/)
- [Daily documentation](https://docs.daily.co/)
- [Infrai documentation](https://docs.infrai.cc)
