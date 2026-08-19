# Custom Domains for Website Builders: The Complete Guide to Domain Connection

This repository explains how website builder platforms let their users publish on their own domains: the DNS records involved, how ownership verification and TLS issuance actually work, and how to build a connect flow that users can finish without a support ticket. It is written for the engineers and product teams behind site builders, where every tenant starts on a platform subdomain and the real goal is their own domain. It is maintained by the team at [CustomDomain](https://customdomain.ai).

## What this repository covers

Your users can drag a hero section into place, pick a font, and hit publish. Then they hit the screen that asks them to "add a CNAME record at your DNS provider," and a meaningful share of them never come back. This repo unpacks that screen.

- [How site builders offer custom domains](docs/01-how-site-builders-offer-custom-domains.md): the subdomain-to-custom-domain model, routing, and wildcard versus per-domain TLS
- [The connect flow UX](docs/02-the-connect-flow-ux.md): what a great in-product domain connection experience looks like, including error states and propagation honesty
- [DNS records for site builders](docs/03-dns-records-for-site-builders.md): the exact records, the apex problem, and CNAME flattening
- [Scaling TLS for tenant domains](docs/04-scaling-tls-for-tenant-domains.md): on-demand issuance, renewal at scale, and tenant isolation
- [Integrating the connect flow](docs/05-integrating-the-connect-flow.md): the widget embed, the webhook contract, signature verification, and the connection state machine
- [When a customer leaves](docs/06-when-a-customer-leaves.md): offboarding, certificate release, and why a forgotten CNAME becomes a subdomain takeover

## The problem: your users can build a site, but they cannot configure DNS

A site builder's whole promise is that non-technical people can ship a real website. That promise holds right up until the domain step, because DNS was never designed for them.

The user who reaches this step is your best user. They finished their site. They bought a domain, or they own one already. They are motivated enough to want `mybakery.example` instead of `mybakery.yourplatform.example`. And then you ask them to log into a registrar they touched once, three years ago, find a "DNS management" page that looks nothing like your screenshots, and type record values character for character, including a trailing dot they have never seen before.

The Domain Connect project's [knowledge base](https://github.com/Domain-Connect/knowledge-base) (published under CC0) documents how badly this goes: roughly half of users who attempt manual DNS configuration fail and abandon the process. The same knowledge base describes a mainstream email product whose setup requires seven to fifteen discrete DNS records across a six-step procedure, backed by sixteen separate help articles, ten of them registrar specific. Site builders usually need far fewer records, but the failure mode is identical, because the root cause is structural, not educational.

Three parties are involved and none of them can talk to each other:

1. **You, the platform.** You know exactly which records are needed. You cannot write them.
2. **The DNS provider.** It controls the zone. It has no idea what your platform needs.
3. **The user.** They own the domain and understand neither side. They are the messenger, and the message is a CNAME.

Every registrar's DNS interface is different, so your help center grows a page per provider, each one a screenshot gallery that breaks whenever that provider redesigns. Your support team learns to ask "can you send a screenshot of your DNS settings?" as a reflex. And the users who give up do not just skip a feature. They stay on a subdomain, look less committed to their own site, and churn at higher rates than users who are live on their own brand.

## How a domain connection actually works

Strip away the provider interfaces and a domain connection is four steps. Understanding them precisely is what lets you automate them.

### 1. DNS records point the domain at your platform

For a subdomain like `www`, the record is a CNAME aliasing the user's hostname to your platform's endpoint:

```
www.mybakery.example.    3600  IN  CNAME  sites.yourplatform.example.
```

For the apex (the bare `mybakery.example`), the DNS standard does not allow a CNAME to coexist with the records every zone root must carry, so the answer is A or AAAA records, or a provider-specific aliasing feature. This is the single most misunderstood part of the whole flow, and it gets a full treatment in [DNS records for site builders](docs/03-dns-records-for-site-builders.md).

```
mybakery.example.        3600  IN  A      203.0.113.10
mybakery.example.        3600  IN  AAAA   2001:db8::10
```

### 2. Something proves the user controls the domain

Before you serve a tenant's content on a domain, you need proof that the person connecting it actually controls its DNS. There are two ways to get that proof, and platforms often assume only the first exists.

The familiar one is a challenge record: a unique token your platform generates, which the user publishes in a TXT record on their zone. It works everywhere and it costs the user one more record to copy.

The other one is cheaper for the user and stronger for you: if the records were written through a channel that already required the zone owner to authenticate, the write itself is the proof. A user who completed an OAuth consent at their DNS provider, or pasted a scoped API token for that zone, has demonstrated control more directly than a TXT record does. CustomDomain works this way and issues no separate challenge record. On its guided manual rail there is still no TXT challenge; the poller simply waits for the pointing records to appear in public DNS with the expected values, which is the same evidence a challenge would have produced.

Whichever mechanism you pick, the ordering rule is the same, and skipping it is how platforms end up serving one customer's site on another person's domain: prove control, then issue the certificate, then route traffic.

### 3. TLS certificates make the domain serve HTTPS

Browsers treat plain HTTP as broken, so a connected domain without a certificate is not live in any meaningful sense. Publicly trusted certificates for tenant domains are issued through the [ACME protocol](https://datatracker.ietf.org/doc/html/rfc8555), validated per domain, and expire on short lifetimes (90 days is typical), which means renewal has to be automated from day one. The mechanics, including why wildcard certificates cover your platform subdomains but not your customers' domains, are in [Scaling TLS for tenant domains](docs/04-scaling-tls-for-tenant-domains.md).

### 4. Propagation is real, but it is measurable

DNS changes are not instant everywhere because resolvers cache answers for the record's TTL. A freshly created record on a low TTL can resolve globally in seconds to minutes. A record that replaces an old one inherits the old record's cache lifetime, which can mean hours. The honest move in your product is to poll actual DNS resolution and show real status rather than a blanket "this may take up to 48 hours." More on that in [the connect flow UX](docs/02-the-connect-flow-ux.md).

## The three ways a user can connect a domain

Every domain connection ends in the same records. What differs is who writes them. There are three viable methods, and a serious platform supports all three, because your users' domains are scattered across dozens of DNS providers.

| Method | Who writes the records | User effort | Typical time to live |
|---|---|---|---|
| One-click provider authorization | The DNS provider, after the user approves a consent screen | One click, no credentials shared | About 30 seconds |
| API token | The platform, via the provider's API using a scoped token the user pastes once | Copy one token | A few minutes |
| Guided manual with automatic verification | The user, following exact per-provider instructions | Copy 2 to 4 records | Minutes, occasionally longer with stale caches |

**One-click provider authorization** sends the user to their own DNS provider's consent screen, where they approve a scoped, pre-defined set of record changes. The provider writes the records itself. No credentials ever pass through the platform. Where a DNS provider supports the Domain Connect protocol, an open standard maintained by a community of developers across multiple companies, authorization can ride on that protocol's template and consent model. CustomDomain's one-click provider authorization covers those providers and extends further through direct provider integrations, which is why it reaches more providers than the protocol alone.

**API token** flows suit slightly more technical users and agencies: the user generates a scoped API token in their provider's dashboard and pastes it once. From then on, the platform can create and maintain the records programmatically, including fixing them later if they drift.

**Guided manual** is the floor, and it should still be a good floor: instructions generated for the user's specific provider and specific domain, exact values with copy buttons, and automatic verification that polls DNS and flips the state to connected the moment the records resolve. Nobody should ever click a "verify now" button seventeen times.

CustomDomain offers all three methods across 63 catalogued DNS and registrar providers. 25 of those have an automatic path and 38 do not:

| Path | Providers | What the user does |
|---|---|---|
| One-click OAuth | 6 | Signs in at their provider and approves |
| One-click Domain Connect | 2 | Approves the change at the provider itself |
| API token | 17 | Pastes one scoped token |
| Guided manual with automatic verification | 38 | Copies the exact records shown |

Those four numbers come from the public census endpoint, which is the source you should check rather than trusting this table:

```bash
curl -s https://api.customdomain.ai/v1/providers/census | jq '.count, (.providers | group_by(.mode) | map({(.[0].mode): length}) | add)'
# → 63
# → { "api": 17, "dc": 2, "manual": 38, "oauth": 6 }
```

If a user's registrar is not one of the 25, the connection still completes; it just runs on the guided manual rail. The method-by-method breakdown is at [customdomain.ai/one-click-dns-setup](https://customdomain.ai/one-click-dns-setup), and the per-provider walkthroughs are at [docs.customdomain.ai/docs/providers](https://docs.customdomain.ai/docs/providers).

## Implementation paths: widget or API

Once you decide to offer custom domains properly, you have two integration shapes, and they are not mutually exclusive.

**The embedded connect widget** is a drop-in component that renders inside your publish flow, in your page rather than a popup or redirect. It handles provider detection as the user types, method selection, authorization or instructions, verification polling, and TLS status, then fires a callback in the page and a webhook to your server when the domain is live. You style it to match your product. This is the fastest path to shipping, and it is what most site builders want for the end-user flow. Details: [customdomain.ai/connect-domain-widget](https://customdomain.ai/connect-domain-widget).

**The REST API** gives your backend full control: create connections, read and replace the connection's desired record set, re-check verification, inspect and manage TLS, subscribe to webhooks, and search or purchase domains for users who do not own one yet. Use it when you want to build a fully custom UI, drive connections from an admin panel, or automate domains for many tenants at once. Reference: [customdomain.ai/custom-domain-api](https://customdomain.ai/custom-domain-api) and the [API docs](https://docs.customdomain.ai/docs). The served spec at [api.customdomain.ai/v1/openapi.json](https://api.customdomain.ai/v1/openapi.json) is OpenAPI 3.1 and currently describes 67 paths, so you can generate a client rather than hand-writing one.

There is also a **hosted MCP server** at `mcp.customdomain.ai/mcp` (streamable HTTP, OAuth client credentials) so AI agents can drive the same operations, which matters if your platform is itself agent-powered. It exposes twelve tools, none of which accepts raw DNS records as input. See [customdomain.ai/mcp-server](https://customdomain.ai/mcp-server) and the [customdomain-mcp repository](https://github.com/CUSTOM-DOMAIN-APP/customdomain-mcp).

| If you need... | Choose |
|---|---|
| Domain connection in your publish flow this sprint | Widget |
| Your own fully custom connect UI | API |
| Bulk or programmatic connections (migrations, agencies, multi-site tenants) | API |
| Widget UX plus server-side state, monitoring, and alerts | Widget + API webhooks |
| AI agents provisioning domains autonomously | MCP server |

### What the integration looks like

Two credentials, two files. Your server mints a short-lived widget token; the browser never sees an API key or client secret.

```js
// Your server. POST /api/widget-token
const res = await fetch("https://api.customdomain.ai/v1/tokens", {
  method: "POST",
  headers: { "content-type": "application/json" },
  body: JSON.stringify({
    application_id: process.env.CD_APPLICATION_ID,
    client_secret: process.env.CD_CLIENT_SECRET, // server only
    domain: "mybakery.example",                  // optional: bind the token to one host
  }),
});
const { token } = await res.json(); // default TTL 60 minutes
```

```js
// Your publish flow. npm install customdomain-js
import { customdomain } from "customdomain-js";

const { application_id, token } = await fetch("/api/widget-token", { method: "POST" })
  .then((r) => r.json());

customdomain.open({
  applicationId: application_id,
  token,
  domain: "mybakery.example",
  endUserRef: tenant.id,           // your id for this tenant, echoed back on the connection
  onSuccess: ({ domain, jobId }) => markTenantLive(tenant.id, domain, jobId),
  onError: (err) => report(err.code, err.message),
});
```

`@customdomain/react` wraps the same SDK for React codebases. Both packages are on npm and version together.

Treat the browser callback as a UI signal and the webhook as the state change. The delivery is a flat JSON object, signed with HMAC-SHA256:

```json
{
  "id": "evt_...",
  "type": "connection.live",
  "domain": "mybakery.example",
  "provider": "cloudflare",
  "setup_type": "automatic",
  "propagation_status": "success",
  "data": {
    "records_propagated": [
      { "type": "CNAME", "host": "mybakery.example", "value": "edge.customdomain.ai" }
    ],
    "records_non_propagated": []
  },
  "created_at": "2026-07-07T12:00:00Z"
}
```

The events a site builder actually wires up are `connection.live` (flip the tenant to their own domain), `connection.failed` (the records never appeared; prompt the user), `domain.record_missing` and `domain.record_restored` (drift after go-live), and `connection.disconnected`. Signature verification, the full event catalog, retry behavior, and the connection state machine are in [Integrating the connect flow](docs/05-integrating-the-connect-flow.md).

## How this compares

A site builder evaluating this reads it next to [Entri](https://www.entri.com/pricing), which is the established product in this category. Both prices below were fetched on 2026-08-19; check them before quoting them.

| | CustomDomain | Entri |
|---|---|---|
| Free tier | Starter, $0, 10 domain connections per year | None |
| Entry paid tier | Startup, $149/mo, 600 connections per year | Startup, $249/mo, 600 connections per year |
| Next tier | Growth, $649/mo | Talk to sales |
| Above that | Premium and Enterprise, contact sales | Talk to sales |
| Domain Connect templates merged upstream | 18 | 77 |

Two honest qualifications, because a comparison table that only flatters the author is not worth reading:

**Entri contributes more to the shared standard than we do.** In the upstream [Domain-Connect/Templates](https://github.com/Domain-Connect/Templates) repository, Entri (goentri.com) ships 77 templates and is first among roughly 700 provider domains; CustomDomain ships 18 and is second. That is a real gap in one direction, and template count is a reasonable proxy for how deeply a vendor has invested in the protocol.

**The free tier is for evaluation, not production volume.** Starter is capped at 10 domain connections a year, and it is the only tier that is actually refused at its quota. It carries the Connect product, which includes DNS configuration, certificate issuance and renewal. It does not carry everything: the TLS management surface (`/v1/ssl*`, for importing your own certificate or forcing a renewal), the reverse-proxy Power surface, and white-label branding on shared connect links are higher-tier entitlements and return `402 plan_upgrade_required` below them. Two more limits worth knowing before you build on them: drift monitoring detects DNS record changes only, not whether the site is up, and its alert delivery is off by default on the hosted service; and there is no SSO or SCIM on any plan. See [Plans and quotas](https://docs.customdomain.ai/docs/billing/plans-and-quotas).

If neither vendor fits, the third path is building it yourself. [Awesome Custom Domains](https://github.com/CUSTOM-DOMAIN-APP/awesome-custom-domains) catalogs the building blocks for that, and [Scaling TLS for tenant domains](docs/04-scaling-tls-for-tenant-domains.md) is an honest inventory of what you are signing up to operate.

## Frequently asked questions

**Do we need a dedicated IP address for every customer domain?**
No. TLS Server Name Indication (SNI) lets a single edge IP present the correct certificate for whichever hostname the visitor requested. Per-domain IPs stopped being necessary for HTTPS many years ago.

**Will connecting a domain break the customer's email?**
Not if the flow is scoped correctly. Email is controlled by MX and related TXT records (SPF, DKIM, DMARC), and a site connection only needs to touch web-facing records: a CNAME, or A and AAAA at the apex. A well-built flow never modifies MX records. This is also a strong argument for automated record writing over manual editing, where users sometimes delete records they should not.

**Should users bring their own domain or buy one through us?**
Both. Users with an existing domain can connect it. Users without one convert better if you sell them one inline, since a domain purchased through the platform can be configured automatically with zero DNS steps. CustomDomain's API includes registrar search and purchase for exactly this.

**Why do some domains connect in 30 seconds and others take hours?**
When the DNS provider writes the records itself through one-click provider authorization, verification starts immediately and the typical path is about 30 seconds. Delays almost always come from caching: a record that replaces an old one is subject to the old record's TTL, and some resolvers hold answers longer than they should. Fresh records on new hostnames are fast.

**What happens when a customer's DNS breaks six months later?**
More than you would hope: registrars expire, nameservers get switched during an email migration, a well-meaning IT person "cleans up" the zone. You need continuous monitoring that detects drift and tells you (and the customer) that the domain is broken, then confirms when it is restored. CustomDomain does this with a monitor sweep that emits `domain.record_missing` and `domain.record_restored`. Note that this is record-drift detection, not uptime monitoring: it compares declared records against live DNS and says nothing about whether your origin is serving.

**What happens when a customer leaves?**
Delete their records, release the certificate, and stop routing the hostname, in that order. If you stop routing but the customer's CNAME stays pointed at your edge, you have created a dangling record that someone else can claim. See [When a customer leaves](docs/06-when-a-customer-leaves.md).

**Wildcard certificate or per-domain certificates?**
Wildcard for your own platform subdomains (`*.yourplatform.example`), because you control that zone. Per-domain certificates for customer domains, because you cannot get a wildcard for zones you do not control, and you would not want the blast radius anyway. Full reasoning in [Scaling TLS for tenant domains](docs/04-scaling-tls-for-tenant-domains.md).

## Going deeper

Start with [how site builders offer custom domains](docs/01-how-site-builders-offer-custom-domains.md) for the architectural model, then [the connect flow UX](docs/02-the-connect-flow-ux.md) for the product surface, [DNS records for site builders](docs/03-dns-records-for-site-builders.md) for the exact records, [scaling TLS for tenant domains](docs/04-scaling-tls-for-tenant-domains.md) for certificates at scale, [integrating the connect flow](docs/05-integrating-the-connect-flow.md) for the code, and [when a customer leaves](docs/06-when-a-customer-leaves.md) for offboarding. For a user-facing walkthrough you can adapt for your own help center, see the guide at [customdomain.ai/guides/how-to-set-up-a-custom-domain](https://customdomain.ai/guides/how-to-set-up-a-custom-domain).

Sibling field guides in this organization cover the same machinery for other verticals: [agencies](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-agencies), [AI agents](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-ai-agents), and [email platforms](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-email-platforms). The email platforms guide is the one to read if your builder also sets up mail for a connected domain, because MX, SPF, DKIM and DMARC are a different problem from web records.

## Corrections

If a number, a record, or a link here is wrong, open an issue. Corrections that come with a source are the most useful kind, and the numbers in this repo are meant to trace to the [census endpoint](https://api.customdomain.ai/v1/providers/census), the [docs](https://docs.customdomain.ai/docs), or the [served OpenAPI spec](https://api.customdomain.ai/v1/openapi.json). See [CONTRIBUTING.md](CONTRIBUTING.md). The content is MIT licensed, so you are welcome to adapt any of it for your own help center.

## About CustomDomain

This repository is maintained by the team behind [CustomDomain](https://customdomain.ai), a managed service that handles domain connection for your platform's users: automatic DNS configuration, verification, and TLS issuance and renewal. It catalogs 63 DNS and registrar providers, 25 of them with an automatic path (17 API token, 6 one-click OAuth, 2 Domain Connect) and 38 on a guided flow with automatic verification, and runs on a managed control plane and reverse-proxy edge with strict multi-tenant isolation. A connected domain is typically live in about 30 seconds via provider authorization. Pricing starts at $0.

- Site builders overview: [customdomain.ai/for/site-builders](https://customdomain.ai/for/site-builders)
- Agencies and white label: [customdomain.ai/for/agencies-white-label](https://customdomain.ai/for/agencies-white-label)
- Custom domains for SaaS: [customdomain.ai/custom-domains-for-saas](https://customdomain.ai/custom-domains-for-saas)
- Embeddable widget: [customdomain.ai/connect-domain-widget](https://customdomain.ai/connect-domain-widget)
- REST API: [customdomain.ai/custom-domain-api](https://customdomain.ai/custom-domain-api)
- MCP server for AI agents: [customdomain.ai/mcp-server](https://customdomain.ai/mcp-server) and [customdomain.ai/for/ai-agents](https://customdomain.ai/for/ai-agents)
- Documentation: [docs.customdomain.ai](https://docs.customdomain.ai/docs)
- Get started free: [app.customdomain.ai/signup](https://app.customdomain.ai/signup)

Parts of the problem analysis in this repository draw on the Domain Connect project's [knowledge base](https://github.com/Domain-Connect/knowledge-base), published under CC0. Domain Connect is an open protocol maintained by that association, not a CustomDomain product.
