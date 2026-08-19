# Contributing

This is a field guide, not a codebase. The most valuable contribution is a correction.

## Corrections

Open an issue with the file, the sentence, and what it should say. If the correction is a number, a record, or a claim about how a provider behaves, include the source you checked. Sources that settle an argument here:

- Provider counts: the public census, `https://api.customdomain.ai/v1/providers/census`
- API shapes and field names: the served spec, `https://api.customdomain.ai/v1/openapi.json` (OpenAPI 3.1), and [docs.customdomain.ai](https://docs.customdomain.ai/docs)
- Product behavior (connection states, webhook events, plan entitlements): [docs.customdomain.ai](https://docs.customdomain.ai/docs)
- DNS and TLS standards: the RFC, not a blog summary

A pull request is welcome for anything you can state in a sentence or two. For a larger rewrite, open an issue first so the change does not collide with one already in flight.

## House style

- Plain language. Short paragraphs. Tables where they clarify.
- No em dashes or en dashes anywhere in this repo.
- American English.
- Real DNS and TLS facts only. No invented numbers, no rounded-up counts, no capability the product does not have.
- Write the brand as **CustomDomain**, one word. The spaced form is the generic category and means something else. Never rename a machine-readable identifier: Domain Connect `providerId` and `serviceId` values, package names, and URLs stay exactly as they are.
- Say the exact number. "25 providers have an automatic path" is correct; "25+" and "more than 25" are not, because 25 is the count. 63 is the catalogued total and must never be attached to an automatic verb.
- Prefer a documented limitation to a soft claim. If something is gated, off by default, or not built, say so.

## Links

Every external link should be one you actually loaded. If a link returns 403 from an anonymous fetch, that is usually a bot block (GitHub and npm both do this) rather than a dead link, so verify before deleting. Write links to their final target: `docs.customdomain.ai/docs/...`, not the deprecated `app.customdomain.ai/docs/...` redirect.

## License

The content is MIT licensed. By contributing you agree your contribution is licensed the same way. Adapting any of this for your own product's help center is expressly fine and needs no permission.
