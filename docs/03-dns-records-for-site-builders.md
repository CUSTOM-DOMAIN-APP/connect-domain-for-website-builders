# DNS Records for Site Builder Custom Domains

This page lists the exact DNS records a site builder platform needs on a customer's domain, explains the apex problem and CNAME flattening, and covers the records your flow must never touch. It is the technical companion to [the connect flow UX](02-the-connect-flow-ux.md) in the [custom domains for website builders](../README.md) guide from [CustomDomain](https://customdomain.ai).

## The records at a glance

A site connection needs remarkably few records. Complexity comes from provider interfaces and the apex, not from volume.

| Record | Name | Purpose |
|---|---|---|
| CNAME | `www` (or another subdomain) | Aliases the hostname to your platform endpoint |
| A / AAAA | apex (`@`) | Points the bare domain at your edge IPs when no alias feature exists |
| TXT | a dedicated verification hostname | Proves control of the zone, when your design needs a challenge record |
| CAA | apex, optional | Authorizes your certificate authority to issue for the domain |

## The subdomain record: CNAME

For `www` and any other subdomain, use a CNAME to an endpoint in your zone:

```
www.mybakery.example.    3600  IN  CNAME  sites.yourplatform.example.
```

The CNAME's value is indirection you control. When your edge IPs change, you update `sites.yourplatform.example` in your own zone once, and every connected customer domain follows automatically. This is why CNAME targets beat handing out raw IPs wherever a CNAME is allowed.

Two details worth getting right in your instructions:

- **The trailing dot.** In zone-file syntax, `sites.yourplatform.example.` with a trailing dot is an absolute name. Most provider web interfaces neither need nor accept it. Generate instructions per provider so users are never left guessing, because "do I type the dot?" is a genuinely common support ticket.
- **One CNAME per name, nothing else beside it.** A CNAME cannot coexist with any other record at the same name. If the user has an old A record on `www`, it must be removed, and your flow should say so explicitly.

## The apex problem

