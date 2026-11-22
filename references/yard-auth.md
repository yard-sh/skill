# Yard Auth

Sign-in for projects: the Yard edge authenticates visitors and hands services trusted headers, so never build OAuth, sessions or password storage. Docs: https://docs.yard.sh/v1/platform/services/yard-auth.md (access modes, headers, session endpoints, guests, products, external apps, consent). Any access other than `public` needs `.team_permissions.yard_auth`.

## Access modes and headers

`access` on the service entry (services.md): `public` (everyone), `authenticated` (must sign in), `users` (must hold a tier, a running trial or a subscription; products do not count), `team` (members of the owning team only, for internal tools). Whenever the visitor is signed in, whatever the mode, the request carries:

| Header | Value |
| --- | --- |
| `X-Yard-User-Id` | the id this project knows the user by, a lowercase UUID; store it as `TEXT` |
| `X-Yard-Email` | omitted when the account has none |
| `X-Yard-Entitlement` | `none`, `trial`, `active` or `team` |
| `X-Yard-Tier` / `X-Yard-Tier-Key` | the display name and the **key** of the tier they hold, sent with `trial` and `active`; gate features on the key |
| `X-Yard-Sandbox` | the sandbox name, empty for the project itself |
| `X-Yard-Guest-Id` | the browser's guest id, signed in or out, once it has one |

| Entitlement | Meaning |
| --- | --- |
| `none` | signed in, holding none of the project's tiers; a refunded purchase, a past-due or ended subscription and a product on its own all count as `none` |
| `trial` | a running free trial of a tier, including a subscription not yet charged |
| `active` | holds a tier, through a one-time purchase or a paid subscription |
| `team` | a member of the owning team, in any role, whatever they bought; carries no tier |

- The user id is per project: the same person has a different id on each project, even on two projects of one team, while emails and usernames are the same everywhere. Within one project it is the same in the project and every one of its sandboxes, and everything yard gives the project names the person by it (`__yard/auth/me`, `ownership()`, external apps' sign-in, webhooks, checkout's `for`, the users the dashboard, CLI, MCP tools and API list), so key data by it within the project and never use it to match people across projects (use the email for that, when the app has it). A team-wide users list has a row and an id for each project a person signed in to or bought, and the subscribers list one for each project they subscribed to. Local dev personas have fixed made-up ids (local-dev.md).
- No `X-Yard-User-Id` means an anonymous visitor (possible only on `public` services). Clients cannot forge the headers: incoming `X-Yard-*` is stripped, except a webhook delivery's `X-Yard-Signature` and `X-Yard-Event` (webhooks.md), which pass so a service can receive its project's webhooks.
- Every member of the owning team, in any role, is `team` everywhere once signed in and passes every gate, so the team never buys its own project. Signing in also gives their browser team access, which is what opens a private or draft project, a private sandbox and a `team` service: `logout` leaves it (a developer still sees their own draft project signed out), and signing out at yard.sh ends it and their sign-in on the project within a minute (anyone off the team stays signed in to the project). A `team` service always gets the team member's headers with `team`, even while they are signed out on the project; `authenticated` and `users` follow the visitor's sign-in alone, so a member who signed out on the project is asked to sign in again there, on a draft project too. Entitlement resolves team, else a paid subscription or purchase (`active`), else a running trial (`trial`), else `none`. A change of entitlement reaches the service within about 60 s; a long-lived UI polls `__yard/auth/me`.
- Gated requests are answered by the edge before your code runs. A navigation (`Sec-Fetch-Dest: document` or `iframe`, or `text/html` in `Accept`): signed out gets a `302` to `__yard/auth/login`, signed in without a purchase on `users` a `302` to the sales page, and signed in outside the team on `team` a `403` "Team only" page. Anything else (`fetch`, curl, a WebSocket): `401 {"error_code": "sign_in_required"}`, `403 {"error_code": "purchase_required"}` or `403 {"error_code": "team_only"}`. So a page's `fetch` sees a lapsed session as a `401`: send the browser to `__yard/auth/login?return=<path>`.

## `__yard/*` endpoints

Answered by the edge at the project root (`/<slug>/__yard/...`, where the landing page, the service at `/`, reaches them) and under every other service (`/<slug>/<service>/__yard/...`), nowhere deeper. Call them with relative URLs from a page at that level (`../__yard/auth/me` from a subfolder).

- `login?return=<path>`: signs the visitor in (an existing Yard session passes silently) and redirects to `return`, a path relative to where login was called (`/` is the service's root, or the landing page at the project root; anything not starting with `/` falls back to `/`). The first sign-in shows a consent screen once, the owning team included.
- `logout?return=<path>`: ends the visitor's sign-in on the project; they stay signed in to yard, and a team member keeps team access.
- `me`: always `200`: `{ authenticated: true, user_id, email, username?, avatar_url?, entitlement, tier?, tier_key?, holding }` or exactly `{ authenticated: false, entitlement: "none" }`. `holding` is what the visitor holds themselves, as `window.yard.ownership()` reports it (`owned`, `is_trial`, `trial_expires_at`, `trial_used`, `is_subscription`, `transaction_id`, `tier_id`, `tier_key`, `tier_name`): a team member's `entitlement` is `team` while `holding` says what they bought.
- `guest`: a stable identity for signed-out visitors (guest play). `POST` issues the browser a guest id (kept 400 days), `GET` answers `{ guest: true|false }`, `DELETE` forgets it; the id itself is never returned. From then on every request from the project's own pages, WebSocket upgrades included, carries `X-Yard-Guest-Id`; only the project's pages may issue or forget one (`403 cross_origin`). After a guest signs in both ids arrive: move the guest's rows to the user, then `DELETE`. Never mint your own token.
- `products` (GET) and `fulfillments/{transaction_id}` (POST): the visitor's product holdings and consumable delivery (products.md).

A session and a team member's team access each cover the landing page, every service and every sandbox of one project on one host (the `yard.sh` address and a custom domain each sign in separately). Page scripts cannot read either, and both count only for requests from the project's own pages and for top-level navigations: a fetch, form post or WebSocket from a page on another host arrives signed out. Never change state on `GET`, since a link from elsewhere still arrives signed in.

## External apps

Software outside the project (desktop, mobile, a backend elsewhere) signs users in with the project as a standard OpenID Connect application (issuer, client, redirect URIs and claims: the docs page above, section External apps). The sign-in must ask for the `yard_account` scope (and `email` for the address), or the endpoints answer `403 insufficient_scope`. Every call also names the app it is made for, by its client ID, and is answered only for a token that was issued to that app, so a server can trust the answer for a token a client handed it; a call that names no app is refused. The access token identifies one user of one project and reaches exactly three endpoints: https://docs.yard.sh/v1/api/yard-auth/get-userinfo.md (identity and entitlement; call it on every launch, since purchase status is not in the ID token; its ids and the ID token's `sub` are the project's id for the person, the same value as `X-Yard-User-Id`), https://docs.yard.sh/v1/api/yard-auth/get-user-products.md and https://docs.yard.sh/v1/api/yard-auth/fulfill-user-purchase.md. Redirect URI origins also count as checkout `return_to` origins (checkout.md).
