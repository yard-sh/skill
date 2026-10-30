# Pricing tiers

Tiers decide what a buyer pays and how seats are counted. Docs: https://docs.yard.sh/v1/platform/projects/pricing.md (models, seats, volume discounts, trials, gifts, price rules). Tier features are plan-gated on the team: read `yard me --json` `.team_permissions` (`max_pricing_tiers`, `seat_based_pricing`, free trials); a refusal about tiers is a plain `400` naming the rule, not `upgrade_required`.

## Tier fields

`key`, `name` (1-100, unique in the release, case-insensitive), `description` (up to 2000), `features` (up to 20 strings of up to 200 chars), `sort_order`, `price_cents` (`0` for a free one-time tier, else 300 to 1,000,000; subscription tiers are never free), `is_default` (exactly one per release; the first tier added is the default), `pricing_model` (`one_time` | `subscription`), `yearly_discount_percent` (subscriptions, 1-100), `seat_type`, `free_trial` (`enabled`, `days` 7-365, `requires_card`), `gift_enabled`. Pricing belongs to a release: the settings.json `pricing.tiers` block and `yard push` set the whole list (project-files.md); `yard projects tiers` edits one.

## Tier keys

- `key` (`pro`, `team-5-pack`: up to 64 lowercase letters, digits, `-` and `_`, starting with a letter or digit) is the tier's identity across releases and pricing revisions; `name` is display text and may change freely. A different key is a different tier: holders and subscribers stay with the old key.
- Always write `key`. A tier without one takes the key its name was last published with, else a key derived from the name, so a rename without a key becomes a new tier.
- A published tier's key is fixed: `yard projects tiers edit` and the dashboard refuse to change it. Add a tier under the new key and `rm` the old one instead.
- Tier ids change with every pricing revision, so code matches on keys: `X-Yard-Tier-Key` in services (yard-auth.md), `tier_key` on webhooks and in `window.yard.ownership()`, `?tier=` at checkout (id or key, never a name).
- Products name the tiers they require by key, and a required tier cannot be removed until the requirement is dropped (products.md).

## Seats and volume

- `single`: one license per purchase. `fixed_pack`: `seat_count` (2 or more) licenses, always bought as one pack with no quantity choice. `per_seat`: the buyer picks a quantity within `min_seats` and `max_seats`, one license per seat.
- `volume_brackets` (`per_seat` only): contiguous `{ min_quantity, max_quantity | null, discount_percent 1-99 }` ranges; the first starts at or above `min_seats`.
- Seat-based pricing needs `seat_based_pricing` on the team.

## Trials and gifts

- Trials are per tier (`free_trial`), never per project (a project-level field is rejected). A trial starts only when the project is public and in `early_access` or `published`; in a sandbox, at any stage. `requires_card` on a subscription tier collects a card and converts when the trial ends. A trial, or a subscription still in one, never counts as holding the tier, so it allows no product purchase. Adjust a running trial with `yard transactions trial` (sales.md).
- Gifts (`gift_enabled`): one-time tiers only, not plan-gated. The buyer enters a recipient email at checkout, the key is minted on activation, and a gift unactivated after 90 days is refunded. In a draft project's sandbox the recipient must be a team member.
- Buyers find their projects, keys and subscriptions in the Library at yard.sh/library: https://docs.yard.sh/v1/platform/library/overview.md.

## yard projects tiers

Tiers are addressed by **key** only. Without `--release` the subcommands edit your open draft (or a new draft from the newest published release), which goes live with `yard releases publish`; `--release <tag|id>` edits a published release, live at once (subscribers on a changed subscription price get 30 days' notice). Inside a working copy a save also updates the local settings.json, merged by key.

- `yard projects tiers <slug> [--release <tag|id>] --json`: the release's tiers as settings.json entries (`jq -r '.[].key'`).
- `add <slug> --spec <file|->`: one tier entry; unknown fields are rejected. `is_default: true` demotes the current default.
- `edit <slug> <key> --spec <file|->`: partial; present fields replace. `echo '{"free_trial":{"enabled":true,"days":14}}' | yard projects tiers edit my-app base --spec -`
- `rm <slug> <key> [--yes] [--promote-default]`: refuses to remove the last tier, the default without `--promote-default`, or a tier a product requires. Existing purchases are unaffected.
