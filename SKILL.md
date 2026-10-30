---
name: yard
metadata:
  author: yard.sh
  files:
    - SKILL.md
    - references/cli.md
    - references/projects.md
    - references/project-files.md
    - references/pricing.md
    - references/coupons.md
    - references/products.md
    - references/checkout.md
    - references/license-keys.md
    - references/releases.md
    - references/updates.md
    - references/sandboxes.md
    - references/landing-pages.md
    - references/services.md
    - references/yard-auth.md
    - references/database.md
    - references/rooms.md
    - references/local-dev.md
    - references/webhooks.md
    - references/api.md
    - references/sales.md
    - references/templates.md
    - references/troubleshooting.md
description: >-
  Yard sells, licenses, distributes and hosts software. Use whenever the user mentions Yard or the yard CLI:
  projects, pricing, license keys, releases and updates, sandboxes, hosted services and databases, yard dev,
  landing pages, Yard Auth sign-in, realtime rooms, templates, coupons, users or sales, and products sold on top
  of a tier (add-ons, DLC, consumables, in-app purchases, add-on subscriptions, fulfillment).
---

# Yard

Yard lets developers sell software: checkout with Yard as the merchant of record, for pricing tiers and for products sold on top of a tier, license keys and device activations, release downloads with an update server, custom landing pages, and hosted services with a database, user sign-in (Yard Auth) and realtime rooms. Developers manage it with the `yard` CLI or Yard's MCP server; shipped software integrates through the REST API.

Install: `curl -fsSL https://cli.yard.sh | sh` (Windows: `irm https://cli.yard.sh/install.ps1 | iex`). `yard login` prints a nine-digit code and opens `https://yard.sh/login/device`: show the user the code, they authorize, and the command exits signed in (run it in the background if your shell only returns output on exit). `yard <command> --help` is always current.

## Ground rules

