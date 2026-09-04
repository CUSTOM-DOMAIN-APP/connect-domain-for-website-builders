# Custom Domains for Website Builders

Connect custom domains to your website builder: a field guide to one-click domain connection.

**Status:** Published · 6 guides · public

[![License](https://img.shields.io/badge/license-MIT-1c1917?style=flat)](LICENSE)

|  |  |
|---|---|
| **What it is** | A field guide to domain connection for website builder platforms |
| **Who it's for** | Engineers and product teams behind site builders and no-code platforms |
| **Live at** | [customdomain.ai/for/site-builders](https://customdomain.ai/for/site-builders) · [docs.customdomain.ai](https://docs.customdomain.ai/docs) |
| **Stack** | Markdown only · 6 guides · no build step · MIT |
| **Status** | Published · every number re-verified against the live provider census on 2026-09-04 |

This repository explains how website builder platforms let their users publish on a domain
they own: the DNS records involved, how ownership verification and TLS issuance actually
work, and how to build a connect flow a non-technical user can finish without a support
ticket. It is maintained by the team behind [Custom Domain](https://customdomain.ai).

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
# 63
# { "api": 17, "dc": 2, "manual": 38, "oauth": 6 }
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
provider census counts how many of the 63 catalogued DNS and registrar providers sit on each:

| Rail | Providers | User effort | Typical time to live |
|---|---|---|---|
| One-click OAuth | 6 | Signs in at their own provider and approves | About 30 seconds |
| One-click Domain Connect | 2 | Approves the change at the provider itself | About 30 seconds |
| API token | 17 | Pastes one scoped token | A few minutes |
| Guided manual with automatic verification | 38 | Copies the exact records shown | Minutes, longer with stale caches |

Treat the browser callback as a UI signal and the webhook as the state change. The events a
site builder wires up are `connection.live`, `connection.failed`, `domain.record_missing`,
`domain.record_restored` and `connection.disconnected`. Signatures are HMAC-SHA256; the full
catalog and the state machine are in [Integrating the connect flow](docs/05-integrating-the-connect-flow.md).

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
  [OpenAPI 3.1 spec](https://api.customdomain.ai/v1/openapi.json), which currently describes
  67 paths. Check them before quoting them.
- Domain Connect is an open protocol maintained by its own association, not a Custom Domain
  product. Parts of the problem analysis draw on that project's
  [knowledge base](https://github.com/Domain-Connect/knowledge-base), published under CC0.

Sibling guides cover the same machinery for other verticals:
[agencies](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-agencies),
[AI agents](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-ai-agents),
[email platforms](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-email-platforms).
Corrections are welcome and the useful ones come with a source: see
[CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT. See [LICENSE](LICENSE).
