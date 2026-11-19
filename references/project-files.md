# Project files: settings.json, push, pull, status

The working directory contract and how it syncs with a release. Docs: https://docs.yard.sh/v1/platform/getting-started/yard-cli/commands.md (sections "Project files" and "When the release changed too"), https://docs.yard.sh/v1/guides/landing-pages/publishing-with-an-ai-agent.md (the whole loop, with push's `--json`).

## Layout

```
<project>/
├── .yard/
│   ├── settings.json
│   ├── migrations/0001_init.sql   # default migrations.dir (database.md)
│   ├── products/gems.png          # default product icon path (products.md)
│   └── landing-page/index.html    # the landing page, a service at "/" (landing-pages.md)
└── api/_service.js                # one directory per service (services.md)
```

`.yard/settings.json` (`"version": 10`; every block is optional):

```json
{
  "version": 10,
  "project_slug": "my-project",
  "ignore_files": ["*.bak", "drafts/**"],
  "services": [
    { "dir": ".yard/landing-page", "name": "landing-page", "url": "/" },
    { "dir": "api", "name": "api", "url": "/api", "access": "authenticated", "database_access": true }
  ],
  "migrations": { "dir": ".yard/migrations" },
  "pricing": { "tiers": [{ "key": "base", "name": "Base", "price_cents": 1900, "is_default": true, "pricing_model": "one_time" }] },
  "products": [{ "key": "gems", "name": "500 Gems", "type": "consumable", "price_cents": 499, "requires": ["base"], "icon": ".yard/products/gems.png" }],
  "downloads": { "buttons": [{ "condition": "ends_with", "value": ".dmg", "label": "Download for Mac" }] }
}
```

- `project_slug`: the project this directory belongs to (a pull keeps yours). `ignore_files`: globs (`**` matches any depth) relative to each service's directory and the migrations directory; dotfiles are always ignored.
- `services[]`: services.md; the entry with `"url": "/"` is the landing page (landing-pages.md). `migrations.dir`: database.md. `downloads.buttons`: releases.md.
- `pricing.tiers`, `products`, `downloads.buttons`: when present, a push makes the release match the list exactly (`[]` removes everything); absent means managed elsewhere (dashboard, `yard projects tiers|products`). Give every tier and product a `key` (pricing.md, products.md).
- The file is parsed non-strictly by the CLI and the server: a misspelled or unknown key, a tier field included, is silently dropped (every `--spec` rejects unknown fields). Re-read the file after editing it.
- A push sends settings.json first, in the same change as the other bundles; one the server refuses (`400` naming the field) lands nothing. The server rewrites each release's copy on dashboard edits, so `yard status` can show a config diff you did not make; `yard pull` merges it by top-level key.
- A file older than version 10 is rewritten as version 10 by the first `yard` command run in the directory, which says so (on stderr with `--json`): its `landing_page` block, or files in `.yard/landing-page/` without one, become the service `landing-page` (or the next free name) at `/`, and every other key stays where it was. A version 10 file with a `landing_page` block is rejected. Retired layouts (versions 4 to 8: a top-level `service` block, per-directory settings files, a `database` key, `objects`, a room `class`) are rejected with an error naming the edit: make it, set `"version": 10`, retry.

## Sync commands

All take `--project`, `--dir`, `--release <tag|id>` and `--json`. Writes go to your open draft or a new draft seeded from the newest published release (`--release` on a published release edits it live). The directory records its last pull or push in `.yard/sync/` (git-ignored), so push and pull are three-way: a file changed only here is pushed, one changed only in the release is left for `pull` (`release_changed`), one changed on both sides stops the command before anything moves (`conflicts`, exit 1); `yard pull --force` takes the release's copies of those files, `yard push --force` replaces them with yours. settings.json merges by top-level key, so only a key changed on both sides conflicts.

- `yard status`: what a push would do, per bundle: `to_upload`, `unchanged`, `remote_only` (the next push removes these unless `--keep`), `release_changed`, `conflicts`; `product_icons.missing` fails a push. `synced: false` (a fresh clone): a push sends every differing file and a pull refuses any differing file except settings.json. It also lists who serves the release and each deploy status.
- `yard push [--keep] [--yes] [--verbose]`: uploads changed files (each service, the landing page among them, migrations, settings.json, product icons) and removes release files missing locally; it asks before removing (`--yes` or `--json` skips; without a terminal it refuses unless `--yes`). Every bundle is validated before anything uploads. It prints a `Review:` URL; nothing serves a draft, so going live is `yard releases publish` (releases.md). The files land as one change (one per 64 MiB of files), so a release that is served deploys once with every file in place; a refused file, named in the error, stops the push with nothing of its change landed. Product icons go after the rest. `--json`: per bundle `{ dir, uploaded, skipped, deleted, remote_only, release_changed }`, `review_url`, `errors`; exit 2 when part of the push landed before a refusal. A push with nothing to send opens no draft. Over the plan's open-draft cap: `403 draft_limit_reached`.
- `yard pull [--force | --discard]`: downloads the release (settings.json, services, migrations, icons; a service directory the release holds is created). Default: the open draft, else the newest published release; an error when neither exists. `--discard` makes every file match the release. `--json`: per bundle `{ destination, written, skipped, removed, local_changed }`.
- `yard ls`: a release's files by bundle plus its downloads, with sizes and hashes. `yard init --page`: adds the landing page service to a linked project and pushes it (landing-pages.md).

Bundle counts and sizes come from the plan: `team_permissions` `service_max_*` and `migrations_max_*` in `yard me --json`. File-type rules are in each bundle's reference file. Everything is checked before any upload.