- **Read the docs, not memory.** Every page of https://docs.yard.sh is Markdown at its URL plus `.md`; https://docs.yard.sh/llms.txt lists them all with one-line descriptions and names a search endpoint. Each reference file below names the pages that hold the detail: fetch a page before quoting a shape, limit or endpoint the file does not show. The REST API reference is online only (api.md).
- **Manage with the MCP server or the CLI, integrate with the API.** Projects, pricing, products, releases, pages, services, coupons, users and sales are management work: Yard's MCP tools when connected, otherwise the CLI. The REST API is for shipped software (license validation, updates, subscriptions, products, Yard Auth in external apps). Never create a project over HTTP.
- **Non-interactive only.** `--json` (result on stdout; messages and errors on stderr) and `--spec <file|->` (JSON input). Never pipe answers into a prompt. Commands that delete or overwrite take `--yes`; a run without it previews the question. Bare `yard init` is for humans.
- **Teams own everything.** Projects, coupons, API keys and payouts belong to a team; the CLI acts as the active team (`yard team --json` `.active_team`), shared with the dashboard. Projects live at `https://<team>.yard.sh/<slug>/`.
- **Check entitlements, never assume them.** Read `yard me --json` `.team_permissions` before proposing a feature (`.permissions` is the user's own and gates nothing on a project). Plan-dependent: license keys and activations, custom pages, services, Yard Auth, rooms, coupons, seat pricing, trials, tier and product counts, extra sandboxes and channels, API keys and webhooks (Pro). Never quote a plan's features from memory.
- **Read errors by `error_code`.** `upgrade_required` or `*_limit_reached`: the plan lacks the feature or hit a limit; send a spec it supports or point to https://yard.sh/pricing (tier and product plan refusals are plain `400`s naming the rule). `forbidden`: the member's team role; upgrading does not help. `no_team`: send the user to https://yard.sh/finish. `not_team_owner`: payouts and billing are owner-only. `cli_outdated`: run `yard update`.
- **Launch stages are one-way** (`draft` -> `early_access` -> `published`). Never advance one to test: a draft project already serves its pages and services to its own team. Launching needs no payout setup; selling does.
- **Sandboxes are for testing.** A sandbox (`/<slug>/@<name>/`) is a private copy with its own data, secrets and simulated commerce: purchases, trials and license keys work, no money moves. A sandbox key validates only when the request names its sandbox; shipped software never sends `sandbox`.
- **Look before creating.** `yard projects --json` first, so a retry never creates a duplicate.
- **Keys, not names.** Tiers and products are identified by `key` everywhere (settings.json, specs, headers, webhooks, checkout); names are display text.

## New project

Ask first: **guided** (the user drives and you explain each step) or **autopilot** (you drive). On autopilot: get a one-line description and a price idea; propose title, pricing model (one-time or subscription), tiers (each with a stable `key`), seat type, prices and launch plan, flagging anything the team's plan lacks, and wait for the user's OK; then `yard projects --json` and `yard init --spec - --json` (schema in projects.md), or `yard init --project <slug> --json` to link an existing project.

Then cover what the project type needs:

- **Installed software** (desktop app, CLI, binary): publish each version with `yard releases publish` (releases.md) and point the app's updater at the update server with the user's license key (updates.md, license-keys.md); otherwise the buy page has nothing to download.
- **Web app, API or backend Yard runs:** services.md, yard-auth.md, database.md; realtime belongs in rooms.md. Check the `service` permission while planning, not at push time.
- **Custom landing page:** `yard init --page`, edit `.yard/landing-page/`, preview with `yard dev` (landing-pages.md).
- **Extras on top of a tier** (DLC, add-ons, consumables, add-on subscriptions): products.md. Only when the software sells extras; they need a tier and a signed-in buyer who holds it.
- **A template others can start from:** templates.md.

## Hosted services loop

`yard service init <name>` scaffolds a bundle whose backend is one file, `_service.js`, exporting a fetch handler: no ports, no `listen()`, no Express, relative URLs only. Never build auth: Yard Auth gives the service trusted `X-Yard-*` headers, `"access": "users"` is a paywall with no code, and tier features are gated on `X-Yard-Tier-Key`. The loop: `yard dev` (local), `yard push` (into a draft; nothing serves a draft), `yard releases publish <tag>` (go live in `Production`), `yard sandbox list` (deploy status). To try it first, pin the storefront and use a sandbox (sandboxes.md).

## Yard's MCP server

When Yard's MCP tools are connected (`https://mcp.yard.sh/`), manage through them; they need no CLI install or `yard login`. The user approved them for one team, which sees each change as yours. `whoami` names the team and what its plan includes; connecting needs `.team_permissions.agent_access`. `yard dev` still needs the CLI. Overview: https://docs.yard.sh/v1/platform/getting-started/mcp-server.md (or `read_yard_docs` with topic `getting-started/mcp-server`; it reads the Platform pages, and the REST reference is fetched from docs.yard.sh).

- Every change takes an optional `note`, a short summary the team reads beside the change.
- A change to something that exists passes the `base_hash` you read for it (a file's `content_hash`, or one of the release's `versions` from `get_release`). `stale_base` means it changed since: read it again and redo the change.
- Text files go inline with `write_file`. Binary files, text over about 1 MB, product icons and release downloads go through `create_upload_link`: `PUT` the bytes to the link, then `finalize_upload`. A release download is added in the background, so poll `get_upload` until it is `done` or `failed`.

## References

Open only the files whose row the task matches, usually one or two. Each file's first lines name the docs pages to fetch for anything it does not show. All live in `references/`.

| File | Open when the task mentions | Commands |
| --- | --- | --- |
| cli.md | installing, signing in, sessions, `--json`/`--spec`/`--yes`, exit codes, the active team | login, logout, me, team, update, skill, uninstall |
| projects.md | creating or linking a project, the init spec, slug, launch stage, project limits | init, projects, projects show/edit |
| project-files.md | settings.json keys, pushing or pulling files, sync conflicts, what a draft holds | push, pull, status, ls, init --page |
| pricing.md | tiers, prices, one-time vs subscription, seats, trials, gifts | projects tiers |
| coupons.md | discount codes, generating, deactivating, redemptions | coupons |
| products.md | add-ons, DLC, consumables, add-on subscriptions, holdings, fulfillment | projects products |
| checkout.md | buy or trial links, pay.yard.sh parameters, return_to, the payouts gate, price order | |
| license-keys.md | license keys, device activations, validating a key from shipped software | projects edit |
| releases.md | publishing a version, channels, rollback, download buttons, storage, GitHub sync | releases, channels, github |
| updates.md | an app's updater, the update endpoints, beta channels | |
| sandboxes.md | testing a release or purchase privately, preview, pin, promote, deploy status | sandbox |
| landing-pages.md | custom page HTML, data-yard, window.yard, ownership, custom domain | init --page |
| services.md | hosted backend, _service.js, routes, static files, secrets, logs | service |
| yard-auth.md | sign-in, X-Yard-* headers, gating by tier, guests, OIDC in external apps | |
| database.md | SQL, env.DB, migrations, a failed migration | db |
| rooms.md | WebSockets, realtime, presence, multiplayer, room storage, alarms | service init --realtime |
| local-dev.md | running locally, personas, the control panel, dev --json events | dev |
| webhooks.md | being notified of sales or renewals, X-Yard-Signature, event_id | |
| api.md | the REST reference, API keys and scopes, the sandbox parameter, conditional writes, errors | keys |
| sales.md | customers, transactions, refunds, extending a trial | users, transactions |
| templates.md | a template repository, the Create in Yard button | |
| troubleshooting.md | an error code or refusal message you do not recognise | |
