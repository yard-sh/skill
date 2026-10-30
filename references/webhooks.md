# Webhooks

Yard posts a signed JSON payload to one URL per project when a tier or product is bought, renewed or refunded, a subscription or product subscription is set to end, resumes, falls past due, ends or changes tier, a gift is activated, or a trial starts or ends. Docs: https://docs.yard.sh/v1/webhooks/getting-started/quick-start.md (every event with its payload), https://docs.yard.sh/v1/webhooks/getting-started/signature-verification.md.

- Pro only. One webhook per project, set on the dashboard's Webhooks page (the signing secret is also readable with an API key). Each delivery carries `X-Yard-Signature`: verify it before acting.
- Every moment has one tier event and one product event, and a product never fires the tier one: `sale.completed` / `product.purchased`, `subscription.renewed` / `product.renewed`, `sale.refunded` / `product.refunded`, `subscription.ending` / `product.ending`, `subscription.resumed` / `product.resumed`, `subscription.past_due` / `product.past_due`, `subscription.canceled` / `product.canceled`; plus `subscription.tier_changed`, `gift.activated`, `trial.started` and `trial.ended`.
- Deduplicate on `event_id` (the same on every retry). Tier events carry `tier_key`; product events carry `product_key` (tell products apart by it: `product_id` changes between releases). `sandbox` is set on simulated purchases. `user_display_id` is the account UUID, the same value as `X-Yard-User-Id`.
- `product.refunded` with `refund_reason: "unfulfilled"` is the automatic 3-day refund of an undelivered consumable (products.md).
