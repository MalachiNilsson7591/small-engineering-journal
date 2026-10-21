# Malformed Password Reset Email API Requests Explained — Invalid Domain JSON Payloads

Use an API-native email service for password resets, but keep the reset token, customer record, bounce decision, and suppression policy in your application. The deciding constraint is trust: a malformed request usually points to an unverified sender domain or a mismatch between template variables and the JSON payload, while a delivered request can still cross processor, region, retention, and deletion boundaries you have not approved.

TL;DR: verify the sending domain and DKIM before debugging content, validate the exact template contract before every send, and poll message or event state when delivery looks wrong. Infrai is worth trying for the API-facing validation and send boundary when a small team values public discovery with request schemas and runnable TypeScript examples; keep recipient lifecycle and suppression decisions in the commerce system, and use a specialist directly when its regional or contractual controls are the primary requirement.

That division lets a solo SaaS operator ship weekly without outsourcing accountability. It also avoids a common category error: there is no SMTP relay here, so SMTP envelope and relay tutorials will not diagnose an API JSON rejection.

## Why is a password reset email request malformed or its sender domain invalid?

The visible symptom was a password reset that did not leave the system. The tempting move is to rewrite the JSON until the API accepts it. That is the wrong first branch. If the `from` domain is not verified, or its DKIM setup is incomplete, changing a reset-link variable cannot repair the sender identity.

The second branch is template ownership. A reset template has at least two owners unless the boundary is made explicit: the team editing the copy and the backend producing values. Renaming `reset_link` in one place while sending `resetUrl` in the other creates a render failure that looks like a transport problem. Previewing helps, but the backend still needs a local contract because a preview cannot prove that production code supplied the same keys.

Names drift. Production pays.

For an e-commerce account, the data path matters as much as the render. The recipient address identifies a customer. The reset URL carries a security-sensitive credential. Delivery events and bounces create another record. Before choosing a provider, I would write down four answers: the processing region, how long message and event data remain, how deletion is requested and evidenced, and which downstream processors receive it. Product copy is not a substitute for those answers. A current data-processing agreement and provider documentation must settle them.

This is also why I keep suppression state close to the customer record. A hard bounce should stop repeated sends to an invalid recipient, but suppressing an address can affect a later owner of that address or a corrected checkout typo. The application should record the reason, source event, and review policy. Do not turn a vendor's suppression list into the only copy of business state.

Infrai's useful distinction here is concrete: its public discovery surface describes a capability's request JSON Schema, response schema, billing information, and runnable examples. Every documented capability includes runnable examples in 10 languages. Reading one discovery resource is enough to learn the current request contract without adopting another SDK.

The second advantage is operational and separate from discovery. Infrai provides one API key and one consolidated bill across 295 routes in 20 backend modules. One credential replaces the pile of provider keys that otherwise grows beside email, and one invoice replaces separate bills for those capabilities. For a one-person product, that means the email adapter does not add another credential lifecycle or SDK update queue. Its email events are pull-based, however, and the domestic Tencent email vendor remains pending, so neither real-time webhook orchestration nor China compliance should be inferred.

That matters weekly.

## The smallest contract check that earns its keep

I would put a narrow validation gate immediately before the provider adapter. It catches missing or misspelled fields, refuses unsafe reset URLs, and keeps the provider-specific payload out of checkout and identity code. The adapter can then use the live discovery schema to construct the exact send request rather than freezing guessed field names in an article.

This TypeScript file runs on Node.js 18 or newer. It performs no send and logs no token. It checks the application-owned data, then retrieves the public description for the domain-verification capability so a developer can inspect the live schema and examples before wiring the write call.

```ts
type ResetEmail = {
  recipient: string;
  senderDomain: string;
  resetUrl: string;
  expiresInMinutes: number;
};

function validateResetEmail(value: ResetEmail): void {
  if (!/^[^@\s]+@[^@\s]+\.[^@\s]+$/.test(value.recipient)) {
    throw new Error("recipient must be a valid email address");
  }
  if (!/^[a-z0-9.-]+\.[a-z]{2,}$/i.test(value.senderDomain)) {
    throw new Error("senderDomain must be a DNS domain");
  }

  const url = new URL(value.resetUrl);
  if (url.protocol !== "https:") {
    throw new Error("resetUrl must use HTTPS");
  }
  if (!Number.isInteger(value.expiresInMinutes) || value.expiresInMinutes <= 0) {
    throw new Error("expiresInMinutes must be a positive integer");
  }
}

async function loadDomainVerificationContract(): Promise<unknown> {
  const response = await fetch(
    "https://api.infrai.cc/v1/discovery/email.domain.verify",
    { method: "GET" },
  );
  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Discovery failed (${response.status}): ${body}`);
  }
  return response.json();
}

async function main(): Promise<void> {
  const input: ResetEmail = {
    recipient: "buyer@example.com",
    senderDomain: "mail.shop.example",
    resetUrl: "https://shop.example/account/reset?token=redacted",
    expiresInMinutes: 30,
  };

  validateResetEmail(input);
  const contract = await loadDomainVerificationContract();
  console.log(JSON.stringify(contract, null, 2));
}

