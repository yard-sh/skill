# Landing pages

A project's landing page at `https://<team>.yard.sh/<slug>/` is optional: it is the developer's own files, shipped in a release, and a project without them shows a Not Found page there (checkout, trials and the library work without one). Building one needs `landing_pages` (`yard me --json` `.team_permissions`). Docs: https://docs.yard.sh/v1/platform/storefront/landing-page.md (adding a page, layout, limits, video, removing it), https://docs.yard.sh/v1/platform/storefront/landing-page/integration.md (`window.yard`, every attribute and field, embedding, CSP), https://docs.yard.sh/v1/platform/storefront/custom-domain.md.

## Bundle

- Files live in `.yard/landing-page/`, or the directory `.yard/settings.json` names in `"landing_page": { "dir": "site" }` (project-files.md). They are the release's landing page; a release without them has none. `yard init --page` scaffolds a starter page in a linked project and pushes it (works without a terminal).
- Extensions `.html .css .js .json .svg .png .jpg .jpeg .webp .gif .woff2 .mp4 .webm .vtt`; names of letters, digits and `._-`; at most one subdirectory; no dotfiles; `__yard/` is reserved; `index.html` is the entry point, and a page without one serves nothing. `index.html` goes only with the page's last other file: deleting every file removes the page. Counts and sizes come from the plan (`page_max_*`). Video: H.264/AAC `.mp4`, `muted playsinline` to autoplay, `preload="metadata"` with a `poster`, captions as `.vtt`.
- Use relative URLs (`href="styles.css"`, `fetch("app/items")`): the page serves under `/<slug>/` and `/<slug>/@<sandbox>/`, so a root-absolute `/styles.css` 404s.
- A page that sends its own `Content-Security-Policy` must allow `script-src https://yard.sh`, `frame-src https://yard.sh` and `style-src 'unsafe-inline'`, or `embed.js` and the hooks below break.

## Hooks

Yard injects `embed.js` at the start of the page's head. `window.yard.project` is the public project JSON (https://docs.yard.sh/v1/api/public/get-public-project.md), synchronous, `null` when it could not load. An empty field is absent rather than `null` (check for the key), and the object reflects the saved release.

- `data-yard="<dotted path>"` sets `textContent` from `project` (`title`, `tiers.0.name`, `latest_release.tag_name`); add `data-yard-html` to write `innerHTML` (`latest_release.release_notes_html`). Copy such as a tagline, a description or an FAQ is the page's own text. After inserting DOM yourself, call `window.yard.refresh()`.
- `data-action="checkout"` with `data-tier-id` (the tier **key**; omit for the default tier), `data-interval` (`monthly` | `yearly`), `data-quantity` (seats on a `per_seat` tier, or 1-99 of a consumable), `data-gift`, `data-product="<key>"` (buy a product; the buyer must be signed in, and hold a required tier if it requires any) and `data-return-to` (with a product: a URL or a path relative to the page; bare means this page). `data-action="trial"` takes an optional `data-tier-id` of a trial-enabled tier. Clicks are handled and their default prevented; the attribute wins over `href`, so never reuse such an element as a plain link.
- `data-yard-when="signed_in|signed_out|owned|not_owned"` shows the element only in that state, by a rule page CSS cannot override; `embed.js` never touches the `hidden` attribute. On a first visit such elements stay hidden until the state resolves. The browser remembers the state for the page, so later visits start in it at first paint and switch if the fresh answer differs.
- `<html>` carries the same state for page CSS, set before paint: `data-yard-pending` (no answer yet), `data-yard-signed-in`, `data-yard-owned`. `html[data-yard-owned] .buy { display: none; }` shows Buy while Yard checks on a first visit and hides it for a known owner.
- JS: `window.yard.checkout({ tier?, interval?, quantity?, gift? } | { product, quantity?, interval?, returnTo?, for? })`, `trial({ tier? })`, `checkoutURL()`, `trialURL()`, `ownership()`, `purchase`, `refresh()`, `checkoutBase`. Link parameters and the `return_to` rules: checkout.md. After a product checkout returns, `window.yard.purchase` is `{ product, status, purchase_id }` (taken off the URL) and `ownership()` already counts a `succeeded` purchase.

## Visitor state

`await window.yard.ownership()` is about this visitor's yard.sh account: `{ signed_in, user: { id, username, avatar_url } | null, owned, is_trial, is_subscription, trial_expires_at, trial_used, transaction_id, tier_key, tier_name, tier_id, tier_keys, products }`. `trial_used` is true once the visitor has had the project's one free trial, running or ended: hide a trial button then. Match `tier_key` against `project.tiers[].key` (a holder from an older pricing revision has a `tier_id` the page no longer lists); `tier_keys` and `products` are the product holdings (products.md). It always waits for the live answer, never the remembered one, and returns `null` when Yard cannot answer (no reply within 8 s, or a page on an address Yard can't tie to the project): treat that as signed out. On a custom domain, browsers that block third-party cookies (Safari) resolve `signed_in: false` for a signed-in visitor, so default to the Buy button.

When the project has services, the page can also use the project's Yard Auth session (what services see as `X-Yard-*`): relative `fetch("__yard/auth/me")`, links to `__yard/auth/login?return=/` and `__yard/auth/logout?return=/`, `__yard/products` and `POST __yard/fulfillments/{transaction_id}` (yard-auth.md).

## Testing

`yard dev` serves the page at `http://localhost:9875/<slug>/` with `embed.js`, live project data when logged in (otherwise a placeholder from settings.json with no `title` or releases, so those bindings keep the page's own text) and `ownership()` from the chosen persona (local-dev.md). Hosted, each sandbox serves the page at `/<slug>/@<name>/` with its own pricing; the try-then-ship recipe is in sandboxes.md.
