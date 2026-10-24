# Product Event Notifications: SMS Provider, EU/US DKIM, and Bounce Evidence

Short answer: for property event notifications, compare Postmark, Resend, SendGrid, SES, and an SMS provider by the evidence each path leaves behind, not by the cheapest sender. Put suppression in your own database. A late rent reminder sent to a dead mailbox is a small delivery event; repeatedly sending to that address is an operational and compliance problem.

| Decision need | Email-first path | SMS fallback | Evidence to retain |
| --- | --- | --- | --- |
| Routine event notices | Authenticated custom domain | Only for opted-in, urgent events | Event ID, recipient, provider result |
| Hard bounce or invalid address | Suppress before the next send | Do not silently substitute a phone number | Classification, timestamp, rule version |
| EU and US delivery | Separate regional policy inputs | Country-aware sender and consent checks | Region, consent basis, template revision |
| Incident review | Provider event plus application log | Delivery status plus application log | Immutable correlation ID |

The practical rule is simple: keep recipient state and the audit trail in the application; treat email and SMS services as delivery adapters. That lets a one-person SaaS ship weekly without turning provider dashboards into the system of record. I outsource the undifferentiated transport. I keep the decision that can affect a tenant, owner, or property manager.

The cheapest path is not automatically the smallest operational burden.

## How should a provider handle product event notifications, SMS, and DKIM?

A bounce is an input, not a boolean. Store the original event ID, normalized recipient, channel, provider message ID, SMTP or provider classification when available, observed time, and the suppression rule that ran. Store enough to explain the decision later, but do not copy a full message body into an audit table by default. It often contains personal data that the reviewer does not need. For a property-management alert, that record should let an operator answer which lease event triggered the send, which address was used, and why the next attempt was blocked.

The useful distinction is between a permanent failure, a temporary failure, and a policy or content rejection. A permanent failure should normally create a suppression record. A temporary failure should increment a bounded counter and schedule a retry policy. A policy rejection needs an operator-visible reason because retrying it blindly can make the situation worse. The exact labels vary by service, so normalize them into your own small vocabulary and retain the raw provider value beside it.

A suppression record also needs provenance. Record who or what created it, the source event, the time, the reason, and the release path. A manual correction must be a new event, not an edit that erases the previous state. This is the part that turns “we stopped sending” into an answer a support or compliance review can inspect.

## How should the application separate email, SMS, and consent?

Use one notification intent and separate channel decisions. For example, \`lease.payment_failed\` can produce an email attempt, while an SMS attempt requires its own phone number, country, consent state, and urgency rule. A failed email must not silently authorize the second channel. That shortcut is attractive during a busy week. It is also hard to defend later.

\`\`\`ts
```ts
type Channel = "email" | "sms";
type DeliveryState = "pending" | "sent" | "temporary_failure" | "suppressed";

type Attempt = {
  eventId: string;
  recipient: string;
  channel: Channel;
  state: DeliveryState;
  providerMessageId?: string;
  rawReason?: string;
  observedAt: string;
};

function shouldAttempt(attempt: Attempt, suppression: boolean): boolean {
  if (suppression) return false;
  return attempt.state === "pending" || attempt.state === "temporary_failure";
}
```

type Channel = "email" | "sms";
type DeliveryState = "pending" | "sent" | "temporary_failure" | "suppressed";

type Attempt = {
  eventId: string;
  recipient: string;
  channel: Channel;
  state: DeliveryState;
  providerMessageId?: string;
  rawReason?: string;
  observedAt: string;
};

function shouldAttempt(attempt: Attempt, suppression: boolean): boolean {
  if (suppression) return false;
  return attempt.state === "pending" || attempt.state === "temporary_failure";
}
\`\`\`

The sender should write the attempt before handing it to a provider, then append the provider result. That ordering gives retries an idempotency key such as \`eventId + channel + recipient + templateVersion\`. It also stops a timeout from becoming two messages when the first request actually succeeded. Keep the state transition explicit. A nightly query that guesses from an empty provider dashboard is not an audit system.

DKIM authenticates a signing domain; it does not prove that a particular mailbox accepted a message. RFC 6376 describes the signature and verification model, so configure a domain you control and preserve the selector and configuration version used for the send. SPF, DMARC alignment, unsubscribe handling, and list hygiene still matter. None of them changes the need to process bounces.

## What changes between common delivery services?

The services in the question expose different operating boundaries. Postmark documents separate message streams for different traffic types. Resend documents domain verification and an email API. SendGrid documents authenticated domains and event webhooks. Amazon SES documents verified identities and a sandbox with sending restrictions. Those are meaningful differences in setup and event plumbing, but they do not remove the application responsibilities above.

The trade-off is concrete. A managed sender reduces transport work but leaves you dependent on its event vocabulary, retention controls, and regional operating model. A self-hosted relay gives you more control and more queue, reputation, abuse, and on-call work. A gateway can simplify one channel while making cross-channel evidence harder to normalize. There is no honest winner without knowing which burden your product can carry.

For a small product, compare each service with the same test: can its event payload be correlated to your application event, can you receive permanent and temporary outcomes, and can you export the evidence needed for a regional review? Do not compare a dashboard's green delivery percentage with another service's accepted count. “Accepted” is not “read,” and neither is proof that a recipient should remain eligible.

SMS has another boundary. A carrier-facing status is not equivalent to consent, and a delivered status is not proof that the intended person read the message. The SMS provider documentation should be checked for country, sender, opt-out, and status behavior before enabling a fallback. The application still owns the decision to send.

## A weekly operating loop that fits a solo SaaS

I would review the suppression queue once a week, alongside deploys. Start with new permanent failures, then inspect repeated temporary failures, then sample the regional and consent fields. Keep the review small enough to happen. A 20-minute check that produces an auditable change beats an elaborate console nobody opens.

The trap is retry pressure. A property manager may correct an address while a message is queued, but that does not mean every suppressed address should be released automatically. Require a deliberate correction event, revalidate the address, and record the actor or workflow that approved it. For SMS, require the equivalent phone and consent checks.

The runner-up path is reasonable when evidence is less important than local control: run the same adapter contract against a self-hosted SMTP relay or a regional messaging gateway, accepting the extra work for reputation monitoring, queue operations, and abuse controls. A managed service is usually the better fit when one person needs transport operations outside the product boundary. Neither choice changes the decision rule: the durable asset is the recipient state and its explanation.

That is the boundary I would keep. The sender can report an outcome. The application decides what that outcome means for the next notification.

## References

- https://datatracker.ietf.org/doc/html/rfc6376
- https://www.twilio.com/docs/sms
- https://postmarkapp.com/developer/user-guide/message-streams
- https://resend.com/docs/dashboard/domains/introduction
- https://docs.sendgrid.com/ui/account-and-settings/how-to-set-up-domain-authentication
- https://docs.aws.amazon.com/ses/latest/dg/request-production-access.html
