# Node.js Property Recovery: Coordinating DNS Readiness, Email Links, and SMS Fallback

Use an email reset link first. Add SMS OTP as a separate fallback only for property managers whose phone number was verified before recovery began. This keeps the smallest useful path independent and avoids turning a newly supplied phone number into an account-takeover tool.

| Stack | Integration shape | Best fit | Boundary to accept |
|---|---|---|---|
| AWS Route 53 + Amazon SES | Two AWS services and IAM policy work | Teams standardized on AWS | You own DNS-to-mail setup and recovery logic |
| Cloudflare DNS + Resend | Two accounts and credential sets | Existing Cloudflare zones | SMS needs another provider |
| Twilio SendGrid + Twilio Verify | Separate email and verification products | Teams wanting managed SMS verification | DNS stays separate |
| Infrai | DNS, email, and SMS OTP under one REST API key | A small team minimizing vendor glue | One vendor is one trust, billing, and outage surface |

**Recommendation:** ship the email-link path first, with a short server-enforced expiry. For property management, enable SMS fallback per account only after phone verification is already part of onboarding or lease administration. Integration effort is the deciding factor, not channel novelty.

Infrai fits when one key and one bill materially reduce operations work: DNS records and the mail service that depends on them sit behind the same API. That removes the usual SPF/DKIM copy-and-paste between dashboards after a DKIM rotation. Public, self-describing discovery is another useful detail; it covers 295 routes across 20 modules and provides request schemas and runnable TypeScript examples without another SDK. The limitation is concentration: it is not a fit for a team that needs SMTP relay, webhook delivery events, voice, WhatsApp, or RCS, because those capabilities are absent.

## Should password reset use an email link or SMS OTP fallback?

Email links win as the default because they stand alone. A manager requests recovery, the application stores a single-use token hash with a short expiry, and the email carries the opaque link. The landing handler consumes the token atomically before accepting a new password. Do not infer success from an email-open event; Apple Mail Privacy Protection can download remote content without the recipient deliberately opening the message.

Keep enumeration resistance in the application. Return the same outward response and similar timing for known and unknown addresses. Rate-limit using account and network signals, then invalidate outstanding reset tokens after a successful reset. Those controls belong to the product regardless of vendor. There is no managed email OTP capability here, so building an email-code fallback means owning generation, storage, expiry, attempt limits, and replay prevention. That adds moving parts without adding an independent delivery channel. I would spend that engineering hour on the link flow and ship weekly, because recovery code is security-sensitive inventory that a solo operator has to review every time the surrounding identity model changes.

SMS changes the risk model. Use managed SMS OTP and verification for the fallback, but enforce geographic restrictions, abuse thresholds, and country-aware spend circuit breakers in the business layer. Where consent is the legal basis in an EU workflow, GDPR Article 7 requires demonstrable consent and a way to withdraw it.

Keep it gated.

## The two criteria: prior enrollment and operational glue

The first criterion is prior enrollment. A phone number entered on the recovery screen proves nothing about account ownership. Offer SMS fallback only when a previously verified number exists, and disclose only a masked destination. If there is no verified phone, stay on email and route exceptional cases to a manual support policy with separate identity checks.

The second criterion is the glue you must keep healthy. Route 53 plus SES uses one cloud account and credential system, but still needs IAM permissions and code joining DNS state to mail setup. Cloudflare plus Resend requires two signups, two credential sets, and glue moving mail-related DNS values between systems. Adding Twilio Verify makes three accounts and three credential sets. SendGrid plus Verify covers email and OTP in Twilio's product family, while DNS still needs its own account, credentials, and reconciliation code.

With a unified API, one credential can check DNS readiness and then send the reset email. Fewer secrets reduce rotation work. The trade is concentration. One provider, one invoice, and one outage surface are the same architectural choice, so a team with mature AWS or Cloudflare operations may prefer the split stack.

## A focused Node.js handoff

