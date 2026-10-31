# Yard Auth

Sign-in for projects: the Yard edge authenticates visitors and hands services trusted headers, so never build OAuth, sessions or password storage. Docs: https://docs.yard.sh/v1/platform/services/yard-auth.md (access modes, headers, session endpoints, guests, products, external apps, consent). Any access other than `public` needs `.team_permissions.yard_auth`.

## Access modes and headers

`access` on the service entry (services.md): `public` (everyone), `authenticated` (must sign in), `users` (must hold a tier, a running trial or a subscription; products do not count), `team` (members of the owning team only, for internal tools). Whenever the visitor is signed in, whatever the mode, the request carries:

| Header | Value |
| --- | --- |
| `X-Yard-User-Id` | the user's id, a lowercase UUID; stable, store it as `TEXT` |
| `X-Yard-Email` | omitted when the account has none |
| `X-Yard-Entitlement` | `none`, `trial`, `active` or `team` |
| `X-Yard-Tier` / `X-Yard-Tier-Key` | the display name and the **key** of the tier they hold, sent with `trial` and `active`; gate features on the key |
| `X-Yard-Sandbox` | the sandbox name, empty for the project itself |
| `X-Yard-Guest-Id` | the browser's guest id, signed in or out, once it has one |

- No `X-Yard-User-Id` means an anonymous visitor (possible only on `public` services). Clients cannot forge the headers: incoming `X-Yard-*` is stripped.
- Every member of the owning team, in any role, is `team` everywhere and passes every gate, so the team never buys its own project. Entitlement resolves team, else a paid subscription or purchase (`active`), else a running trial (`trial`), else `none`; canceled, past-due and refunded grant nothing. Verdicts are cached up to 60 s; a long-lived UI polls `__yard/auth/me`.
- Gated requests are answered by the edge before your code runs. A navigation (`Sec-Fetch-Dest: document` or `iframe`, or `text/html` in `Accept`): signed out gets a `302` to `__yard/auth/login`, signed in without a purchase on `users` a `302` to the sales page, and signed in outside the team on `team` a `403` "Team only" page. Anything else (`fetch`, curl, a WebSocket): `401 {"error_code": "sign_in_required"}`, `403 {"error_code": "purchase_required"}` or `403 {"error_code": "team_only"}`. So a page's `fetch` sees a lapsed session as a `401`: send the browser to `__yard/auth/login?return=<path>`.

## `__yard/*` endpoints

Answered by the edge at the project root (`/<slug>/__yard/...`, what the landing page reaches) and under every service (`/<slug>/<service>/__yard/...`), nowhere deeper. Call them with relative URLs from a page at that level (`../__yard/auth/me` from a subfolder).

- `login?return=<path>`: signs the visitor in (an existing Yard session passes silently) and redirects to `return`, a path relative to where login was called (`/` is the service's root, or the landing page at the project root; anything not starting with `/` falls back to `/`). The first sign-in by someone outside the team shows a consent screen once.
- `logout?return=<path>`: ends the project session (the person stays signed in to Yard).
- `me`: always `200`: `{ authenticated: true, user_id, email, entitlement, tier?, tier_key? }` or exactly `{ authenticated: false, entitlement: "none" }`.
- `guest`: a stable identity for signed-out visitors (guest play). `POST` issues the browser a guest id (kept 400 days), `GET` answers `{ guest: true|false }`, `DELETE` forgets it; the id itself is never returned. From then on every request from the project's own pages, WebSocket upgrades included, carries `X-Yard-Guest-Id`; only the project's pages may issue or forget one (`403 cross_origin`). After a guest signs in both ids arrive: move the guest's rows to the user, then `DELETE`. Never mint your own token.
- `products` (GET) and `fulfillments/{transaction_id}` (POST): the visitor's product holdings and consumable delivery (products.md).

A session covers the landing page, every service and every sandbox of one project on one host (the `yard.sh` address and a custom domain each sign in separately). The cookie is HttpOnly, Secure and SameSite=Lax, honored only for requests from the project's own pages and for top-level navigations: a fetch, form post or WebSocket from a page on another host arrives signed out. Never change state on `GET`, since a link from elsewhere still arrives signed in.

## External apps

Software outside the project (desktop, mobile, a backend elsewhere) signs users in with the project as a standard OpenID Connect application (issuer, client, redirect URIs and claims: the docs page above, section External apps). The access token identifies one user of one project and reaches exactly three endpoints: https://docs.yard.sh/v1/api/yard-auth/get-userinfo.md (identity and entitlement; call it on every launch, since purchase status is not in the ID token), https://docs.yard.sh/v1/api/yard-auth/get-user-products.md and https://docs.yard.sh/v1/api/yard-auth/fulfill-user-purchase.md. Redirect URI origins also count as checkout `return_to` origins (checkout.md).
