# Products

What a tier holder buys on top of the tier: one-time add-ons and DLC, consumables (gem packs, credits), add-on subscriptions. Docs: https://docs.yard.sh/v1/platform/projects/products.md (types, requirements, settings, selling, holdings, fulfillment, limits). A product never grants access to the project or changes the buyer's tier (the `X-Yard-*` headers and `"access": "users"` look at tiers only), and buyers need a Yard account. Plan gate: `.team_permissions.max_products` (a plain `400` names the limit).

## Types, keys, requirements

| `type` | Pays | Gets |
| --- | --- | --- |
| `one_time` | once | the product while they hold a required tier; bought once |
| `consumable` | per purchase, 1-99 at a time | whatever your app delivers; bought again and again, each purchase waits to be fulfilled |
| `subscription` | `price_cents` monthly, or yearly with `yearly_discount_percent` | the product while it runs, on its own schedule |

- `key` (same grammar as tier keys) is permanent: checkout, holdings and webhooks name it. A published key cannot be renamed and keeps its type for good.
- `requires`: tier keys, any one is enough; empty means any tier. A release with products needs at least one tier, every required key must be a tier of that release, and a required tier cannot be removed. Holding a tier means bought, claimed free, subscribed or granted; a trial or a past-due subscription does not count. Losing the tier leaves a `one_time` product `active: false`, sets a `subscription` product to end at period end, and changes nothing for consumables already bought.
- Price 99 to 1,000,000 cents, always full price (no coupons, gifts, trials or launch discounts). Icons are optional: exactly 256x256 PNG, JPEG or WebP, at most 1 MiB (only the dashboard resizes).

## Setting up

Products belong to a release, like pricing. The settings.json `products` block (project-files.md) is applied by `yard push` and GitHub sync: present, the release's products match it exactly (`[]` removes all), each `icon` path (relative to the directory holding `.yard`) uploads when its bytes change, a product without `icon` loses its icon, and a declared icon missing locally fails the push. Absent, products stay dashboard-managed.

`yard projects products <slug> [--release <tag|id>] --json` lists them; `add <slug> --spec <file|-> [--icon <path>]`, `edit <slug> <key> [--spec <file|->] [--icon <path> | --remove-icon]` (partial) and `rm <slug> <key> [--yes]` (buyers keep it) follow the same draft and `--release` rule as tiers (pricing.md).

## Selling and holdings

Signed-in holders buy on the project page's Products tab, from a custom page with `data-action="checkout" data-product="<key>"` (landing-pages.md), or from a checkout link (checkout.md: `product`, `quantity`, `interval`, `return_to`, `for`, `sandbox`). The return parameters are a hint to refresh, never proof of payment: read the holdings.

Holdings have one shape everywhere: `{ tier_keys: ["pro"], products: [{ key, name, type, requires, transaction_id | subscription_id, active, status, quantity?, acquired_at, requirement_met }] }`. Use a product while `active`; `tier_keys` are the tiers that let the visitor buy products (compare with each `requires`); each unfulfilled consumable purchase is its own entry until it is fulfilled.

Read them from a project page or the landing page with `GET __yard/products` (relative URL; `{ authenticated, tier_keys, products }`; `503 products_unavailable` means retry, not "holds nothing"), from a page script with `window.yard.ownership()` (`.tier_keys`, `.products`), from an app outside the project with a Yard Auth token (https://docs.yard.sh/v1/api/yard-auth/get-user-products.md), or from a server, `_service.js` included (the edge strips the browser session), with https://docs.yard.sh/v1/api/users/project-user-products.md and `products:read` (`?sandbox=` for a sandbox; `404` never bought, `409` pass the email). API keys are Pro-only, so on Basic use the page or token routes.

## Delivering consumables

Each consumable purchase waits unfulfilled until your app delivers it. Find it (holdings with `status: "unfulfilled"`, the `product.purchased` webhook, or https://docs.yard.sh/v1/api/fulfillments/list-pending-fulfillments.md), grant `quantity` times the product keyed on `transaction_id` so a retry never grants twice, then fulfill by `transaction_id`: `POST __yard/fulfillments/{transaction_id}` as the signed-in visitor, https://docs.yard.sh/v1/api/yard-auth/fulfill-user-purchase.md with a token, or https://docs.yard.sh/v1/api/fulfillments/fulfill-purchase.md with `products:fulfill`. Fulfilling twice returns the purchase unchanged; `409 purchase_refunded` means take the grant back; `409 purchase_not_paid` means try later. Fulfill only after a grant your own trusted code made. A purchase still unfulfilled 3 days after payment is refunded automatically, sandboxes included, and `product.refunded` fires with `refund_reason: "unfulfilled"`.

- Sandbox: product checkout is simulated. `__yard/products` and `__yard/fulfillments` on a sandbox's pages act on that sandbox, the API routes take `?sandbox=` (a service reads the name from `X-Yard-Sandbox`), and webhooks carry `sandbox`.
- `yard dev`: one `buyer:<product key>` persona per product holds it and the first qualifying tier; a consumable's buyer has one unfulfilled purchase that stays fulfilled until restart. No webhooks fire, nothing is refunded after 3 days, and the team API still reaches Yard, where personas do not exist (local-dev.md).