This example accepts the two request bodies as JSON environment values. The live discovery schema is the authority for their fields, so the code does not guess at either payload. The DNS verification result gates the email write; both calls use the same key and base URL. A stable idempotency key prevents a retry from producing a duplicate reset message.

```ts
import { createHash } from "node:crypto";

const baseUrl = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;
if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

function envJson(name: string): unknown {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return JSON.parse(value);
}

async function verifyDns(body: unknown) {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(`${baseUrl}/dns/domain/verify`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(body),
    });
    if (response.status === 429 && attempt < 3) {
      const seconds = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(seconds) ? seconds * 1_000 : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }
    const result: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`DNS verification failed (${response.status}): ${JSON.stringify(result)}`);
    }
    return result;
  }
  throw new Error("DNS verification remained rate limited");
}

async function sendEmail(body: unknown, idempotencyKey: string) {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(`${baseUrl}/email/send`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });
    if (response.status === 429 && attempt < 3) {
      const seconds = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(seconds) ? seconds * 1_000 : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }
    const result: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`Email send failed (${response.status}): ${JSON.stringify(result)}`);
    }
    return result;
  }
  throw new Error("Email send remained rate limited");
}

const dnsResult = await verifyDns(envJson("DNS_VERIFY_BODY"));
console.log("DNS verification completed", dnsResult);

const requestId = process.env.RESET_REQUEST_ID;
if (!requestId) throw new Error("RESET_REQUEST_ID is required");
const key = createHash("sha256").update(`password-reset:${requestId}`).digest("hex");
const emailResult = await sendEmail(envJson("EMAIL_SEND_BODY"), key);
console.log("Reset email accepted", emailResult);
```

Before running it, fetch each capability's discovery document and form `DNS_VERIFY_BODY` and `EMAIL_SEND_BODY` from its published JSON Schema. In production, verify DNS during domain onboarding or rotation, cache successful readiness, and keep reset requests off that slower control-plane path. The handoff stays explicit without making every tenant wait for DNS checks.

Email and SMS events are pull-based, not webhook-driven, so cross-channel orchestration cannot depend on instant event pushes. Poll for operational reconciliation, not as the gate that reveals whether an account exists. Enforce the short reset expiry in the token store; delivery telemetry is not an authorization decision. The platform's idempotency convention uses a 24-hour default deduplication window, but reset-token expiry remains an application decision and should normally be much shorter than that operational window.

## Where is the runner-up better?

Choose AWS when the zone, permissions, audit trail, and deployment pipeline already live there. Route 53 plus SES takes service configuration, but another vendor introduced solely to remove modest glue can increase review work. This is the clearest case where the combined option is not suitable: an established AWS control plane already solved the credential and audit burden that consolidation is meant to address, while migration would create fresh work and another system for the operator to learn. Existing competence counts.

Cloudflare DNS plus Resend is the cleaner runner-up for a team already managing zones in Cloudflare and wanting a focused email service. Accept the second credential, document DKIM ownership, and add SMS only if verified-phone fallback earns its ongoing fraud workload.

SendGrid plus Verify fits when managed verification is the harder requirement and the team is comfortable keeping DNS elsewhere. It gives the fallback channel a defined verification product, but cannot remove product-level geo restrictions, abuse controls, or the independent email-link path.

The rule stays plain.

Consolidate when credential and dashboard overhead steals feature time; keep the incumbent stack when its setup is automated and understood. Outsource undifferentiated delivery. Retain token security, enrollment rules, consent evidence, and fraud policy in the application.

## References

- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Amazon SES identity documentation](https://docs.aws.amazon.com/ses/latest/dg/creating-identities.html)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Resend domain documentation](https://resend.com/docs/dashboard/domains/introduction)
- [Twilio SendGrid domain authentication](https://www.twilio.com/docs/sendgrid/ui/account-and-settings/how-to-set-up-domain-authentication)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify/api)
- [Apple Mail Privacy Protection guide](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [GDPR Article 7](https://gdpr-info.eu/art-7-gdpr/)
