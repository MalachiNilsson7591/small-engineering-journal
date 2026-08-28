# Delivery Evidence for Compliance Notices: Password Reset Email and SMS Backup in Node.js

A compliance notice is a message you may have to defend a year after you sent it. In a developer-tools product the password reset itself is the easy part — a token, a 15-minute expiry, one screen. Use email as the channel of record, keep the reset flow email-only by default, and treat an SMS backup as a separate attempt with its own record instead of a silent fallback that fires whenever an email looks slow. That is the practical strategy for a small US/EU Node.js service, and the reasoning has little to do with which channel feels faster: email is the channel that hands you durable, standard-shaped evidence, and evidence is the thing a customer's security review asks for.

Evidence first. Channels after.

The constraint that decides this isn't deliverability. It's the question in the review: show me what this user was sent, when it left, and what the receiving system said back. If the answer is a log line that reads `email sent: true`, there is no record there. That's a memory of an intention.

## What makes a delivery record auditable?

Three layers, and they prove different things. The first is what your application decided and rendered — recipient, template revision, the exact bytes, the actor who triggered the notice. You own that layer completely, which is why it's the only one you can promise anything about. The second is the handoff: a receiving SMTP server answering 250 with a queue identifier is accepting responsibility for the message, which is what the reply-code semantics in RFC 5321 actually mean. The third layer is what the far side reports afterward, and it is the thinnest of the three: hard failures come back as delivery status notifications in the RFC 3464 format, complaints arrive out of band through feedback-loop reports, and silence carries no information at all.

Silence is not delivery.

| Artifact | What it proves | What it does not prove |
| --- | --- | --- |
| Rendered body plus template revision | The exact wording sent, at a known version | That anything left your network |
| 250 reply with queue id | A receiving server took responsibility | That the mailbox exists or a person read it |
| Delivery status notification | A specific failure, with a status code | Anything about messages that never bounce |
| Carrier status for an SMS attempt | An operator accepted or reported handset delivery | Who was holding the handset |
| Your own append-only event rows | Your decision, actor, and timing | Any recipient-side behavior |

SMS evidence has a different shape. A carrier status tells you an operator accepted the message or reported a handset delivery, not who held the handset, and the retention of those statuses is set by the operator chain rather than by you. For account recovery that distinction is the whole argument: a phone number is a weaker identity claim than a mailbox the account was verified against, and in the EU it's one more category of personal data you have to justify keeping. (Both channels can still share one evidence table — the schema is identical, only the status codes differ.)

Templates belong in the record too, which means they need revisions. A logic-less syntax like Mustache fits this job because a non-developer can review the copy without being handed a code-execution surface, and the rendered output stays a pure function of a data model you can store next to the notice row.

## What does a password reset in Node.js gain from an SMS backup instead of email-only?

Email-only, until a named population justifies the second channel. Stay email-only when the mailbox is the account identity, when the notice carries no deadline shorter than the mailbox's realistic latency, and when you can retain the evidence for the whole audit window you promise customers. Add SMS when a specific group genuinely cannot reach the mailbox — in developer tools that's usually a shared team inbox behind an admin who left — or when the notice has a deadline measured in hours rather than days.

The catch is that a second channel doubles the evidence surface without doubling the reliability. Two retention policies, two processors to name in a subprocessor list, two vocabularies of status codes to normalize into your own, and a phone number stored against an account that previously needed only an address. That's a real trade-off, and it's the kind that costs a one-person team a weekend every quarter. If you need the second channel for support reasons, take it — just don't file it under compliance evidence, because the SMS trail is the weaker of the two records.

The failure I would design against first isn't a bounce, though. It's routing a security notice down the same path as marketing mail. US commercial email rules turn on the primary purpose of a message, and transactional or relationship messages sit in a different bucket from promotional ones; the FTC's compliance guide spells out that split. Put both through one pipeline and an unsubscribe from a product newsletter can suppress a password reset, which is a support incident and an awkward paragraph in the next questionnaire. Keep the notice path separate in code: its own credentials, its own suppression rules, its own template store, its own dashboard.

## A minimal Node.js code path that records the evidence

The interesting part is the ordering, not the sending. Write the record before the dispatch, in the same transaction that creates the reset token, and let the row id be the idempotency key.

