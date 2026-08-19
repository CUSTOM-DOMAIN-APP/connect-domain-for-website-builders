# Integrating the Connect Flow: Widget, Webhooks, and State

The first four pages of this guide describe the machinery. This one is the integration itself: minting a token, mounting the widget in a publish flow, the connection state machine your database has to mirror, and the webhook contract that drives it. It is part of the [custom domains for website builders](../README.md) guide from [CustomDomain](https://customdomain.ai).

Everything below is the CustomDomain contract specifically. If you are building your own system, the shapes are still worth reading as a checklist of what a connect integration has to expose: a short-lived browser credential, a state machine with a terminal failure, signed server-side events, and a way to re-check on demand.

## The credential split

Two credentials, and only one of them may reach a browser.

| Credential | Lives | Can do |
|---|---|---|
| API key (`sk_live_...`) or client secret | Your server only | Everything: applications, keys, connections, billing |
| Widget JWT | The browser | The connect endpoints for one application, and one hostname if you bind it |

Your server exchanges the application's client secret for a widget JWT and hands only that to the page:

```js
// Your server. POST /api/widget-token
const res = await fetch("https://api.customdomain.ai/v1/tokens", {
  method: "POST",
  headers: { "content-type": "application/json" },
  body: JSON.stringify({
    application_id: process.env.CD_APPLICATION_ID,
    client_secret: process.env.CD_CLIENT_SECRET,
    domain: "mybakery.example", // optional: the token may then act on this host only
  }),
});
const { token, expires_in } = await res.json(); // expires_in is seconds, default 3600
```

`POST /v1/tokens` is not bearer-authenticated. It authenticates on the `client_secret` in the body, which is exactly why it must never run in a browser. Mint one per connect session. When it expires the widget shows a session-expired state and you mint another.

If your application was created through the console, its client secret is sealed at signup and never displayed, so there is nothing to put in the call above. Use the console's `POST /api/widget-token` with one of your API keys instead; it opens the sealed secret server-side and returns the same widget JWT.

## Mounting the widget

```bash
npm install customdomain-js
```

```js
import { customdomain } from "customdomain-js";

async function connectDomain(tenant, domain) {
  const { application_id, token } = await fetch("/api/widget-token", {
    method: "POST",
    body: JSON.stringify({ domain }),
  }).then((r) => r.json());

  const { close } = customdomain.open({
    applicationId: application_id,
    token,
    domain,                       // prefill; the user can still change it
    endUserRef: tenant.id,        // your id, stored on the connection for attribution
    applicationName: "Your Platform",
    whiteLabel: {
      colors: { primary: "#0ea5e9", background: "#ffffff", text: "#111827" },
      borderRadius: "10px",
    },
    onStepChange: (step) => track("domain_connect_step", { step }),
    onSuccess: ({ domain, jobId, setupType, alreadyConnected }) => {
      // UI signal. Do not treat this as your source of truth; see below.
      showLiveState(domain, { jobId, setupType, alreadyConnected });
    },
    onError: (err) => report(err.code, err.message),
  });
}
```

A browser build also attaches `window.customdomain`, and `@customdomain/react` wraps the same SDK for React codebases. Pass a `container` selector instead of taking the default modal if you want the flow rendered as a panel inside your own publish step. Theming is applied as CSS custom properties on the widget's shadow host, so it cannot leak into your page, and `tokens` lets you override any of the 85 raw design tokens if the quick keys are not enough. Full surface: [widget SDK reference](https://docs.customdomain.ai/docs/widget-sdk/reference).

Two things to get right here:

- **`endUserRef` is the field that makes your console usable later.** Without it every connection shows as a bare direct connection and you cannot answer "which tenant owns this domain" without a join you have to maintain yourself.
- **`onSuccess` is a UI event, not a state change.** It fires in a browser you do not control, on a page the user can close mid-flight. Flip your tenant's state on the webhook.

## The connection state machine

Mirror these four statuses and no others. Inventing a `verified` or `pending_ownership` state means writing code against a machine the product does not have.

```
pending  --[ a rail applies records: OAuth, one-click setup, API token ]-->  propagating
pending  --[ manual: records observed in public DNS ]---------------------->  propagating

propagating  --[ every desired record resolves ]-->  live
propagating  --[ window expires, records unseen ]->  failed
```

| Status | Meaning | What your product should show |
|---|---|---|
| `pending` | Created; records not written or not yet observed | The connect flow, resumable |
| `propagating` | Records written by a rail, being checked against public DNS | "Setting up," with real per-record status |
| `live` | Every desired record resolves to its intended value; the edge serves TLS | The domain as a clickable HTTPS link |
| `failed` | The records never appeared inside the window | The reason plus a way to retry |

There is no separate ownership challenge in this machine, and no TXT challenge record. Control of the zone is proven by the rail: an OAuth authorization, a one-click setup apply, or a scoped API token each require the customer to authenticate where the zone actually lives. On the manual rail the poller waits for the pointing records to show up with the expected values.

The two windows differ on purpose. An automatic rail wrote the records itself, so if they have not resolved in 24 hours something is wrong and the connection goes `failed` with `error_code: propagation_timeout`. A manual connection is waiting on a human, so it gets 72 hours before `setup_incomplete`. Both clear automatically if the records later appear.

`POST /v1/connections/{id}:recheck` resolves the record set in-request and returns a per-record verdict rather than whatever the background poller last saw. It promotes a resolvable connection straight to `live`, and re-enters a `failed` one into verification with its clock re-based. This is the call behind a "check again" button, and it is also the right call to make when a user tells support their domain is working but your dashboard disagrees.

## The webhook contract

Register an endpoint and store the secret; it is returned exactly once.

```bash
curl -X POST https://api.customdomain.ai/v1/webhooks \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url":"https://api.yourplatform.example/webhooks/domains",
       "events":["connection.live","connection.failed","domain.record_missing","domain.record_restored","connection.disconnected"]}'
# → { "id": "whk_...", "secret": "whsec_...", ... }
```

An empty `events` array subscribes to everything. The events a site builder actually acts on:

| Event | What it means for your product |
|---|---|
| `connection.created` | A tenant started a connect. Useful only for funnel analytics. |
| `connection.applied` | A rail wrote the records. `data.via` distinguishes OAuth from one-click setup. |
| `connection.live` | Serve this tenant on their own domain. This is the state change. |
| `connection.failed` | The records never appeared. Prompt the user; the connection is resumable. |
| `domain.record_missing` | Drift after go-live. The customer's DNS changed under you. |
| `domain.record_restored` | The drift resolved. |
| `connection.disconnected` | The connection was removed. Stop routing the hostname. |
| `secure_status` | A certificate's provisioning status changed. |

The delivery is a flat JSON object:

```json
{
  "id": "evt_...",
  "type": "connection.live",
  "domain": "mybakery.example",
  "subdomain": "",
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

Fields that do not apply to an event are omitted. `updated_objects` rides only the events that actually wrote something (`connection.applied`, `connection.records.updated`), so an event that merely observed a connection never claims to have changed it.

### Verifying the signature

Every delivery carries `X-JE-Timestamp` (unix seconds) and `X-JE-Signature: sha256=<hex>`. The signature is an HMAC-SHA256, keyed by your endpoint secret, over the exact string `${timestamp}.${rawBody}`.

```js
const crypto = require("crypto");

function verify(rawBody, tsHeader, sigHeader, secret, toleranceSec = 300) {
  const ts = parseInt(tsHeader, 10);
  if (!Number.isFinite(ts)) return false;
  if (Math.abs(Date.now() / 1000 - ts) > toleranceSec) return false; // reject replays
  const expected =
    "sha256=" +
    crypto.createHmac("sha256", secret).update(`${ts}.`).update(rawBody).digest("hex");
  const a = Buffer.from(sigHeader || "");
  const b = Buffer.from(expected);
  return a.length === b.length && crypto.timingSafeEqual(a, b);
}
```

Three rules that people get wrong: compute the HMAC over the bytes you received rather than a re-serialized copy of the parsed JSON, prepend the timestamp and a dot before hashing, and compare in constant time. The timestamp is inside the signature specifically so you can reject a replayed delivery; a five-minute window is a reasonable default, and every retry is re-signed with a fresh timestamp.

### Delivery behavior you have to design for

Deliveries succeed on any `2xx` and retry with exponential backoff up to 12 attempts across roughly a day before being dead-lettered. A non-transient `4xx` other than `429` is not retried at all, on the reasoning that a retry cannot fix a misconfiguration. After a `429` or three consecutive failures, deliveries to that URL are suppressed for about five minutes.

So your handler must be idempotent. The same event can arrive more than once; deduplicate on the envelope `id`, or on `domain` plus `type`. And because dead-lettering is possible, the webhook should not be your only path to truth: reconcile against `GET /v1/connections` on a schedule, or call `:recheck` when a user reports a discrepancy. `GET /v1/webhook-deliveries` shows attempts, status codes and errors when you are debugging why an event never landed.

## One limitation worth knowing before you build on drift alerts

`domain.record_missing` and `domain.record_restored` come from a monitor sweep that compares a connection's declared records against live DNS. Two caveats. It is record-drift detection only: nothing probes the domain or your origin for reachability, status code, or latency, so it is not uptime monitoring and should not be wired to an on-call rotation as if it were. And delivery of those two events is gated by a deployment flag that is off by default on the hosted service, so the sweep runs but the alerts do not send unless it is enabled for your tenant. Ask before you design a customer-facing "your domain is broken" email around them. The connection lifecycle events (`connection.live`, `connection.failed`, `connection.disconnected`) are not affected.

## Reference

- [Widget install and embed](https://docs.customdomain.ai/docs/widget-sdk/installation-and-embed)
- [Widget SDK reference](https://docs.customdomain.ai/docs/widget-sdk/reference)
- [Widget tokens](https://docs.customdomain.ai/docs/authentication/widget-tokens)
- [Create a connection](https://docs.customdomain.ai/docs/connect-flow/create-a-connection)
- [Connections: lifecycle, records, drift](https://docs.customdomain.ai/docs/concepts/connections)
- [Webhooks overview](https://docs.customdomain.ai/docs/webhooks/overview) and [verifying signatures](https://docs.customdomain.ai/docs/webhooks/verifying-signatures)
- The served spec: [api.customdomain.ai/v1/openapi.json](https://api.customdomain.ai/v1/openapi.json), OpenAPI 3.1, 67 paths at the time of writing

## About CustomDomain

This guide is maintained by the team behind [CustomDomain](https://customdomain.ai), which catalogs 63 DNS and registrar providers, 25 of them with an automatic path and 38 on a guided flow with automatic verification. Integrate through the [connect widget](https://customdomain.ai/connect-domain-widget) or the [REST API](https://customdomain.ai/custom-domain-api). Docs: [docs.customdomain.ai](https://docs.customdomain.ai/docs). Pricing starts at $0, with the free tier capped at 10 domain connections a year: [app.customdomain.ai/signup](https://app.customdomain.ai/signup).
