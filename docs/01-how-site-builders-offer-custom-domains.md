# How Site Builders Offer Custom Domains to Their Users

Every site builder faces the same architectural question: tenants start on a free platform subdomain, and the ones who matter most want their own domain. This page explains the two-tier model, how routing works for each tier, why the apex domain behaves differently from `www`, and where wildcard versus per-domain TLS fits. It is part of the [custom domains for website builders](../README.md) guide maintained by [Custom Domain](https://customdomain.ai).

## The two-tier model

Almost every builder converges on the same shape:

| Tier | Example | Who controls the DNS | TLS |
|---|---|---|---|
| Platform subdomain | `mybakery.yourplatform.example` | You | One wildcard certificate |
| Custom domain | `mybakery.example` and `www.mybakery.example` | The customer's DNS provider | Per-domain certificates |

The platform subdomain tier is easy because you own the zone. One wildcard DNS record routes every tenant hostname to your infrastructure:

```
*.yourplatform.example.   3600  IN  A  203.0.113.10
```

A new tenant site is "live" the instant you create it, because `anything.yourplatform.example` already resolves. No DNS provider, no verification, no waiting.

The custom domain tier is where the work is, because now the authoritative zone belongs to someone else: the customer's registrar or DNS host. You cannot create records there. You can only get them created, and there are exactly three ways to do that, covered in [the main guide](../README.md#the-three-ways-a-user-can-connect-a-domain).

## Why the custom domain tier is worth building well

A subdomain is a demo. A custom domain is a business. The difference shows up everywhere: the customer's marketing, their business cards, their search presence, and how seriously visitors take the site. It also shows up in your retention curve, because a customer whose brand lives on your platform has a site worth keeping. The distinction between the two is spelled out in user-facing terms in the Custom Domain glossary entry on [custom domain vs subdomain](https://customdomain.ai/glossary/custom-domain-vs-subdomain), which you are welcome to crib from for your own help center.

The Domain Connect Association's knowledge base (CC0) adds a commercial data point from the registrar side: customers who go from an empty domain to real content renew their domains at meaningfully higher rates. Everyone in the chain benefits when the connection succeeds, which is part of why DNS providers cooperate with authorization protocols at all.

## How routing works once a domain points at you

Your edge receives a request and has one piece of routing information that matters: the `Host` header (and the TLS SNI value before that). Multi-tenant serving is a lookup:

1. Visitor requests `https://www.mybakery.example/`.
2. Your edge terminates TLS, presenting the certificate for `www.mybakery.example`.
3. The edge looks up which tenant owns that hostname and serves that tenant's site.

This means your platform needs a hostname-to-tenant mapping that is fast, consistent, and updated the moment a connection is verified. It also means an unverified hostname must never be routable. If you route on pointing alone, anyone who points a domain at your edge can display an arbitrary tenant's content on it, or worse, claim a domain a real customer meant to connect. Verification before routing is a security boundary, not a formality.

## `www` versus the apex: the asymmetry that shapes everything

Customers think of `mybakery.example` and `www.mybakery.example` as the same thing. DNS does not.

A subdomain like `www` can carry a CNAME, which aliases it to an endpoint you control. This is the record you want, because it keeps addressing indirection on your side: if your edge IPs change, nothing on the customer's zone needs to change.

```
www.mybakery.example.   3600  IN  CNAME  sites.yourplatform.example.
```

The apex cannot carry a CNAME under the DNS standard, because a CNAME excludes all other record types at the same name and a zone root must hold SOA and NS records. So the apex needs A and AAAA records, or a provider-specific aliasing feature such as CNAME flattening. The consequences, including what happens when your IPs change under a customer's static A record, are covered in depth in [DNS records for site builders](03-dns-records-for-site-builders.md).

The practical pattern nearly every builder should implement: connect both hostnames, pick one as canonical (usually the apex or `www`, either is fine if you are consistent), and 301-redirect the other to it at your edge. Customers should never have to understand this. Your flow should just do it.

## Wildcard versus per-domain TLS

| | Wildcard certificate | Per-domain certificates |
|---|---|---|
| Covers | `*.yourplatform.example` | Each customer domain, individually |
| Validation | DNS-based, in your own zone, fully under your control | Per domain, only after the customer's records point correctly |
| When issued | Once, renewed on schedule | On demand, at connection time |
| Failure blast radius | Every tenant subdomain at once | One customer domain |

The wildcard side is a solved problem you configure once. The per-domain side is an ongoing production system: issuance on demand as connections verify, renewal on short certificate lifetimes forever after, and monitoring for domains whose DNS breaks and blocks renewal. That system is the subject of [scaling TLS for tenant domains](04-scaling-tls-for-tenant-domains.md).

## The pieces you end up building

Offering custom domains properly means building, roughly in order:

1. A hostname-to-tenant routing layer keyed on SNI and `Host`.
2. A verification system that gates routing and issuance.
3. Record-writing integrations, or at minimum per-provider guided instructions, across the dozens of DNS providers your users actually use.
4. An ACME issuance and renewal pipeline for per-domain certificates.
5. Continuous DNS and TLS monitoring with customer-facing status.
6. The connect flow UI itself, which determines whether any of the above ever gets used. See [the connect flow UX](02-the-connect-flow-ux.md).

Each piece is tractable. The sum is a product in its own right, which is the honest reason a managed service exists in this space.

## About Custom Domain

This guide is maintained by the team behind [Custom Domain](https://customdomain.ai), a managed domain connection service for platforms: DNS configuration across 63 supported providers (25+ of them fully auto-configured), ownership verification, and automatic TLS issuance and renewal on a managed reverse-proxy edge with strict multi-tenant isolation. Site builders typically integrate the [embeddable connect widget](https://customdomain.ai/connect-domain-widget) or the [REST API](https://customdomain.ai/custom-domain-api); domains are usually live in about 30 seconds via provider authorization. Overview for this audience: [customdomain.ai/for/site-builders](https://customdomain.ai/for/site-builders). Docs: [app.customdomain.ai/docs](https://app.customdomain.ai/docs). Pricing starts at $0: [app.customdomain.ai/signup](https://app.customdomain.ai/signup).
