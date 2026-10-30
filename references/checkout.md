# Checkout

Yard's hosted checkout at `pay.yard.sh`; Yard is the merchant of record. Docs: https://docs.yard.sh/v1/platform/payments/checkout.md (parameters, products, gifts, the return flow), https://docs.yard.sh/v1/platform/payments/merchant-of-record.md, https://docs.yard.sh/v1/platform/payments/payouts.md. On a custom landing page `data-action="checkout"` and `window.yard.checkout()` build these links (landing-pages.md).

`https://pay.yard.sh/<team-username>/<slug>?...` (the first segment is the team's username: `yard team --json` `.active_team.username`):

| Param | Meaning |
| --- | --- |
| `tier` | tier id or key (never a name); omit for the default tier |
| `product` | a product key, instead of `tier` |
| `quantity` | seats for a `per_seat` tier (kept within its range; other tiers always buy 1), or 1-99 of a consumable |
| `interval` | `monthly` or `yearly` for subscription tiers and products (`month`, `year`, `annual`, `annually` accepted) |
| `gift` | gift purchase on a `gift_enabled` one-time tier (`true`, `1` or bare) |
| `ref` | affiliate code (credited on one-time purchases and free claims only) |
| `sandbox` | sandbox name: simulated checkout, no card charged |
| `return_to`, `for` | with `product` only: where to send the buyer afterwards, and the Yard user id the purchase is for (anyone else is asked to switch accounts) |

Invalid values are ignored. Trials go to `https://yard.sh/trial/<team-username>/<slug>` with `tier` and `sandbox` only.

- `return_to` must be an absolute URL on the project's own sites (`https://<team>.yard.sh/<slug>/...`, an active custom domain over https, or the origin of a Yard Auth redirect URI such as `http://localhost:3000`), or checkout shows Invalid Checkout Link. On the way back it adds `yard_product`, `yard_status` (`succeeded`, `processing`, `canceled`) and `yard_purchase` (the transaction id; absent on cancel and while a subscription's first payment is processing). A hint to refresh, never proof of payment: read the holdings or wait for `product.purchased` (products.md).
- Selling needs payout setup: until the owner finishes it, checkout, free claims and trials answer `410 payouts_missing` and the project stays out of Explore. `window.yard.project.team.payouts_ready` and `team_inactive` tell a page. Launching (projects.md) is not gated.
- Price of a tier: the base price with any volume bracket, then the early-access discount (one-time purchases only), then the coupon, then tax by the buyer's location. A product is its price (times `quantity` for a consumable) plus tax, with no discounts. A subscription that starts with a free trial takes no coupon.
- Checkout refuses a product buyer who holds no required tier, a `one_time` product already owned, a `subscription` product already running, and any product while the project is in pre-order.
- A custom domain (Pro, dashboard only) serves the landing page and counts as a `return_to` origin; it never serves sandboxes: https://docs.yard.sh/v1/platform/storefront/custom-domain.md.