The apex (root, naked domain) is `mybakery.example` itself, and the DNS standard forbids a CNAME there. The rule (from [RFC 1034](https://www.rfc-editor.org/rfc/rfc1034)) is that a CNAME excludes all other record types at the same name, and the zone apex must always carry SOA and NS records. So a standards-compliant CNAME at the apex is impossible, and any provider that appears to allow one is doing something else behind the scenes.

That leaves three real options:

**Option 1: A and AAAA records.** Universally supported, works everywhere:

```
mybakery.example.    3600  IN  A      203.0.113.10
mybakery.example.    3600  IN  AAAA   2001:db8::10
```

The cost is that the customer's zone now pins your IP addresses. If your infrastructure moves, every connected apex breaks until its records are updated. Platforms handle this by publishing stable anycast IPs designed never to change, or by holding write access to the zone (via provider authorization or API token) so records can be updated automatically when needed.

**Option 2: Provider alias features (ALIAS, ANAME, CNAME flattening).** Many DNS providers offer a nonstandard record that behaves like a CNAME at the apex: the provider itself resolves your endpoint's A/AAAA records and serves the resulting addresses as if they were native apex records, re-resolving as they change. Different providers brand it differently, but the mechanism, often called CNAME flattening, is the same. Where available, this gives you apex support without pinning your IP addresses in the customer's zone. Your flow should detect the provider and prefer this automatically.

**Option 3: Redirect the apex to `www`.** Some registrars offer HTTP-level apex forwarding to `www`, which then carries a normal CNAME. This works but adds a redirect hop and an extra provider feature dependency, so treat it as a fallback, not a default.

A good connect flow chooses among these per provider without asking the user anything. This is precisely the kind of provider-by-provider knowledge that is tedious to build and maintain in-house, and it is a core part of what [CustomDomain](https://customdomain.ai/one-click-dns-setup) handles across its 63 catalogued providers, 25 of them with an automatic path. Whether a given provider supports flattening is exposed per domain by the pre-flight check (`cnameFlattening` on `POST /v1/domains:check`), so your UI can branch on it rather than guessing.

## The verification record: TXT, and when you do not need one

If your platform generates its own ownership challenge, the shape is a unique token at a hostname reserved for the purpose:

```
_verify.mybakery.example.   300  IN  TXT  "yourplatform-verification=7f3a9c1b24d8..."
```

Keep the TTL short (300 seconds is typical) so verification polls see the record quickly, and keep the hostname namespaced (an underscore prefix by convention) so it never collides with real service records.

A challenge record is one way to prove control, not the only way, and it costs the user an extra record to copy. If the pointing records were written through a channel that already required the zone owner to authenticate, the write itself is the proof. CustomDomain works this way and issues no TXT challenge at all: an OAuth authorization, a one-click setup apply, or a scoped API token each demonstrate control directly, and on the guided manual rail the poller waits for the pointing records to resolve to their expected values, which is the same evidence a challenge would have produced.

Whichever mechanism you choose, the ordering is not negotiable: control must be proven before certificates are issued and before traffic is routed. The reasons are covered in [how site builders offer custom domains](01-how-site-builders-offer-custom-domains.md), and the consequence of getting it wrong is in [when a customer leaves](06-when-a-customer-leaves.md).

## Records your flow must never touch

The fastest way to turn a happy customer into a furious one is to break their email while connecting their website. These records are out of scope for a site connection, always:

- **MX**: mail routing.
- **TXT at the apex containing `v=spf1`**: sender authorization for their mail.
- **DKIM (`*._domainkey`) and DMARC (`_dmarc`) records**: mail authentication policy.

A scoped flow adds web records and, if your design uses one, a verification TXT, and nothing else. This scoping is also central to the consent model in one-click provider authorization: the user approves a specific, limited set of changes, a design the Domain Connect protocol (an open standard maintained by a community of developers across multiple companies, whose CC0 [knowledge base](https://github.com/Domain-Connect/knowledge-base) informs parts of this guide) formalizes with vetted templates and provider-rendered consent screens.

If your builder does also set up mail for a connected domain, that is a separate product surface with its own failure modes, and the [email platforms field guide](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-email-platforms) covers it.

## CAA: the quiet certificate blocker

CAA records declare which certificate authorities may issue for a domain. Most domains have none, and issuance proceeds normally. But when a customer's domain carries a restrictive CAA record (often added by a previous provider or an IT policy), certificate issuance for your platform fails even though every pointing record is perfect. Check CAA during connection and surface it with the exact record to add, because "your site is connected but shows a security error" is otherwise nearly undebuggable for a user. More in [scaling TLS for tenant domains](04-scaling-tls-for-tenant-domains.md).

## TTL choices

| Record | Suggested TTL | Why |
|---|---|---|
| Verification TXT | 300 | Fast polling, short-lived purpose |
| CNAME / A / AAAA during setup | 300 to 3600 | Quick correction if the user makes an error |
| Established records | 3600 or higher | Fewer resolver queries once stable |

Remember that TTLs govern the transition away from a record, not toward it: replacing a record with an 86400-second TTL means stale answers can persist for up to a day. This is the honest explanation behind most "slow propagation," and your flow should read the old record's TTL and tell the user the real number, as described in [the connect flow UX](02-the-connect-flow-ux.md).

## About CustomDomain

This guide is maintained by the team behind [CustomDomain](https://customdomain.ai), which catalogs 63 DNS and registrar providers. On the 25 with an automatic path (17 by scoped API token, 6 by one-click OAuth, 2 by Domain Connect) it writes and verifies the pointing records for you; on the other 38 it generates exact guided instructions and verifies them automatically as they appear. It chooses the right apex strategy per provider, and issues and renews TLS certificates the moment a connection verifies, typically taking a domain live in about 30 seconds via provider authorization. Integrate through the [connect widget](https://customdomain.ai/connect-domain-widget) or the [REST API](https://customdomain.ai/custom-domain-api), with working code in [integrating the connect flow](05-integrating-the-connect-flow.md) and full details in the [docs](https://docs.customdomain.ai/docs). Pricing starts at $0, with the free tier capped at 10 domain connections a year: [app.customdomain.ai/signup](https://app.customdomain.ai/signup).
