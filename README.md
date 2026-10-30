# Yard Skill

The agent skill for [Yard](https://yard.sh): knowledge files that teach AI coding agents (Claude Code and other skill-aware tools) how to work with the Yard CLI, Yard's MCP server and the REST API.

The skill is deliberately short. Each file holds the facts an agent needs to pick the right command and avoid a trap, and links the page of [Yard's docs](https://docs.yard.sh) that holds the full detail. Every docs page is served as Markdown at its URL plus `.md` (index: https://docs.yard.sh/llms.txt), so the detail is read live rather than copied here and left to go stale.

## Layout

- `SKILL.md`: the entry point loaded into the agent's context: ground rules, the first steps of a new project, the MCP server, and a table of which reference to open for each task.
- `references/`, one file per feature, opened on demand:
  - `cli.md`: install, sign-in, conventions (`--json`, `--spec`, `--yes`, exit codes), account and team, maintenance
  - `projects.md`: creating and linking a project, the `init` spec, project settings, launch stages
  - `project-files.md`: `.yard/settings.json`, push, pull, status and the three-way sync rules
  - `pricing.md`: tiers, tier keys, seats, volume brackets, trials, gifts
  - `coupons.md`: discount codes
  - `products.md`: add-ons, consumables and add-on subscriptions, holdings, fulfillment
  - `checkout.md`: the hosted checkout URL and its parameters, the return flow, the payouts gate
  - `license-keys.md`: license keys, device activations, validation
  - `releases.md`: releases, channels, rollback and pinning, download buttons, GitHub sync
  - `updates.md`: the update server for installed software
  - `sandboxes.md`: sandboxes, what serves where, simulated commerce
  - `landing-pages.md`: custom landing pages and `window.yard`
  - `services.md`: hosted services, the runtime, the bundle, settings, secrets, logs
  - `yard-auth.md`: sign-in headers, gated requests, the `__yard/*` endpoints, guests, external apps
  - `database.md`: Yard DB and migrations
  - `rooms.md`: realtime rooms
  - `local-dev.md`: `yard dev`, personas, the control panel
  - `webhooks.md`: webhook events and verification
  - `api.md`: where the REST reference lives, credentials and scopes, the `sandbox` parameter, conditional writes, errors
  - `sales.md`: users and transactions
  - `templates.md`: project templates and the Create in Yard button
  - `troubleshooting.md`: an index of error codes and refusals, pointing at the file that explains each

## Install

The Yard CLI ships an installer:

```sh
yard skill install
```

This pulls the latest release into `~/.agents/skills/yard/`. To update later:

```sh
yard skill update
```

## Contributing

This repo is consumed by anyone with the skill installed, so changes propagate through the parent Yard repository's submodule pointer. Open an issue or PR here for content fixes; coordinate larger restructures with the team first. Keep each file to highlights plus docs links: anything the docs explain belongs there, not here.

## Releases

A release is cut by Yard's production deploy when it moves its pointer to this repository: it tags the commit it pins and creates the GitHub release, named after the Yard commit that shipped it (`v<date>-<sha>`). `yard skill install` and `yard skill update` read the latest release.