```ts
import { createHash, randomUUID } from "node:crypto";

type Channel = "email" | "sms";

interface NoticeRecord {
  id: string;
  userId: string;
  kind: "password_reset";
  channel: Channel;
  templateRevision: string;
  bodyHash: string;
  createdAt: string;
  handoffId?: string;
}

interface Store {
  insertNotice(row: NoticeRecord): Promise<void>;
  attachHandoff(id: string, handoffId: string): Promise<void>;
  appendEvent(id: string, ev: { at: string; source: string; code: string; raw: string }): Promise<void>;
}

// One transport shape per channel. An adapter returns the identifier the far
// side handed back and nothing else; policy stays in the application.
interface Transport {
  send(to: string, body: string): Promise<{ handoffId: string }>;
}

export async function sendNotice(
  db: Store,
  transports: Record<Channel, Transport>,
  input: { userId: string; to: string; channel: Channel; templateRevision: string; body: string },
): Promise<NoticeRecord> {
  const row: NoticeRecord = {
    id: randomUUID(),
    userId: input.userId,
    kind: "password_reset",
    channel: input.channel,
    templateRevision: input.templateRevision,
    bodyHash: createHash("sha256").update(input.body).digest("hex"),
    createdAt: new Date().toISOString(),
  };

  await db.insertNotice(row);

  try {
    const { handoffId } = await transports[input.channel].send(input.to, input.body);
    await db.attachHandoff(row.id, handoffId);
    await db.appendEvent(row.id, { at: row.createdAt, source: "transport", code: "accepted", raw: handoffId });
    return { ...row, handoffId };
  } catch (err) {
    // Unknown outcome, not a negative one. Keep the row, retry with the same
    // id later, and let the reconciler decide once a status arrives.
    await db.appendEvent(row.id, { at: new Date().toISOString(), source: "transport", code: "unknown", raw: String(err) });
    throw err;
  }
}

// Status callbacks and bounce reports append. They never overwrite.
export async function ingestStatus(db: Store, id: string, ev: { source: string; code: string; raw: string }) {
  await db.appendEvent(id, { at: new Date().toISOString(), ...ev });
}
```

Everything load-bearing in that file is about ordering and immutability. The record exists before the network call, so a process that dies mid-dispatch leaves a notice with no terminal status rather than a send with no notice — the first is a queue item, the second is a hole in the audit trail. Retries reuse the row id, so a transport timeout that later turns out to have succeeded produces one record with two events instead of two records that disagree. Events append and never mutate, which is what lets you answer "what did we know, and when" months later rather than "what does the row say now". Store the raw payload of every status you receive, in whatever shape it arrived, because normalizing at write time throws away the field you'll need for a question nobody has asked yet. And keep the reset link, the token, and the full phone number out of ordinary logs — the evidence table is the place where sensitive material is deliberate, controlled, and covered by a retention rule. One honest gap: I'm not sure a body hash alone satisfies every reviewer. Some want the rendered message itself, which turns into an encrypted copy plus a decision about how long you keep it, and that decision is a conversation with counsel rather than an architecture question.

## Rollout notes: an outbox worker, region splits, retention

The dispatch call moves behind an outbox worker, so the request path only writes rows. The evidence store gets partitioned by region, with EU notices staying in an EU database, because a data-residency answer is much cheaper as a schema decision than as a migration. I'd also add an export command that dumps one notice with its full event history as a single JSON document, since that document is the artifact a reviewer actually wants, and a monitor for notices with no terminal event after 15 minutes.

## Where the record stops helping: failure modes worth naming

Proof of dispatch is not proof of receipt, and no amount of event plumbing changes that. If a regulation in your market requires acknowledged receipt, neither channel is suitable on its own and you need an in-product acknowledgement, a signed statement, or registered post — stick with whatever your legal counsel already accepts as service of notice. If your product is consumer-facing with a long tail of abandoned mailboxes, email-only recovery will strand real users, and that's a reachability decision your support data should make, not your audit story. If you need per-recipient read confirmation, none of this provides it; the open pixel is unreliable by design in modern mail clients, and treating it as evidence is worse than having no evidence at all.

## Further reading

- https://mustache.github.io/mustache.5.html
- https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
- https://datatracker.ietf.org/doc/html/rfc5321
- https://datatracker.ietf.org/doc/html/rfc3464
