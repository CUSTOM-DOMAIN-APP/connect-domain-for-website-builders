# When a Customer Leaves: Disconnecting Domains Without Leaving a Takeover

Connecting a domain gets all the design attention. Disconnecting one gets none, right up until a security researcher emails you about a hostname that still points at your edge and now serves someone else's content. This page covers the teardown path: what to remove, in what order, why a forgotten CNAME is a real vulnerability rather than a tidiness problem, and what to do about the records you cannot remove. It is part of the [custom domains for website builders](../README.md) guide from [CustomDomain](https://customdomain.ai).

## The asymmetry that creates the bug

When a customer connects, you gain a hostname you did not previously serve. When a customer leaves, you lose the right to serve it, but you do not automatically lose the pointing record: that record lives in a zone you do not control, and the customer has no reason to think about it. They cancelled a subscription. They are done.

So the default outcome of churn is a hostname in someone else's zone still aliased to your infrastructure, and a tenant slot on your side that no longer belongs to anyone. That combination is subdomain takeover.

## How subdomain takeover works

The mechanism is short enough to state completely.

1. `shop.northwind.example` has a CNAME to `sites.yourplatform.example`, left over from a customer who left.
2. Your platform no longer maps that hostname to any tenant, so the routing lookup misses.
3. An attacker creates an account on your platform and connects `shop.northwind.example`.
4. If your connect flow accepts a hostname purely because it resolves to your edge, the attacker now controls what visitors see at a hostname on the victim's domain.

Step 4 is the one you control, and it is the reason [how site builders offer custom domains](01-how-site-builders-offer-custom-domains.md) insists that pointing is not proof. A hostname pointing at your edge tells you where DNS sends traffic. It tells you nothing about who asked for that record to exist. The proof has to come from a channel that required the zone owner to authenticate: a provider authorization, a scoped API token, or a challenge the platform generated and the zone owner published.

The damage from a takeover is worse than a defaced page, because the hostname carries the victim's domain. Cookies scoped to the parent domain may be readable. Any allowlist, CORS policy, or OAuth redirect entry that trusts the domain now trusts the attacker. Certificate issuance for the hostname succeeds for the attacker, because from the certificate authority's perspective, whoever controls the DNS controls the name.

## The teardown order

Order matters, because each step removes a different capability and reversing them opens a window.

1. **Stop routing the hostname.** Remove the hostname-to-tenant mapping first. From this moment the domain does not serve the departing customer's content, which is usually what they asked for.
2. **Make the hostname unclaimable until it is re-proved.** This is the step platforms skip. Removing the mapping is not enough on its own; the hostname must go into a state where a new connection for it requires fresh proof of control rather than a resolution check. A hold on recently released hostnames, expiring after enough time that the record has plausibly been removed, is a reasonable default.
3. **Release the certificate.** Stop renewing it and remove the key material from the edge. There is no reason to keep a private key for a hostname you no longer serve, and an unrenewed certificate is one more thing that alerts at expiry for no reason.
4. **Remove the records if you can.** If the connection was made through provider authorization or an API token, you still hold the write path, so the cleanest teardown reverts the records you created. This is the strongest argument for the automatic rails that has nothing to do with onboarding convenience.
5. **Tell the customer what is left.** If you could not remove the records, say exactly which ones remain and where. "Remove the CNAME on `shop.northwind.example` at your DNS provider" is a specific, completable instruction. "Please update your DNS" is not.

CustomDomain's `DELETE /v1/connections/{id}` follows this shape: for a managed connection the template is reverted through the stored grant before the connection is torn down, and `connection.disconnected` fires so your platform can drop its own mapping in the same beat. For a manual connection there is no grant to revert, so the records stay in the customer's zone and the notice in step 5 is the only remedy available. Be honest about which case you are in.

## Records you will not get back

Some teardowns cannot be complete, and pretending otherwise is how the hold in step 2 gets skipped.

- **Manual connections.** The customer typed the records; only the customer can remove them.
- **Revoked grants.** An OAuth authorization the customer revoked before leaving takes your write path with it.
- **Deleted accounts.** The customer closed their account at the DNS provider, or transferred the domain, and the record moved to a zone you never had access to.
- **A domain that expired.** The record is gone with the zone, but the domain can be re-registered by anyone, which is a different and worse variant of the same problem.

Design for the case where the record outlives the relationship, because it frequently will.

## What to check on your own platform

A short audit, worth running once and then on a schedule:

| Check | Why |
|---|---|
| Does connecting a hostname require proof of control, or only that it resolves to us? | This is the takeover boundary. |
| Can two tenants hold the same hostname at once, even transiently? | Races during transfer and re-connect are where isolation fails. |
| Do we hold a released hostname before letting a different account claim it? | Closes the window between churn and record removal. |
| Do we revert records when we hold the write path? | The only complete teardown available. |
| Do we stop renewing certificates for disconnected hostnames? | Stale key material and noisy expiry alerts. |
| Do we tell departing customers exactly which records to remove? | The only remedy on the manual rail. |
| Do we alert on hostnames that resolve to our edge but map to no tenant? | This is the takeover candidate list, and you can generate it. |

That last row is the highest-value one. The set of hostnames pointing at your edge is knowable from your own certificate and request logs; the set mapped to a live tenant is knowable from your database. The difference is your exposure, and most platforms have never computed it.

## Transfers are not deletions

A customer moving a domain between two accounts on your platform, which happens constantly with agencies and with businesses that change hands, looks superficially like a disconnect followed by a connect. Treat it as its own operation. During a transfer the record is correct and should stay correct, so the naive teardown path would break a working site and force the customer through the connect flow again for no reason. The safer shape is to move the mapping atomically and keep the certificate, having verified that the requesting account has a claim on the hostname. The [agencies field guide](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-agencies) covers the client-handover case in more depth.

## About CustomDomain

This guide is maintained by the team behind [CustomDomain](https://customdomain.ai), a managed domain connection service for platforms: DNS configuration across 63 catalogued providers (25 of them with an automatic path, 38 on a guided flow with automatic verification), certificate issuance and renewal, and a reverse-proxy edge with strict multi-tenant isolation. Disconnects run through `DELETE /v1/connections/{id}`, which reverts the applied template through the stored grant where one exists and emits `connection.disconnected`. See [Connections](https://docs.customdomain.ai/docs/concepts/connections) and [Managed connections](https://docs.customdomain.ai/docs/connect-flow/managed-connections) in the [docs](https://docs.customdomain.ai/docs). Pricing starts at $0: [app.customdomain.ai/signup](https://app.customdomain.ai/signup).
