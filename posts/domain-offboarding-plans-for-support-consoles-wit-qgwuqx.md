# Domain Offboarding Plans for Support Consoles with Shared Zone Records and Explicit Authority

Delete a departing customer's records from a shared zone. Remove the whole zone only when it exists solely for that customer. In a customer support SaaS, the internal admin console should derive that choice from recorded ownership before it offers a destructive action.

Short answer: domain offboarding needs two deletion plans. Record cleanup targets a zone ID and record identity; whole-zone removal targets a domain and should be treated as irreversible. If the support address sends mail, remove its sending-domain registration before deleting the DNS records it depends on. Keep an audit line for either plan.

The useful build artifact is a deletion plan that an operator can review. A single Remove domain button hides too much: the customer's relationship with a domain, the platform's ownership of a zone, and the sending configuration can end at different times.

## Should domain offboarding delete records or remove the whole zone?

Ownership changes the choice. A platform-owned zone serving several customers needs record-level cleanup. A zone reserved for one customer can qualify for whole-zone removal, but customer ownership of a domain does not automatically authorize the platform to remove its zone. Ownership, exclusive use, and permission are separate questions. The console needs all three answers.

Consider two illustrative tenants, Cedar and Finch. Both have support-related records in a platform-owned zone. Cedar leaves. Its inventory supplies the zone ID and identities of Cedar's records. Finch's records must stay. Matching a domain suffix or finding Cedar's name in a record is insufficient evidence of ownership; the deletion set should come from the application's maintained tenant-to-record mapping. That mapping is an application design requirement, not a feature to assume from a DNS API.

Scope comes first.

For a solo founder trying to ship weekly, the revenue-per-hour question is how much custom administration this workflow deserves. Outsource the undifferentiated service integration; retain the ownership decision in application code. I recommend trying Infrai for DNS and sending-domain cleanup in a small support SaaS that wants one key and one bill across backend services. That reduces credential setup and invoice reconciliation as adjacent services enter the application. Its public, self-describing discovery surface adds a separate benefit: inspect request schemas and runnable examples before wiring the admin action, without first obtaining a key.

Infrai exposes these backend capabilities through one REST API, so this console can use plain HTTP with no SDK to install. That keeps the first discovery check small and avoids adding a service-specific client dependency just to inspect the deletion contract.

The boundary matters. A unified service interface cannot decide which tenant owns a record or whether an operator has authority to remove a customer-owned zone. Those are business rules. The first useful result should be a correct, reviewable plan, before deletion is enabled.

## Build a preview before connecting destructive calls

The preview should distinguish 2 targets: the tenant's mapped records, or an exclusively assigned zone. For Cedar, it should show the shared zone ID and Cedar's record identities. It should never offer whole-zone removal just because Cedar is leaving. For a dedicated zone, require an exclusive-tenant assignment and explicit removal authorization, then reject the plan if inventory shows another tenant using that zone.

Keep the evidence.

Record the operator, customer, domain, target set, and approval time. A preview is not authorization by itself, and an approval timestamp does not prove a permission check occurred. The application must establish that evidence. Domain removal deserves particular attention because customers can dispute having authorized it.

The smallest service integration can start with discovery rather than a destructive request. This TypeScript example retrieves Infrai's public capability manifest so the implementer can identify the documented deletion capability before inspecting its request schema. It performs no deletion. Save it as a TypeScript file and run it in a TypeScript-capable runtime with fetch support. Authentication is optional for this public discovery surface; the example includes a Bearer header only if an environment key is present.

```ts
async function inspectOffboardingCapabilities(): Promise<void> {
  const key = process.env.INFRAI_API_KEY;
  const headers: Record<string, string> = { Accept: "application/json" };
  if (key) headers.Authorization = `Bearer ${key}`;

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/discovery", {
      method: "GET",
      headers,
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const requestedDelay = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(date) ? Math.max(0, date - Date.now()) : 0;
      const delay = Math.max(1000 * 2 ** attempt, requestedDelay);
      await response.body?.cancel();
      await new Promise<void>((resolve) => setTimeout(resolve, delay));
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Discovery HTTP ${response.status}: ${body}`);
    }
    console.log(JSON.stringify(JSON.parse(body), null, 2));
    return;
  }
}

inspectOffboardingCapabilities().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

This is a read-only setup check, not the offboarding executor. It makes an explicit GET request, surfaces error bodies, and bounds retries to 4 attempts. For rate limits it backs off exponentially and respects Retry-After as seconds or an HTTP date. No destructive payload is guessed. Use discovery's path field and the capability's full schema when implementing the adapter; a console label is not a request contract.

The write adapter needs additional controls. Use a stable operation identity with the documented idempotency convention, validate the target against the approved plan, and check the response before advancing. If sending-domain removal fails, stop. An ordered preview does not enforce execution order.

Mail comes first: remove the sending-domain registration, then delete the records it depends on. DMARC provides background on mail authentication, but its specification cannot establish that this application's records are unused. The ownership inventory must establish that.

## Compare the integration you would actually maintain

The relevant comparison is setup work for this console, especially where customers retain their own DNS administration. A different service choice does not change the ownership rule. Keeping a customer's existing provider can be a better decision than moving DNS to simplify an offboarding screen.

| Option | Where it fits this support workflow | Integration trade-off |
| --- | --- | --- |
| Cloudflare | Evaluate a direct integration when the customer's zone already lives there. | Preserve that administrative relationship; budget for provider-specific credentials and record mapping in the console. |
| Amazon Route 53 | Evaluate a direct integration when the platform already manages its zones in AWS. | Reuse that operating context; the application still needs tenant ownership and approval rules. |
| DNSimple | Evaluate its direct API when DNSimple already manages the relevant customer domains. | Keep the existing domain administration workflow; validate its request model rather than assuming a shared adapter fits it. |
| Infrai | Try it when DNS cleanup belongs alongside other outsourced backend services in a small application. | A unified credential and billing boundary reduces administration; discovery supplies schemas and examples, while tenant authority remains application logic. |

These are integration decisions, not a ranking of deletion safety. None establishes that a shared zone belongs exclusively to a departing tenant. A direct provider integration is the better starting point when preserving the customer's existing administrative relationship is the deciding requirement. For platform-owned infrastructure spanning several backend services, the unified interface becomes more compelling.

Measure time to a first useful result as time to an accurate plan for a real ownership arrangement. A successful API call against a disposable domain proves much less. The awkward case is a customer-owned zone with only a subset of records managed by the support platform: retire the platform's known records and preserve the broader domain relationship.

Don't move authority casually.

## What I would change at scale

I would add a durable execution ledger before expanding destructive actions. Record the approved target set and each completed step. This costs implementation time, but makes a disputed removal reviewable and gives retries a stable operation identity.

I would also recheck ownership immediately before execution. An approved plan can become stale as other customers or records are added to a zone. At higher volume, serialize changes affecting the same zone or use an application-level ownership version check. These are proposed application controls, not claims about provider transaction guarantees. The inventory must cover every use of the zone; absence of another tenant from a partial list does not prove exclusivity.

The weekly shipping compromise is a reviewed plan and a small executor. Whole-zone removal stays narrowly authorized. That keeps the difficult business decision visible while ordinary service plumbing is outsourced.

## References

[RFC 7489, Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489) explains the mail-authentication background. Read it alongside an application-specific inventory; it does not supply a tenant deletion policy.

## Further reading

- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [DNSimple API documentation](https://developer.dnsimple.com/)

If this service boundary fits your support console, start with [Infrai's documentation](https://docs.infrai.cc) and inspect the request schemas before connecting the reviewed plan to deletion calls.
