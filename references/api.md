# REST API

For shipped software and automation: validating licenses, serving updates, managing a user's existing subscription, reading and fulfilling products, Yard Auth in external apps, publishing from CI. An agent managing the team's own catalog uses Yard's MCP server or the CLI instead; never create a project over HTTP.

## Where the reference lives

The endpoint reference is online and generated from the API, so this file names no endpoint shapes. Read an endpoint's page before calling it; never call one from memory.

- Index: https://docs.yard.sh/llms.txt lists every docs page with a one-line description (its header also names the docs search endpoint); each REST entry names the method, path and the credential it takes.
- Any page as Markdown: append `.md` to its URL (`https://docs.yard.sh/v1/api/licenses/validate-license.md`), or request it with `Accept: text/markdown`. An endpoint page gives the request table, JSON examples, response fields and errors, and is checked against the API on every build.
- Exact shapes: https://api.yard.sh/openapi.json (OpenAPI 3.1, about 550 KB: read `.paths | keys` or one path with `jq`).
- Start here: https://docs.yard.sh/v1/api/getting-started/introduction.md, https://docs.yard.sh/v1/api/getting-started/authentication.md (scope catalog), https://docs.yard.sh/v1/api/getting-started/error-handling.md, https://docs.yard.sh/v1/api/getting-started/conditional-writes.md.

## Addressing

Base `https://api.yard.sh`. Integration endpoints address a project as `/v1/projects/{username}/{slug}/…`, where `username` is the owning **team's** username (`yard team --json` `.active_team.username`), which routinely differs from the signed-in user's. Management endpoints take the project UUID, `/v1/projects/{id}/…` (`yard projects show <slug> --json | jq -r .id`).

## Credentials

- **API key** (`Authorization: Bearer yard_...`): a team credential, Pro only (a Basic team's keys answer `403 upgrade_required`). `yard keys create [name] --scopes <csv> [--expires-in-days 7|30|60|90|365] --json` prints the secret once (`.key`); without `--scopes` it prints the scope catalog; `yard keys list` shows prefixes only. Keys survive their creator leaving; up to 100 per team. A key reaches exactly the endpoints its scopes allow (`403 insufficient_scope` otherwise; endpoints outside the reference answer `401`), and scopes do not imply one another. Integration scopes (`metadata:read`, `licenses:validate`, `licenses:activate`) are safe to ship inside an app (every install shares the key and its rate limit, so validate once at launch and keep the answer). Management scopes (`projects:*`, `releases:*`, `sandboxes:*`, `services:*`, `secrets:write`, `db:query`, `users:read`, `subscriptions:*`, `transactions:*`, `coupons:*`, `products:read`, `products:fulfill`) act on the team's account and users: keep such keys on servers the team controls; `releases:read` downloads every file in a public channel without a purchase, so a shipped app updates with the user's license key instead (updates.md). No key can manage API keys, the team, its plan, payouts, custom domains, project media, the webhook secret or the GitHub App install: those are session-only.
- **License key** (`Authorization: Bearer <license key>` or `?license_key=`): the user's own credential for the `/v1/updates/…` endpoints (updates.md).
- **Yard Auth access token**: one user of one project, from an external app's OpenID Connect sign-in; reaches the three `/v1/yard-auth/…` endpoints (yard-auth.md).

## The `sandbox` parameter

An omitted or empty `sandbox` means the project itself, the only place money is real; naming a sandbox selects it (query string on reads and subscription writes, request body on license validation; each page says). A private sandbox answers only the owning team and gives everyone else the same `404` as an unknown name. Shipped software never sends `sandbox`: license validate and deactivate find a sandbox's keys only when the body names it, and an unknown sandbox there answers like an unknown key (`200`, `valid: false`, `license_not_found`), so nothing leaks.

## Release writes need a base

Every write to a release's contents says what it was based on: `If-Match: "<version>"` (a file's `content_hash`, a product icon's `sha256`, or the matching entry of the release's `versions`), or `If-None-Match: *` to create a file that must not exist. A write answers its new version in `ETag`. A moved base is `412 stale_base` (read again and redo the change); no base is `428 base_required`. Creating, publishing, promoting and archiving a release take no base.

## Errors

Every error is `{ error, error_code, error_id }` (the page-route `__yard/*` refusals carry no `error_id`). Branch on `error_code`, never on the text. Common: `unauthorized` (401), `insufficient_scope` (403), `upgrade_required` and the `*_limit_reached` codes (the team's plan), `subscription_inactive` (402: the team's own Yard plan lapsed, so its keys stop working), `stale_base` (412), `base_required` (428), `rate_limited` (429). License validation answers `200` with `valid: false` and its own codes (license-keys.md); the update endpoints use 400/403/404 (updates.md). The full conventions and the `*_limit_reached` list are on the Error Handling page.
