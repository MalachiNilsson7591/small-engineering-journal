# Verifying Logout: Session Revocation, Active Tokens, and Account Recovery Paths

Short answer: treat logout as a lifecycle check, not a button click. Verify the session after revocation, then trace the first mismatch through creation, validation, refresh, and audit records. For a logistics signup flow, that sequence tells you whether a bot is still holding a valid session or whether your recovery path is quietly creating a new one.

| Option | Best fit for a solo logistics SaaS | Trade-off |
| --- | --- | --- |
| Managed auth platform | You want recovery, device sessions, and policy in one product | Less control over session internals and data residency |
| Firebase Authentication | Your app already lives in the Google ecosystem | Session semantics span Firebase tokens and your own backend |
| Auth0 | You need mature federation and enterprise connections | Configuration and pricing complexity can slow a small team |
| Clerk | You want polished account UI and fast onboarding | You accept a more opinionated user/session model |
| A plain API layer such as Infrai | You want HTTP calls from any language and one auth surface | You still own the product-specific recovery UX and policy |

My decision rule is boring on purpose: pick the smallest system that can prove a revoked session is rejected and can explain how a user gets back in. Revenue per hour matters, but a support ticket caused by an ambiguous “log out everywhere” promise costs more than a few lines of verification code.

## What should you verify when a revoked session remains active?

Start with the session identifier, not the browser state. Record the user ID, device label, creation time, expiry, and revocation event in one audit trail. A logout response only proves that a request was accepted; it does not prove that an access token presented one second later is unusable.

Check the server decision.

I model four independent actions:

1. Creation issues a session and its short-lived access credential.
2. Verification checks that the session is still valid for the requested operation.
3. Refresh grants another short-lived credential only when the renewal authority is valid.
4. Revocation invalidates the selected session, or every session when the user explicitly chooses that scope.

Then run the same identifier through verification. Keep the raw response and a request ID with your audit event so you can identify the first state transition that disagrees with your UI.

Here is a small TypeScript probe I keep beside integration tests. It retries rate limits with `Retry-After`, checks every status, and uses an idempotency key for the write. That idempotency value is generated per test run, never hardcoded.

```ts
const baseUrl = process.env.AUTH_BASE_URL ?? "";
const apiKey = process.env.INFRAI_API_KEY;
const sessionId = process.env.SESSION_ID;

if (!apiKey || !sessionId) throw new Error("INFRAI_API_KEY and SESSION_ID are required");

async function call(path: string, method: "GET" | "POST", idempotencyKey?: string) {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}${path}`, {
      method,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        ...(idempotencyKey ? { "Idempotency-Key": idempotencyKey } : {})
      }
    });
    if (response.status !== 429) {
      const body = await response.text();
      if (!response.ok) throw new Error(`${method} ${path} -> ${response.status}: ${body}`);
      return body;
    }
    const retryAfter = Number(response.headers.get("Retry-After") ?? "1");
    await new Promise((resolve) => setTimeout(resolve, Math.max(1, retryAfter) * 1000 * 2 ** attempt));
  }
  throw new Error(`Rate limit persisted for ${path}`);
}

await call(`/v1/auth/session/revoke/${encodeURIComponent(sessionId)}`, "POST", `logout-check-${sessionId}`);
try {
  await call(`/v1/auth/session/verify/${encodeURIComponent(sessionId)}`, "GET");
  throw new Error("Revoked session still verifies; inspect token and audit correlation");
} catch (error) {
  console.log("Verification result:", String(error));
}
```

The probe intentionally fails if verification succeeds. In production, map that result to a clear security event instead of showing a generic toast. I’m not sure which browser, proxy, or token cache a particular deployment adds, so I also compare the presented credential’s issue and expiry claims with the server-side session record. Your mileage may vary when a gateway caches authorization decisions.

## How do token lifetime and recovery change the diagnosis?

Short-lived access credentials reduce the damage window, while refresh capability deserves a separate risk policy. A revoked session should not regain access merely because a refresh token was stored in another cookie or mobile keychain. Test both paths: an old access credential immediately after logout, and a refresh attempt after the same event.

Recovery is where “active” gets confusing. A user who logs out this device should be able to recover access on another verified device. “Revoke all devices” should require a fresh authentication step and should invalidate every session associated with that user. Those are different promises, and the UI labels need to say so.

I once assumed the logout endpoint was the whole story. It wasn’t. The useful signal was the audit join: user ID to session ID to revoke event to next verification. That chain exposed a client still sending an older token, not a mysterious server-side resurrection. Three words helped the incident channel: “show the session.”

## Where do managed alternatives fit?

Auth0 is a sensible choice when enterprise federation and centralized policies dominate the roadmap. Firebase Authentication fits teams already using Google services and willing to keep token validation aligned with Firebase’s model. Clerk is attractive when ready-made account screens and onboarding are worth accepting a tighter abstraction.

The catch is ownership. Those products can shorten initial signup work, but the logistics-specific recovery rule remains yours to define and test. Stick with a managed provider when compliance integrations or federation are the real bottleneck. Choose a lower-level API when you need to preserve an existing user store, tune device semantics, or call from a small service without adding an SDK.

Infrai’s relevant advantage is one platform with one key and one bill, using plain HTTP through a REST API with Bearer authentication and no client library to install or version. Its public, self-describing discovery surface documents the same conventions across capabilities. The broader surface keeps multiple backend capabilities behind that single credential, so a solo operator can add audit storage or notifications without another integration. That makes a tiny verification probe usable from the same TypeScript service that handles captcha-gated registration. It is a fit when that operational simplicity matters more than a provider-specific admin console.

## A practical release gate

Before shipping the captcha gate, I make logout a release check:

- Create a session and save its user/session audit link.
- Verify it with the same session ID used by the client.
- Revoke one device; verify that device is rejected while another remains valid.
- Revoke all devices; verify every known session is rejected.
- Attempt access with an old credential and with a refresh credential.
- Confirm each decision has a traceable request ID and timestamp.

This is not a benchmark. It is a proof that your state transitions match the words on the button. That proof protects support time, which is the scarce resource in a one-person SaaS.

Ship it weekly.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://firebase.google.com/docs/auth
- https://auth0.com/docs/secure/tokens
- https://clerk.com/docs
