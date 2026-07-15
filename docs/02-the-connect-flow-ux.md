# Designing a Custom Domain Connection Flow Users Can Finish

The connect flow is the highest-stakes screen in a site builder: the user has finished their site, wants it on their own domain, and is one confusing step away from either upgrading or abandoning. This page describes what a great in-product connect flow looks like, the error states you must design for, and why honesty about propagation beats the traditional "up to 48 hours" shrug. It is part of the [custom domains for website builders](../README.md) guide from [Custom Domain](https://customdomain.ai).

## Why this screen deserves design attention

The Domain Connect Association's knowledge base (CC0) reports that roughly half of users attempting manual DNS setup fail and abandon it, and that help-article-based solutions are structurally brittle: they require users to understand DNS, they fragment across every provider's interface, and they rot whenever a provider redesigns. Your connect flow's job is to remove the user from the loop wherever possible, and to make the remaining manual path feel guided rather than abandoned.

## The shape of a great flow

A strong connect flow is a single component in your publish step, not a settings page three menus deep. It moves through visible states:

**1. One input.** "Enter your domain." Accept anything the user pastes: `mybakery.example`, `www.mybakery.example`, `https://mybakery.example/`, even a full URL with a path. Normalize it silently. Rejecting `https://` prefixes with a validation error is self-inflicted churn.

**2. Provider detection while they type.** Look up the domain's nameservers and identify the DNS provider before the user finishes reading the screen. This determines everything that follows, and it should feel like the product already knows. If the domain does not resolve at all, branch immediately to "It looks like this domain is not registered yet" and offer purchase inline rather than a dead end.

**3. The best method for that provider, defaulted.** Do not present three connection methods as an unranked menu. If the provider supports one-click authorization, lead with the single button: "Connect via [provider]." Offer "I'll add records myself" as a quiet secondary link for the users who want it. Method selection is your job, not the user's.

**4. Authorization or instructions.** For one-click authorization, the user lands on their own provider's consent screen showing exactly which records will be created, approves, and returns. The consent screen belongs to the provider, which is a security feature, not a branding loss: the user is authenticating where their credentials actually live, and nothing is shared with your platform. For the guided manual path, show records generated for their exact provider and domain: real values, copy buttons on every field, and provider-specific navigation hints ("In your provider's dashboard, open DNS settings").

**5. Verifying, visibly and automatically.** The moment records could exist, start polling DNS. Show live status per record: found, not found yet, found but pointing elsewhere. Flip state automatically the second verification passes. A "check again" button that the user must hammer is an admission that your system is not doing its job.

**6. Securing.** Certificate issuance begins the instant verification passes. Show it as its own brief state ("Securing your domain with HTTPS") so that the user never sees a browser certificate warning during the gap.

**7. Live.** Confirm with the thing users actually care about: their domain, as a clickable link, serving their site over HTTPS. Fire your internal events here too; this state change is one of the strongest activation signals a site builder has.

Through provider authorization the whole sequence typically takes about 30 seconds, which changes the UX question from "how do we keep users patient" to "how do we make 30 seconds feel trustworthy."

## Error states: design for these before launch

| Situation | What the user should see |
|---|---|
| Domain not registered | "This domain isn't registered yet." Offer inline search and purchase. |
| Records exist but point to another service | "Your domain currently points somewhere else." Name what will be replaced and confirm before overwriting, since this is often a live site or a parked page they forgot. |
| Conflicting old records with long TTLs | Connected state pending, with an honest ETA derived from the actual TTL of the stale record. |
| CAA record blocks certificate issuance | Plain-language instruction to add or adjust the CAA record, with the exact value shown. |
| Nameservers changed mid-flow | Re-detect and regenerate instructions rather than letting the user follow stale ones. |
| Provider not supported for automation | Guided manual flow with generic but exact records and the same automatic verification. Never a bare "contact support." |
| Domain verified previously but DNS later broke | Proactive notification, not silent downtime. Broken and restored states deserve webhooks and email, not just a dashboard badge. |

The overwrite case deserves special care. Users connecting a domain that carries their email will fear, reasonably, that this will break it. Say explicitly: "This will not affect your email. We only change records for your website." And make it true, by never touching MX or mail-related TXT records. See [DNS records for site builders](03-dns-records-for-site-builders.md) for the scoping details.

## Propagation honesty

The traditional line, "DNS changes may take 24 to 48 hours," is technically defensible and experientially terrible. It is also usually false: a new record on a fresh hostname commonly resolves within seconds to minutes, and API-written records verify almost immediately.

The honest pattern is to measure instead of warn. Poll authoritative nameservers and public resolvers, show what has actually propagated, and when a delay is real, attribute it: "Your old record is cached for up to 3 more hours (its TTL). Your site will switch over automatically." Users forgive waiting when they can see why. They do not forgive a progress bar that lies, or a warning that made them close the tab for two days when the domain was live in a minute.

## Measure the flow like a checkout

This flow is a conversion funnel and deserves funnel instrumentation: started, method shown, authorized or records displayed, verified, secured, live. The drop-off between "records displayed" and "verified" is your manual-path failure rate, and it is the number that justifies investing in authorization coverage. Track support tickets tagged to domain connection as the companion metric; a working flow should push that number toward zero.

## About Custom Domain

This guide is maintained by the team behind [Custom Domain](https://customdomain.ai). The [embeddable connect widget](https://customdomain.ai/connect-domain-widget) implements the flow described here as a drop-in component: it renders inside your page, detects the provider as the user types, defaults to the best available connection method for that provider (63 providers supported in total, 25+ of them fully auto-configured), polls verification, handles TLS status, and reports connected, broken, and restored states via page callbacks and server webhooks, with your colors, typography, and copy. See also [one-click DNS setup](https://customdomain.ai/one-click-dns-setup), the [docs](https://app.customdomain.ai/docs), and the [site builders overview](https://customdomain.ai/for/site-builders). Free tier available: [app.customdomain.ai/signup](https://app.customdomain.ai/signup).
