# Troubleshooting index

Symptom, what it means, and where the detail is. Branch on `error_code` when there is one; the CLI's own message usually names the fix. Install and sign-in problems: https://docs.yard.sh/v1/platform/getting-started/yard-cli.md.

| Symptom | Meaning and fix |
| --- | --- |
| `426 cli_outdated` on every command | The CLI version does not match the server: `yard update` (cli.md) |
| `401 unauthorized`, "sign in again" | The 90-day session ended, was revoked, or was evicted by an eleventh sign-in: `yard login` (cli.md) |
| `403 no_team`, "A team is required" | The account belongs to no team. Not a plan problem: https://yard.sh/finish, then `yard team` (cli.md) |
| Projects or coupons missing, or under the wrong username | The CLI acts as one team, stored on the account and shared with the dashboard: `yard team`, `yard team use <username>` (cli.md) |
| `403 forbidden` | The member's team role does not allow it; upgrading does not help. `403 not_team_owner`: payouts and billing are owner-only |
| `upgrade_required` | The plan lacks the feature (license keys, activations, API keys, custom page, services, Yard Auth, rooms, coupons): send a spec the plan supports or point to https://yard.sh/pricing. Tier and product plan refusals are plain `400`s naming the rule (pricing.md, products.md) |
| `project_limit_reached`, `draft_limit_reached`, `channel_limit_reached`, `sandbox_limit_reached`, `413 storage_limit_reached` | A plan limit; `yard me --json` `.team_permissions` has the numbers |
| `412 stale_base`, `428 base_required` | A release write without, or with a moved, base: read again and redo (api.md, SKILL.md MCP rules) |
| `yard init` refuses: "no --spec or --project given" | Bare `init` is the interactive wizard; use `--spec -` or `--project <slug>` (projects.md). A hanging `init` has a terminal attached: interrupt it |
| "<repo> is linked to <project>" | A repository syncs to one project at a time: `yard init --project <slug>` for that project, or `yard github link --move` (releases.md) |
| GitHub App not installed | `yard init --json` or `yard github link` print a single-use install link (15 min): hand it to the user and rerun (releases.md) |
| `no tier with the key "Pro"`, `the tier "pro" has been published, so its key is fixed`, `requires "Pro", which is not a valid tier key`, `tier_required_by_product` | Tiers and products are addressed by key, never by name; a published key never changes; drop a product's `requires` before removing its tier (pricing.md, products.md) |
| `product icons declared in ... not found locally`, an icon refused | Fix the `icon` path or `yard pull`; icons are exactly 256x256 PNG, JPEG or WebP, at most 1 MiB (products.md) |
| Push stops with `conflicts` | A file changed here and in the release: `yard pull --force` or `yard push --force` for those files only (project-files.md) |
| settings.json "uses the retired ... key" | An old layout (`service` block, `database`, `objects`, room `class`): make the named edit, set `"version": 9`, retry (project-files.md). A GitHub tag with an old layout fails to sync until the next tag |
| A pushed file is refused by name or extension | Service bundles have an extension allowlist and path grammar; pages allow one subdirectory (services.md, landing-pages.md) |
| `port 9875 is in use`, "another 'yard dev' is already running" | `--port <n>`, or stop the other session (one per working copy) (local-dev.md) |
| `yard dev`: the runtime download did not verify, or fails to start | Rerun; delete `~/.yard/runtime/` if it persists. Linux needs glibc 2.35+, macOS 13.5+; otherwise test in a sandbox with the pin-first recipe (sandboxes.md) |
| `yard dev --json` exits with no event | A fatal refusal prints `Error: ...` on stderr; read it (local-dev.md) |
| A room still runs the old code after publishing | Rooms move to new code seconds to minutes after the service; connections end and clients reconnect; add fallbacks for new message fields (rooms.md) |
| `license_not_found` for a key bought in a sandbox | Validate and deactivate find sandbox keys only when the body names the sandbox (license-keys.md) |
| `410 payouts_missing` at checkout | Selling needs payout setup by the team owner; launching does not (checkout.md) |
| `401 sign_in_required` / `403 purchase_required` / `403 team_only` from a service `fetch` | The edge gated it: send the browser to `__yard/auth/login?return=<path>`; `team_only` means the account is not on the owning team (yard-auth.md) |
| `503 products_unavailable` / `fulfillment_unavailable` | Yard could not look it up right now: retry, do not treat as holding nothing (products.md) |
