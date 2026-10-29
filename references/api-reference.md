# Yard API Reference

> **What this API is for.** Integrating Yard into shipped software: validating licenses, serving updates, managing your users' existing subscriptions, reading and fulfilling the [products](products.md) they buy, and signing your users in with Yard Auth. An agent managing the team's own catalog (projects, pricing, products, releases, pages, services, coupons, users, sales) uses [Yard's MCP server](../SKILL.md#yards-mcp-server) when it is connected, or the **Yard CLI** ([cli-commands.md](cli-commands.md)). Never create a project over HTTP.

## Where the reference lives

The endpoint reference is online and generated from the API itself, so this file names no endpoint shapes. **Read an endpoint's page before calling it; never call one from memory.**

- **The index:** https://docs.yard.sh/llms.txt lists every docs page with a one-line description. Each REST API entry names the endpoint, method and path, and the credential it takes: an API key scope, a license key, a Yard Auth access token, or none. Fetch it first to see what exists.
- **A page as Markdown:** append `.md` to any page URL (`https://docs.yard.sh/v1/api/licenses/validate-license.md`), or request the page with `Accept: text/markdown`. An endpoint page gives the request table, JSON examples, response fields and errors, and is checked against the API on every build.
- **Exact shapes:** https://api.yard.sh/openapi.json is the OpenAPI 3.1 document of every public operation with its request and response schemas. It is about 550 KB: read `.paths | keys`, or one path, with `jq`, not the whole file.
- **Start here:** https://docs.yard.sh/v1/api/getting-started/introduction, then [Authentication](https://docs.yard.sh/v1/api/getting-started/authentication) (the scope catalog), [Error Handling](https://docs.yard.sh/v1/api/getting-started/error-handling) and [Conditional Writes](https://docs.yard.sh/v1/api/getting-started/conditional-writes).
- **Offline:** `yard keys create` prints the scope catalog and `yard <command> --help` is always current. The endpoint reference itself needs the network.

Create an API key with the scopes you need at **https://dash.yard.sh/api-keys?action=create** or with `yard keys create`.

## Base URL and addressing

```
https://api.yard.sh
```

Paths are relative to it: `/v1/licenses/validate` is `https://api.yard.sh/v1/licenses/validate`.

Integration endpoints address a project by its owning team's username plus its slug, `/v1/projects/{username}/{slug}/…`, matching the public URL `https://yard.sh/@{username}/{slug}` and the subdomain `https://{username}.yard.sh/{slug}`. Management endpoints take the project UUID instead, `/v1/projects/{id}/…` (`yard projects show <slug> --json | jq -r .id`). A team username is **not** the user's username (they share one namespace but routinely differ): read it from `yard team --json` → `.active_team.username` or `team.username` on the public project, never assume it matches the signed-in user.

## Credentials

Three kinds. Each endpoint page says which one it takes.

### API key (integrations and the team's servers)

```
Authorization: Bearer yard_{key}
```

API keys start with `yard_` and belong to a **team** (created with `yard keys create` or in the dashboard); a key keeps working when the person who minted it leaves. A key reaches exactly the endpoints its scopes allow: a listed endpoint whose scope the key lacks answers `403` with `insufficient_scope`, and an endpoint outside the REST API section answers `401` whatever the scopes. Scopes do not imply one another; pick only what you use.

- **Integration scopes** (`metadata:read`, `licenses:validate`, `licenses:activate`) are safe to ship inside the app your users run. Every install shares the key and its rate limit, so validate once at launch and keep the answer.
- **Management scopes** act on the team's account, its release files and its users: keep keys holding them on servers the team controls. `releases:read` downloads every file in a public channel without a purchase, the subscription scopes act on the subscription of any user named by email, the product scopes read and fulfill any user's purchases, and `db:query` runs SQL. A shipped app lists channels, checks for updates and downloads with the user's **license key** instead.
- A key with the matching scope can do over HTTP what the CLI does, except manage API keys, the team, or money.

### License key (the update endpoints)

A built-in updater reaches the `/v1/updates/…` endpoints with just the paying user's license key, as `?license_key=` or `Authorization: Bearer <license key>`, so no shared secret ships in the binary. The workflow is in [releases-and-updates.md](releases-and-updates.md#downloading-releases-with-a-license-key); the pages are the Updates category of the reference.

### Yard Auth access token (a user, in an external app)

Software that runs **outside** the project (a desktop or mobile app, a backend on another host) signs its users in with the project's own OpenID Connect issuer; the issuer, client, redirect URIs, scopes and claims are on [Yard Auth: External apps](https://docs.yard.sh/v1/platform/services/yard-auth#external-apps), and the hosted side is in [service-and-database.md](service-and-database.md#identity-yard-auth-never-your-own). The access token identifies **one user of one project**, never the team, and reaches exactly three endpoints: [Get User Info](https://docs.yard.sh/v1/api/yard-auth/get-userinfo) (identity and entitlement; call it on every launch, since purchase status is not in the ID token and changes underneath it), [A User's Products](https://docs.yard.sh/v1/api/yard-auth/get-user-products) and [Fulfill a User's Purchase](https://docs.yard.sh/v1/api/yard-auth/fulfill-user-purchase). Requires `.team_permissions.yard_auth` on the owning team.

### Sessions (CLI and dashboard only)

The CLI and dashboard use a signed-in session; integrations never do. Whatever an API key's scopes, it cannot mint or manage API keys, manage the team or its plan, payouts or payment methods, custom domains, project media, the webhook secret, the GitHub App install, or account security: those endpoints are not in the REST API section and answer `401`.

## The `sandbox` parameter

A project has its own real data plus any number of **sandboxes**, each an optional copy with simulated commerce ([pricing-and-licensing.md](pricing-and-licensing.md#commerce-in-a-sandbox)). Everywhere, an **omitted or empty `sandbox` means the project itself**, the only place money is real; naming a sandbox selects it. Each page says where the parameter travels: the query string on reads and on the subscription-management writes, the request body on license validation. An unknown sandbox is a `404` naming it. A **private** sandbox (the default) answers only members of the owning team and gives everyone else the same `404`; a **public** one answers anyone. Responses that resolve a sandbox are never cacheable.

Shipped software never sends `sandbox`: a sandbox license key is found only when the request names its sandbox, so a simulated purchase can never unlock the real build.

## Release writes need a base

The dashboard, the CLI and agents may change the same release, so every write to a release's contents says what it was based on: `If-Match: "<version>"` (a file's `content_hash`, a product icon's `sha256`, or the matching entry of the release's `versions`), or `If-None-Match: *` to create a file that must not exist yet. A write answers its new version in `ETag`. A base that moved is `412` with `stale_base`: read it again and redo the change on what it holds now. A write with no base is `428` with `base_required`. Creating, publishing, promoting and archiving a release take no base. The writes this covers, with examples: [Conditional Writes](https://docs.yard.sh/v1/api/getting-started/conditional-writes).

## Errors

Every error is a JSON body with `error` (a message for people), `error_code` (a lowercase code to branch on, present on the errors a client may handle on its own) and `error_id` (identifies the request for support). **Branch on `error_code`, never on the `error` text.** The codes and status conventions are on [Error Handling](https://docs.yard.sh/v1/api/getting-started/error-handling), and each endpoint page lists its own. The ones an agent meets most:

- `unauthorized` (`401`) and `insufficient_scope` (`403`): the key, or its scopes.
- `upgrade_required` and the `*_limit_reached` codes: the team's plan lacks the feature or has reached a limit. `subscription_inactive` (`402`): the team's own Yard plan lapsed, so its API keys stop working until it is active again.
- `stale_base` (`412`) and `base_required` (`428`): see above.
- `rate_limited` (`429`): back off and retry.
- License validation answers `200` with `valid: false` for an unusable key; that is not an HTTP error. Its `error_code` says why, with the full list on [Validate License](https://docs.yard.sh/v1/api/licenses/validate-license); `license_validation_disabled` means the team's plan, not the user's key. The update endpoints refuse a key with the same codes as `403`s.
