# Projects

Creating, linking and editing a project, and launch stages. Docs: https://docs.yard.sh/v1/platform/projects/overview.md (settings, launch stages), https://docs.yard.sh/v1/platform/getting-started/yard-cli/commands.md (init modes). Pricing: pricing.md. Files: project-files.md. Who may view it: sandboxes.md.

## yard init

Links the current directory to a project and writes `.yard/settings.json`. Run `yard projects --json` first so a retry never creates a duplicate.

| Mode | Invocation | Use |
| --- | --- | --- |
| Spec | `yard init --spec <file\|-> --json` | Create a project from JSON, then pull its draft (settings.json holds the pricing). Refused in a directory already linked to a project. |
| Link | `yard init --project <slug-or-id> --json` | Link an existing project. A fresh directory also pulls what `yard push` would write to (`--no-pull` skips). |
| Interactive | `yard init` | Humans only. Without a terminal it refuses (in a linked directory it runs as `--project <linked slug>`). |

Flags: `--page` / `--no-page` (scaffold a custom landing page, landing-pages.md; the non-interactive modes default to no page), `--link-repo` / `--no-link-repo` (link the GitHub repository of a clone the Yard GitHub App can reach; the default links when it can and otherwise carries on). Refused in the home directory. `yard init --help` prints the spec.

Spec (only `title` and `tiers` are required):

```jsonc
{
  "title": "My Project",            // 1-60 chars, must contain a letter or digit
  "pricing_model": "one_time",      // "one_time" (default) | "subscription"
  "tiers": [{
    "key": "base",                  // stable identity: always set it (pricing.md)
    "name": "Base",                 // display only
    "price_cents": 1900,            // 0 (one-time tiers only) or 300..1000000
    "is_default": true,             // exactly one
    "seat_type": "single",          // "single" | "fixed_pack" (seat_count >= 2) | "per_seat" (min_seats, max_seats, volume_brackets); seat-based is plan-gated
    "yearly_discount_percent": null, // subscription only, 1..100
    "free_trial": { "enabled": false, "days": 14, "requires_card": false }, // per tier; days 7..365, default 7
    "gift_enabled": false           // one-time tiers only
  }],
  "license_key_enabled": false,     // plan-gated (license-keys.md)
  "activations_enabled": false,
  "max_activations": null           // 1..10000
}
```

`--json`: `{ project: { id, slug, title, buy_url, profile_url, created }, settings_file, github_repo_linked, landing_page }`. A plan refusal about tiers (count, trials, seat pricing) is a plain `400` naming the rule, not `upgrade_required`.

## yard projects

- `yard projects --json`: the team's projects; `.[].slug` is what every other command wants (titles are display text). Slugs and UUIDs are interchangeable.
- `yard projects show <slug> --json`: one project with `.id` (the REST API's project id) and the served release's `tiers[]` (key, pricing model, seat type, trial and gift fields). A project that has published nothing shows no tiers; read the draft's with `yard projects tiers`.
- `yard projects edit [slug] --spec <file|->`: a sparse object of project settings: `license_key_enabled`, `activations_enabled`, `max_activations` (license-keys.md), `slug` and `launch_stage`. Unknown fields are rejected, missing ones untouched; `--json` emits `{ project, settings }`. Tiers are not project settings (`yard projects tiers`).

## Launch stages

`draft` -> `early_access` -> `published`, one way, set in the dashboard, with `projects edit --spec '{"launch_stage": "..."}'`, the MCP `update_project` tool or the API. Never advance a stage to test: a draft project already serves its pages and services to its own team, and a sandbox tests commerce. Leaving draft needs no payout setup; selling does (checkout.md). `early_access` may carry a discount on one-time purchases (`early_access_discount_percent`, dashboard only). Visibility is separate: `yard sandbox visibility private` hides the project from strangers at any stage (sandboxes.md).

- Deleting a project archives it: hosting stops, buyers keep their library and their keys keep validating.
- Over the plan's project count: `403 project_limit_reached`.