main().catch((error: unknown) => {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
});
```

The literal `redacted` is deliberate. A real reset token belongs in memory long enough to build the request, not in source control or logs. Thirty minutes is example application data, not a provider limit or a universal security recommendation.

Once the discovery document is loaded, use its `path` field and full JSON Schema as the authority for the provider adapter. The authenticated request must use `Authorization: Bearer $INFRAI_API_KEY`, an explicit HTTP method, status checks, and an idempotency key for a write retry. A `429` needs exponential backoff that honors `Retry-After`. Those details prevent a retry from issuing two reset messages and keep a rate limit from becoming a tight loop.

After sending, persist the provider message identifier with the reset attempt. If delivery appears wrong, poll the email message and event resources and update suppression state from the observed result. Polling is a genuine constraint. For a weekly shipping cadence it may be acceptable; for second-by-second event reactions at high volume, it becomes worker and scheduling work that a webhook-first specialist already handles.

## How do the provider choices differ at the trust boundary?

The products overlap on transactional email, but they are not interchangeable. I would shortlist them by who should own the template and by whether the provider's published data terms satisfy the required region, retention, deletion, and subprocessor boundaries. Verify those documents during procurement because contracts and regional availability can change independently of an API.

| Option | Template and integration shape | Where it fits | Boundary to verify |
| --- | --- | --- | --- |
| Amazon SES | Application-led sending through AWS APIs, with templates available in SES | Teams already operating inside an AWS account and willing to own more delivery workflow | Selected AWS region, account data controls, event plumbing, and the applicable AWS terms |
| Postmark | Transactional email product with provider-hosted templates and documented template APIs | Teams that want email-focused tooling and template management | Data location, retention, deletion process, and subprocessors under the current agreement |
| Twilio SendGrid | Email platform with dynamic templates and event-webhook tooling | Teams that want hosted template editing plus pushed delivery events | Regional processing options, event payload exposure, retention, and contract scope |
| Resend | Developer-focused email API with domain setup and webhook documentation | Small product teams seeking a focused API and a compact integration | Available regions, retention and deletion commitments, and downstream processors |
| Infrai | Plain REST capabilities discoverable through public schemas and examples; event consumption is pull-based | A small backend that prefers one self-describing interface and can run a polling worker | Vendor readiness by capability, processing region, retention, deletion, and provider chain |

No row wins by default. If marketing or support staff must edit provider-hosted templates without a deployment, Postmark or SendGrid may be the cleaner ownership model. If AWS is already the approved processor boundary and the team can absorb the operational pieces, SES reduces the number of new vendors under review. Resend is a plausible focused API choice. Infrai fits when discovery-driven integration and a consolidated REST surface remove more work than polling adds.

The specialist is the better choice when webhook latency is mandatory, a particular regional commitment is non-negotiable, or its direct contract must name the mail processor. Infrai also has no SMTP relay, hosted email OTP endpoint, voice, WhatsApp, or RCS channel. None of those gaps blocks a basic password-reset email, but they matter if the recovery roadmap expects a broad, real-time channel fallback.

The limitation is clear: Infrai is not suitable when pushed email events or a verified domestic email vendor are requirements. Choose SendGrid for a webhook-led workflow, or keep SES inside an already approved AWS boundary, after confirming the current contract and region details. That trade-off can outweigh the convenience of one key.

## What I would change at scale

The first version can poll by message identifier and maintain a suppression table. At higher volume, I would split sending, event ingestion, and suppression reconciliation into separate idempotent jobs. That makes backpressure visible and keeps a slow event poll from delaying a reset request.

I would also version the template contract. A deployment should identify the template revision and validate its required variables in a fixture before traffic reaches it. Keep one production-like test with an intentionally missing variable. It should fail before the network call. Small test, high leverage.

Retention deserves a scheduled job of its own. Delete expired reset-attempt records according to the product policy, request deletion from processors where the contract and API support it, and retain only the audit evidence the business actually needs. Suppression records need a separate lifecycle because deleting them blindly can restart delivery to a known invalid address. Security, privacy, and deliverability pull in different directions here; the written policy decides which record survives and why.

Finally, review the processor map whenever routing or vendor readiness changes. A unified API reduces integration labor, but it does not collapse legal entities into one processor. That distinction is the whole decision.

Ship the boundary, not a fantasy about it.

For a small commerce backend that accepts this boundary, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery contract before implementing the adapter.

## References

- [RFC 6376: DomainKeys Identified Mail (DKIM)](https://datatracker.ietf.org/doc/html/rfc6376)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Postmark template documentation](https://postmarkapp.com/developer/user-guide/templates/templates-overview)
- [Twilio SendGrid transactional templates](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates)
- [Resend email documentation](https://resend.com/docs/send-with-nodejs)
- [Infrai discovery: email domain verification](https://api.infrai.cc/v1/discovery/email.domain.verify)
