# Sales: users and transactions

Read-only views of who bought and what they paid, plus trial adjustments. Docs: https://docs.yard.sh/v1/platform/getting-started/yard-cli/users.md, https://docs.yard.sh/v1/platform/getting-started/yard-cli/transactions.md. Both read the project itself only: sandbox (simulated) sales and users never appear, and there is no `--sandbox` flag.

- `yard users list [--project <slug>] [--sort ...] [--json]`: users with at least one completed, unrefunded purchase; `yard users show <user-id-or-email>` for one. Ids look like `user_deadbeef` (truncated); two users sharing one is a `409`, so pass the email. Money is pre-formatted text (`"$87.00"`); use transactions for arithmetic.
- `yard transactions list [--project] [--start] [--end] [--search] [--type] [--status] [--kind tier|product] [--json]`: `.items[]` with `amount_cents` (subtotal), `total_cents` (with tax) and `team_earnings_cents`. `.stats` covers the date range and `--project`; `--search`, `--type`, `--status` and `--kind` narrow the rows only. `yard transactions show <order-id>` (`order_xxxxxxxx` or the UUID).
- `yard transactions trial <order-id> --add-days N` (negative shortens; up to 365 either way): days are added to the current expiry, not today, so an expired trial can come back. A card-required trial's first charge moves with it, and ending it now charges the card, so the command asks first (`--yes` accepts; required without a terminal). The user is notified by email and in-app.
- Refunds: teams cannot issue them from the CLI or the dashboard; Yard support (support@yard.sh) does.
