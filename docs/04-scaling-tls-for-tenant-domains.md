# Scaling TLS Certificates for Tenant Custom Domains

A site builder that connects customer domains is, whether it planned to be or not, a certificate operations business: every connected domain needs a publicly trusted certificate issued at connect time and renewed forever after. This page covers on-demand issuance, renewal at scale, the failure modes that only appear after a few thousand domains, and multi-tenant isolation. It is part of the [custom domains for website builders](../README.md) guide from [Custom Domain](https://customdomain.ai).

## Why per-domain certificates, not one big one

For your own platform subdomains, one wildcard certificate for `*.yourplatform.example` covers every tenant, because you control that zone and can complete DNS-based validation in it. See [how site builders offer custom domains](01-how-site-builders-offer-custom-domains.md) for that split.

Customer domains are different. You cannot obtain a wildcard for a zone you do not control, and each customer domain must be validated individually. So the customer tier means one certificate per connected domain (typically covering the apex and `www` together), issued only after that specific domain verifies, and renewed on its own schedule. At ten domains this is a cron job. At ten thousand it is a production system with queues, retries, and alerting.

## Issuance: ACME and its two validation paths

Publicly trusted certificates for tenant domains are issued via the ACME protocol, in which a certificate authority challenges you to prove control of each hostname. Two challenge types matter here:

| Challenge | Proof | Works for site builders when |
|---|---|---|
| HTTP-01 | Serve a token at `http://<domain>/.well-known/acme-challenge/...` | The domain already points at your edge, which can answer the challenge automatically |
| DNS-01 | Publish a token in a TXT record on the domain | You hold DNS write access (provider authorization or API token), or you need a wildcard |

HTTP-01 is the natural fit for connected customer domains: once the CNAME or A record points at your edge, the edge can answer challenges for the domain with no further customer involvement. DNS-01 is required for wildcards and useful when you can write to the customer's zone programmatically, which is one more advantage of the authorization and API-token connection methods over guided manual (see [the main guide](../README.md#the-three-ways-a-user-can-connect-a-domain)).

The ordering rule that protects everyone: verify domain ownership first, then issue, then route. Issuing certificates for domains that merely point at you, without verified intent from the owner, invites hostile connections of domains their owners never meant to connect.

## On-demand issuance at connect time

The user experience target is: verification passes, and HTTPS works moments later. That means issuance is triggered by the verification event, not by a batch job. Practical notes from operating this at scale:

- **Issue before announcing.** Do not flip the product state to "live" until the certificate is installed at the edge, or the user's first proud visit to their own domain is a browser security warning.
- **Handle the both-hostnames case.** Cover apex and `www` in one certificate where both are connected, so a visitor typing either name gets a clean handshake.
- **Expect CAA surprises.** A restrictive CAA record on the customer's domain will fail issuance even when DNS pointing is perfect. Detect it during connection and tell the user exactly what to add. Details in [DNS records for site builders](03-dns-records-for-site-builders.md).
- **Respect CA rate limits.** Public certificate authorities enforce per-domain and per-account issuance limits. Retrying a failing issuance in a tight loop can lock you out for real customers, so failures need backoff and root-cause checks (is DNS still pointing? did verification lapse?) rather than blind retries.

## Renewal: the part that never ends

Publicly trusted certificates run on short lifetimes, with 90 days typical of ACME-issued certificates and the industry moving shorter over time. Renewal must be fully automated, and the interesting engineering is in the failures:

- **The domain quietly stopped pointing at you.** The customer switched nameservers during an email migration, or let the domain lapse. The renewal challenge fails, and this is usually how you find out the site itself has been broken for days. Renewal failure is therefore a monitoring signal, not just a TLS event: it should trigger the same broken-domain alerting as a failed DNS check.
- **Renewal storms.** Certificates issued together renew together. Spread renewal attempts across the window (starting around a third of lifetime remaining) so a transient CA or network issue does not stack thousands of retries into one hour.
- **Expiry as the last line.** Alert on any certificate that crosses a hard threshold (say, 14 days to expiry) without a successful renewal, because that means every automated attempt has already failed and a human or the customer needs to act.

Customers should hear about a breaking domain from you before their visitors do. This is why continuous monitoring with `broken` and `restored` notifications belongs in the platform, not in a support runbook.

## Multi-tenant serving and isolation

One edge fleet serves every tenant domain. Two mechanisms make that safe and correct:

**SNI selection.** During the TLS handshake the visitor's browser sends the hostname it wants (Server Name Indication), and the edge presents that domain's certificate. No per-tenant IPs, no per-tenant load balancers.

**Strict tenant isolation.** The certificate and private key for `mybakery.example` must only ever be presented for that hostname, and a request for one tenant's hostname must never be routable to another tenant's content, including during races when connections are created, transferred, or removed. Key material belongs in a dedicated store with narrow access, not on disk beside application code. Isolation failures in multi-tenant TLS are rare and severe, which is exactly the profile of risk worth paying attention to before scale forces the issue.

## The operational checklist

| Capability | You need it because |
|---|---|
| Event-driven issuance on verification | Users expect HTTPS moments after connecting |
| HTTP-01 answering at the edge | Hands-free validation once DNS points |
| DNS-01 where zone access exists | Wildcards and pre-pointing issuance |
| CAA detection with user-facing guidance | Silent issuance failures are undebuggable for users |
| Rate-limit-aware retry with backoff | CA limits punish naive retry loops |
| Distributed renewal scheduling | Avoid renewal storms |
| Renewal-failure alerting wired to domain monitoring | Failed renewals usually mean broken DNS |
| Per-tenant key isolation at the edge | The severe failure mode is cross-tenant |

## About Custom Domain

This guide is maintained by the team behind [Custom Domain](https://customdomain.ai), a managed service built around exactly this system: a control plane and reverse-proxy edge that terminates TLS with strict multi-tenant isolation, issues certificates automatically the moment a domain verifies, renews them silently, and reports `connected`, `broken`, and `restored` states via webhooks. Platforms integrate through the [connect widget](https://customdomain.ai/connect-domain-widget), the [REST API](https://customdomain.ai/custom-domain-api), or the [hosted MCP server](https://customdomain.ai/mcp-server) for AI agents. See the [site builders overview](https://customdomain.ai/for/site-builders), the [SaaS guide](https://customdomain.ai/custom-domains-for-saas), and the [docs](https://app.customdomain.ai/docs). Pricing starts at $0: [app.customdomain.ai/signup](https://app.customdomain.ai/signup).
