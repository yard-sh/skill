# Checkout

Yard's hosted checkout at `pay.yard.sh`; Yard is the merchant of record. Docs: https://docs.yard.sh/v1/platform/payments/checkout.md (parameters, products, gifts, the return flow), https://docs.yard.sh/v1/platform/payments/merchant-of-record.md, https://docs.yard.sh/v1/platform/payments/payouts.md. On a landing page `data-action="checkout"` and `window.yard.checkout()` build these links (landing-pages.md).

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
| `return_to`, `for` | with `product` only: where to send the buyer afterwards, and the user id the purchase is for, as the project's service got it in `X-Yard-User-Id` (anyone else is asked to switch accounts; it must be the id from the project being bought, since another project's id names no one) |
| `from` | the host the visitor is on (`acme.com`): with it, a visitor signed in to the project with a project account arrives at checkout already named; the page script adds it by itself |

Invalid values are ignored. Trials go to `https://yard.sh/trial/<team-username>/<slug>` with `tier`, `sandbox` and `from` only.

- `return_to` must be an absolute URL on the project's own sites (`https://<team>.yard.sh/<slug>/...`, an active custom domain over https, or the origin of a Yard Auth redirect URI such as `http://localhost:3000`), or checkout shows Invalid Checkout Link. On the way back it adds `yard_product`, `yard_status` (`succeeded`, `processing`, `canceled`) and `yard_purchase` (the transaction id; absent on cancel and while a subscription's first payment is processing). A hint to refresh, never proof of payment: read the holdings or wait for `product.purchased` (products.md).
- Selling needs payout setup: until the owner finishes it, checkout, free claims and trials answer `410 payouts_missing` and the project stays out of Explore. `window.yard.project.team.payouts_ready` and `team_inactive` tell a page. Launching (projects.md) is not gated.
- Price of a tier: the base price with any volume bracket, then the early-access discount (one-time purchases only), then the coupon, then tax by the buyer's location. A product is its price (times `quantity` for a consumable) plus tax, with no discounts; a free product asks for no card. A subscription that starts with a free trial takes no coupon.
- Checkout refuses a product buyer who holds none of the tiers the product requires (a product requiring none sells to anyone signed in), a `one_time` product already owned, a `subscription` product already running, and any product while the project is in pre-order.
- A custom domain (Pro, dashboard only) serves the landing page and counts as a `return_to` origin; it never serves sandboxes: https://docs.yard.sh/v1/platform/storefront/custom-domain.md.

## Project accounts

On a project with project accounts on (yard-auth.md), a visitor signed in with one buys without a yard account. Docs: https://docs.yard.sh/v1/platform/payments/checkout.md, sections Buying with a Project Account, Buying Without Signing In and Gifts.

- Buy links don't change, and nothing in the project's code does. A Buy click passes through the project's host and back, and checkout opens already naming the buyer ("Buying as ada@example.com on Acme") with a link back. They type a card: checkout never offers a card or an address saved earlier, and a subscription renews on the card typed for it. A yard account buys under its yard sign-in, on every project. Tiers, subscriptions, products, trials and free claims all work; a project account buys only where it was made (the project itself, or its one sandbox).
- The project sees the purchase at once, exactly as for a yard account, keyed by the project's user id: `X-Yard-Entitlement`, `__yard/auth/me`, `ownership()`, `__yard/products`, webhooks and the Users page. Never look a buyer up by email.
- The receipt goes to the account's own address (it can't be changed at checkout) and says where to manage the purchase: at yard.sh, signed in with that address. Signing in there (which makes a yard account if there is none) shows it in the library: receipt, downloads, license keys, cancel and resume, and the card a subscription charges. Only a confirmed address finds it, and a yard account's own purchases are never found by address. What the project sees stays on the project account.
- Signed out: on a project with project accounts on, a one-time tier bought with a typed address makes a project account with that address, which only a link mailed to it can enter (the buyer types the address on the project's sign-in page). If the address already has an account there, checkout asks them to sign in. With project accounts off it makes a yard account from the address. Subscriptions, products, trials and free claims always need a sign-in. Checkout's Log in goes to the project's sign-in page and back, with a smaller Use a yard account.
- "Sign in required for purchases" (the project's Authentication page, off by default, every project has it) turns signed-out buying off for both kinds of account. Read it as `purchase_sign_in_required` in `window.yard.project` (beside `project_accounts`), `yard projects show` and MCP `get_project`. Only the dashboard changes it: tell the user where it is, never try to change it.
- Gifts: Claim sends the recipient to the project's sign-in page, and the gift lands on the account they sign in with; nobody needs a yard account to claim. A gift bought in a sandbox is always claimed on yard's gift page.
- Deleting a project account that bought ends its subscriptions and trials at once with a refund of the unused part (yard-auth.md, Deleting a project account).
