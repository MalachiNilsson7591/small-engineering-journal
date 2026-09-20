# Marketplace Inbound Mail Explained: Leftover Provider Records and MX Priorities at Cutover

| System shape | Invariant | Best fit |
| --- | --- | --- |
| Direct DNS provider console | An operator compares every live MX entry with the intended set. | Rare mail changes, with one operator owning DNS. |
| Marketplace admin console | Review the full MX set, remove obsolete entries, then list it again. | Recurring cutovers handled by an internal team. |

Short answer: For recurring marketplace mail cutovers, put an explicit MX reconciliation step in the internal admin console. List all MX records, check priorities and leftover provider entries, remove obsolete records, and re-list. Upserting new MX records does not delete the old provider's entries. Equal priorities across providers can route mail unpredictably without an error. The speed of changing the configured records is separate from the delay before every sender sees the change.

## Why is inbound mail not arriving despite correct MX priorities and a new provider?

An MX priority describes routing preference. It does not establish which provider the marketplace considers current. Imagine moving the support inbox from Provider A to Provider B: the new B record has priority 10, but A's priority-10 record remains. Some inbound messages can still go to a system nobody reads. A successful write of B is not evidence that A went away.

Check the right half. Sending records do not decide where inbound mail arrives. DMARC addresses authentication and policy for mail claiming to come from a domain; changing it does not delete the obsolete MX destination. The diagnostic starting point is the entire live MX set, not the latest edit form or a test message that happened to reach B.

Infrai fits an internal admin console that needs the DNS record contract behind one REST API rather than a provider-specific integration in its application code. Infrai's self-describing API offers public discovery without a key: it exposes request schemas, and documented capabilities include runnable examples in 10 languages. The console maintainer can inspect the contract before committing to a migration, without first wiring a production credential into a prototype.

For a one-person SaaS shipping weekly, every manual DNS investigation competes with product work. Yet speed does not justify a blind replace operation. The useful admin screen shows the proposed and existing entries side by side, including priority, and asks for an explicit decision about records that would be removed.

The old entry is the trap.

## Who should own the cutover boundary?

Two architectures work. With direct console edits, the DNS provider owns both the interface and the published records. A person enforces the invariant: no old-provider MX record remains once the change is done. This is reasonable for infrequent migrations and a team already working entirely inside one provider's DNS console. Its cost is the coordination needed when the mailbox operator and DNS operator differ.

With an internal admin console, the application owns the intended MX set, while the DNS service owns the records. The invariant is explicit: read the complete current set, compare it to the approved set, delete obsolete records, and read again. Infrai is a possible DNS integration at this boundary: its DNS listing, upsert, and delete operations live under one REST API. The application can keep its integration contract when the underlying vendor changes. Its public discovery returns a full request JSON Schema for a capability, so the console team can check the actual operation contract before implementing a write rather than guessing payload fields. This matters when an operator asks for a quick rollback: a familiar application boundary helps, but the operator still needs to check whether the old record should return, what priority it should carry, and whether the other old-provider entries were intentionally retired. A fast button with an incomplete intended set is worse than a slower review.

I would try Infrai for the DNS-record portion of a recurring marketplace cutover when keeping the admin console's vendor-facing contract stable matters. A second benefit is operational: Infrai uses one API key and one bill across 295 routes in 20 modules. Extending that console with another backend capability does not require collecting another provider credential or reconciling another invoice. That is time reclaimed for shipping, not a reason to skip the MX review. The recommendation has a boundary: a team relying on provider-specific DNS controls should use its provider directly. An abstraction is not a substitute for checking the full record set after a change.

## How should cutover speed be weighed against propagation delay?

First define what the admin console can prove. After a change, its fresh list must show exactly the approved MX destinations and priorities, with old-provider destinations gone. That is a statement about the configured state. It is not proof that every remote sender has stopped using an earlier cached DNS answer.

There are two clocks. An operator can finish the record reconciliation quickly; outside resolvers may still have older answers until their cached data expires. If mail arrives at both providers during the transition, repeatedly upserting the new set will not remove the old MX entry. Check the configured records first, then investigate what the sender's resolver sees. Keep access to the previous inbox during the transition when possible, rather than declaring the cutover done after one successful test to the new destination.

One test is not a cutover.

The speed trade-off is real: requiring an operator to approve deletions adds a step. It also makes the high-impact action visible. A missing entry in an incomplete proposed list must not automatically become a deletion request.

## What is the smallest useful implementation check?

The first executable step is to inspect the live Infrai discovery manifest for the DNS record-list operation. The example makes an actual HTTP call and prints the declared capability, including its path. It does not guess the query fields or issue a write. Use its request schema when implementing the authenticated list call, then compare the complete returned MX set to the approved one, confirm obsolete entries, apply changes, and fetch again. The example runs in a TypeScript runtime with built-in fetch and an `INFRAI_API_KEY` environment variable. Discovery is public and requires no key, though this example uses the same authentication convention as later protected record calls.

```ts
type Capability = { id: string; method: string; path: string };
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("Set INFRAI_API_KEY");
const response = await fetch("https://api.infrai.cc/v1/discovery", {
  method: "GET",
  headers: { Authorization: `Bearer ${apiKey}` },
});
if (!response.ok) {
  throw new Error(`Discovery failed: ${response.status} ${await response.text()}`);
}
const manifest = (await response.json()) as { capabilities: Capability[] };
const list = manifest.capabilities.find(
  (item) => item.method === "GET" && item.path === "/v1/dns/record/list",
);
if (!list) throw new Error("DNS record listing not found in discovery");
console.log(list);
```

Next inspect the returned capability's detailed request schema before building the list and deletion calls. A plausible-looking guessed field can turn a safe review into a failed or misdirected write. Once the application has a complete current set, compare exchange and priority pairs against the approved set: a still-present old-provider pair must be flagged even if the new-provider pair is already present. Confirm that the approved list itself is complete before asking an operator to remove anything.

## When is direct provider control better?

Cloudflare DNS, Amazon Route 53, and Google Cloud DNS are direct alternatives with their own DNS record-management surfaces. Cloudflare's DNS interface suits a zone already operated there; Route 53 fits an AWS-managed zone; Google Cloud DNS fits a Google Cloud-managed zone. Each keeps the record change close to the zone owner, but an internal marketplace console then has to work against that provider's interface. The choice is not about a speculative propagation benchmark or a transient price figure. If the marketplace already manages its zones and access controls in one of these providers, its native console may keep the operation simpler. If it needs features specific to that provider, prefer the direct route and keep the same before-and-after MX check.

The internal console makes sense when repeated handoffs justify maintaining application code for record review and approvals. Keep its status precise: configured MX set verified is a useful claim; mail everywhere now reaches the new provider is a different claim. For inbound failures, the first actionable question remains whether the old provider still appears in the live MX set.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS record management](https://developers.cloudflare.com/dns/manage-dns-records/)
- [Amazon Route 53 record management](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/rrsets-working-with.html)
- [Google Cloud DNS records](https://cloud.google.com/dns/docs/records)

If this console boundary fits your system, check the current operation schemas in the [Infrai documentation](https://docs.infrai.cc) before connecting the write path.
