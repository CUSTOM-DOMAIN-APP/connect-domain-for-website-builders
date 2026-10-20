# Custom domains for website builders: the CustomDomain™ field guide

Custom domains for website builders and no-code platforms: the records, verification, TLS and connect-flow UX,
with Next.js on Vercel, Caddy and Cloudflare for SaaS compared.

**Status:** Published · 6 guides · live API numbers re-verified on 2026-10-20 · public

[![docs](https://img.shields.io/badge/docs-docs.customdomain.ai-1c1917?style=flat)](https://docs.customdomain.ai/docs)
[![license](https://img.shields.io/badge/license-MIT-1c1917?style=flat)](./LICENSE)

[Website](https://customdomain.ai) · [Docs](https://docs.customdomain.ai/docs) · [Console](https://app.customdomain.ai) · [For site builders](https://customdomain.ai/for/site-builders) · [Connect widget](https://customdomain.ai/connect-domain-widget)

|  |  |
|---|---|
| **What it is** | A field guide to domain connection for website builder platforms |
| **Who it's for** | Engineers and product teams behind site builders and no-code platforms |
| **Live at** | [customdomain.ai/for/site-builders](https://customdomain.ai/for/site-builders) · [docs.customdomain.ai](https://docs.customdomain.ai/docs) |
| **Stack** | Markdown only · 6 guides · no build step · MIT |
| **Status** | Published · provider census and API path count re-verified 2026-10-20 |

This repository explains how website builder platforms let their users publish on a domain
they own: the DNS records involved, how ownership verification and TLS issuance actually
work, and how to build a connect flow a non-technical user can finish without a support
ticket. It is maintained by the team behind [CustomDomain™](https://customdomain.ai).

## The problem

A site builder's whole promise is that non-technical people can ship a real website. That
promise holds right up until the domain step, because DNS was never designed for them. The
user who reaches this step is your best user: they finished their site, they own a domain,
and they want their own name in the address bar. Then you ask them to log into a registrar
they touched once three years ago and copy record values character for character, trailing
dot included.

Three parties are involved and none of them can talk to each other. You know exactly which
records are needed and cannot write them. The DNS provider controls the zone and has no idea
what your platform needs. The user owns the domain, understands neither side, and is the
messenger. So your help center grows a page per registrar, your support team learns to ask
for a screenshot of the DNS settings as a reflex, and the users who give up stay on a
subdomain and churn harder than the ones who went live on their own brand.

## Who it's for

Website builders, page builders and no-code platforms where every tenant starts on a
platform subdomain and the real goal is their own name. E-commerce platforms with a
storefront per merchant, creator tools with a domain per creator, and agencies publishing
client sites hit the same wall at the same screen. If your product ends in a publish button,
the domain step is the last thing standing between a user and a site they are proud to send
to someone.

## What it covers

- [How site builders offer custom domains](docs/01-how-site-builders-offer-custom-domains.md): the subdomain-to-custom-domain model, routing, and wildcard versus per-domain TLS
- [The connect flow UX](docs/02-the-connect-flow-ux.md): error states, propagation honesty, and what a good in-product connect experience looks like
- [DNS records for site builders](docs/03-dns-records-for-site-builders.md): the exact records, the apex problem, and CNAME flattening
- [Scaling TLS for tenant domains](docs/04-scaling-tls-for-tenant-domains.md): on-demand issuance, renewal at scale, and tenant isolation
- [Integrating the connect flow](docs/05-integrating-the-connect-flow.md): the widget embed, the webhook contract, signature verification, and the connection state machine
- [When a customer leaves](docs/06-when-a-customer-leaves.md): offboarding, certificate release, and why a forgotten CNAME becomes a subdomain takeover

The guides are plain Markdown. Clone the repo, or read them on GitHub. They are MIT
licensed, so adapting any of it for your own help center needs no permission.

## Quickstart

The repo has no build step. To check the provider numbers the guides quote, call the public
census endpoint, which is the source they trace to:

```bash
curl -s https://api.customdomain.ai/v1/providers/census \
  | jq '.count, (.providers | group_by(.mode) | map({(.[0].mode): length}) | add)'
# 178
# { "api": 55, "dc": 1, "manual": 116, "oauth": 6 }
```

To run the flow the guides describe, mint a short-lived widget token on your server and open
the widget in your publish flow. The browser never sees an API key or client secret:

```js
// Your publish flow. npm install customdomain-js
import { customdomain } from "customdomain-js";

const { application_id, token } = await fetch("/api/widget-token", { method: "POST" })
  .then((r) => r.json());

customdomain.open({
  applicationId: application_id,
  token,
  domain: "mybakery.example",
  endUserRef: tenant.id,        // your id for this tenant, echoed back on the connection
  onSuccess: ({ domain, jobId }) => markTenantLive(tenant.id, domain, jobId),
  onError: (err) => report(err.code, err.message),
});
```

## How it works

Strip away the provider interfaces and a domain connection is four steps: DNS records point
the hostname at your platform, something proves the user controls the zone, an ACME
certificate is issued for the domain, and resolvers pick the change up as their cached
answers expire. Order matters. Prove control, then issue the certificate, then route traffic,
or you will eventually serve one customer's site on another person's domain.

What differs between platforms is who writes the records. There are three rails, and the
provider census counts how many of the 178 catalogued DNS and registrar providers sit on each:

| Rail | Providers | User effort | Typical time to live |
|---|---|---|---|
| One-click OAuth | 6 | Signs in at their own provider and approves | About 30 seconds |
| One-click Domain Connect | 1 | Approves the change at the provider itself | About 30 seconds |
| API token | 55 | Pastes one scoped token | A few minutes |
| Guided manual with automatic verification | 116 | Copies the exact records shown | Minutes, longer with stale caches |

Treat the browser callback as a UI signal and the webhook as the state change. The events a
site builder wires up are `connection.live`, `connection.failed`, `domain.record_missing`,
`domain.record_restored` and `connection.disconnected`. Signatures are HMAC-SHA256; the full
catalog and the state machine are in [Integrating the connect flow](docs/05-integrating-the-connect-flow.md).

## Next.js on Vercel, Caddy and Cloudflare for SaaS

Most site builders asking how to offer custom domains already run on one of three stacks, and each one
handles routing and TLS for customer hostnames well. None of them writes the record into the customer's DNS
zone, which is the step this guide is mostly about. What each one does, in its own vendor's terms:

**Next.js on Vercel.** Add each customer's domain to your project with the Vercel SDK
(`projectsAddProjectDomain`) or the REST API, show the customer the record Vercel reports for it (an `A`
record, `76.76.21.21` or the value on the domain's card, at the root; a `CNAME` for a subdomain), and Vercel
issues a certificate per domain. A domain already in use on another Vercel account also needs a `TXT` record,
which you check with `projectsVerifyProjectDomain`. The app itself maps the request's hostname to a tenant;
Vercel's [Next.js multi-tenant example](https://github.com/vercel/platforms) does it for tenant subdomains in
`proxy.ts`, and a customer's own domain is one more hostname to look up. Vercel's guide: [Configuring custom domains](https://vercel.com/docs/platforms/multi-tenant-platforms/configuring-domains).

**Caddy with on-demand TLS.** On servers you run, Caddy can issue a certificate for a hostname the first time
a TLS handshake asks for it. Restrict it with an `ask` endpoint: Caddy calls it with `?domain=` and issues
only on a `2xx`, so only domains in your database get certificates.

```caddyfile
{
	on_demand_tls {
		ask http://localhost:5555/domain-allowed
	}
}

https:// {
	tls {
		on_demand
	}
	reverse_proxy localhost:3000
}
```

Customers point a `CNAME` (or `A` records, at the root) at your servers. You run the servers, the `ask`
endpoint, shared certificate storage if you run more than one instance, and your share of the CA's rate
limits. Caddy's docs: [On-Demand TLS](https://caddyserver.com/docs/automatic-https#on-demand-tls).

**Cloudflare for SaaS.** Put your platform zone on Cloudflare, set a fallback origin, and create a custom
hostname per customer (`POST /zones/{zone_id}/custom_hostnames`). The customer adds a `CNAME` to your CNAME
target, for example `customers.yourplatform.com`, and Cloudflare validates the hostname and issues and renews
its certificate at the edge. A root domain needs a provider that can flatten a `CNAME` at the apex, or
Cloudflare's apex proxying, a paid Enterprise add-on. On the Free, Pro and Business plans 100 custom hostnames
are included and each additional one is $0.10 (the Cloudflare for SaaS plans page, read 2026-10-20). Cloudflare's
guide: [Getting started](https://developers.cloudflare.com/cloudflare-for-platforms/cloudflare-for-saas/start/getting-started/).

**Where CustomDomain™ fits.** Every option above still ends with your customer copying a record into a DNS
panel. CustomDomain™ is the part that writes it: it detects the customer's DNS provider, writes the records
through the provider where it can (62 of the 178 providers in the census, by OAuth, Domain Connect or a scoped
API token), shows the exact records for the other 116, and verifies them in public DNS. It can point domains
at its own edge, which terminates TLS and proxies to your origin (the Growth plan and up), or at your own
infrastructure on every plan: a Vercel project, your Caddy servers, or your Cloudflare for SaaS CNAME target,
which keeps issuing the certificates. If your customers are happy to paste records and you already run one of
the stacks above, you may not need it.

## Repository layout

```
docs/
  01-how-site-builders-offer-custom-domains.md   the architectural model
  02-the-connect-flow-ux.md                      the product surface
  03-dns-records-for-site-builders.md            the records themselves
  04-scaling-tls-for-tenant-domains.md           certificates at scale
  05-integrating-the-connect-flow.md             widget, webhooks, state machine
  06-when-a-customer-leaves.md                   offboarding and dangling records
AGENTS.md                                        API shapes and status vocabulary for coding agents
CONTRIBUTING.md                                  house style, and how to file a correction
LICENSE                                          MIT
```

## Limitations

- This is documentation, not a library. Nothing here is installable, and the code blocks are
  illustrative extracts of the real SDK surface documented at [docs.customdomain.ai](https://docs.customdomain.ai/docs).
- Numbers age. Provider counts and API shapes are re-verified against the
  [census endpoint](https://api.customdomain.ai/v1/providers/census) and the served
  [OpenAPI 3.1 spec](https://api.customdomain.ai/v1/openapi.json), which described 129 paths
  on 2026-10-20. Check them before quoting them.
- Domain Connect is an open protocol maintained by its own association, not a CustomDomain™
  product. Parts of the problem analysis draw on that project's
  [knowledge base](https://github.com/Domain-Connect/knowledge-base), published under CC0.

Corrections are welcome and the useful ones come with a source: see
[CONTRIBUTING.md](CONTRIBUTING.md).

## Related

- [connect-domain-for-agencies](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-agencies): custom domains for agencies, the field guide
- [connect-domain-for-ai-agents](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-ai-agents): custom domains for AI agents, the field guide
- [connect-domain-for-email-platforms](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-email-platforms): custom sending domains for email platforms, the field guide
- [customdomain-sdk](https://github.com/CUSTOM-DOMAIN-APP/customdomain-sdk): the browser SDK `customdomain-js` and the React wrapper `@customdomain/react`
- [customdomain-mcp](https://github.com/CUSTOM-DOMAIN-APP/customdomain-mcp): the hosted MCP server for AI agents
- [docs](https://github.com/CUSTOM-DOMAIN-APP/docs): the CustomDomain™ documentation source, rendered at [docs.customdomain.ai](https://docs.customdomain.ai/docs)
- [awesome-custom-domains](https://github.com/CUSTOM-DOMAIN-APP/awesome-custom-domains): the curated list of the category, including the alternatives to this product

## Support

- **Docs:** [docs.customdomain.ai](https://docs.customdomain.ai/docs)
- **Questions and ideas:** [GitHub Discussions](https://github.com/CUSTOM-DOMAIN-APP/docs/discussions)
- **Bugs and corrections:** [open an issue](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-website-builders/issues) on this repository
- **Service status:** [status.customdomain.ai](https://status.customdomain.ai)
- **Account and billing:** connect@customdomain.ai
- **Security:** report privately to security@customdomain.ai, never in a public issue. Policy: [app.customdomain.ai/security](https://app.customdomain.ai/security)

## License

[MIT](./LICENSE). CustomDomain™ is a product of EverJust Company.
