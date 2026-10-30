# Releases and channels

A release is a project-wide snapshot: landing page, pricing, products, download buttons, services, migrations and download files. Docs: https://docs.yard.sh/v1/platform/projects/releases.md (channels, promoting, archiving, rolling back, pinning, beta channels, files), https://docs.yard.sh/v1/guides/releases/sync-github-releases.md (GitHub App and sync), https://docs.yard.sh/v1/platform/projects/storage.md (storage limit and auto-archive).

## How releases serve

- Every release starts as a draft; `yard push` writes into it, and the plan caps open drafts (`403 draft_limit_reached`). Nothing serves a draft. Publishing stamps a tag (unique among non-archived releases, `409` on reuse) and puts the release in one channel; it stays editable, and an edit to a served release is live on save.
- The project and each sandbox follow one channel and serve its most recently published release. The project follows the protected, always-public `Production` channel; a new sandbox follows none. Precedence: pinned > rolled back > following. A rollback lasts until the next release lands in the channel; a pin lasts until `unpin` (sandboxes.md). So `yard releases publish <tag>` with the default channel is the go-live step.
- Other channels: `yard channels create <name>` (private and empty; `max_channels` per project with Production included, `403 channel_limit_reached`; Basic has Production only), `update <name> --name <new> | --visibility public|private` (a private channel is invisible to buyers' license keys and to `releases:read` keys), `delete <name> [--move-to <channel>] [--yes]` (moves its releases and followers; a run without `--yes` previews what changes), `list --json` (`channels[]` with `latest_version`, `sandboxes`, `project_follows`).
- Archiving takes a release out of its channel, frees its tag and deletes its download files; refused (`409`) while the project or a sandbox is pinned or rolled back to it. Unarchive returns it to the channel it left (or the one that channel's releases moved to), without its downloads.
- Download buttons (`downloads.buttons` in settings.json, project-files.md): up to 10 rules `{ condition: contains|starts_with|ends_with|has_extension, value, label }` matched against the release's files: https://docs.yard.sh/v1/platform/library/download-buttons.md. Files count toward the plan's storage (`413 storage_limit_reached` on publish).

## Commands

Writes target your open draft (or a new draft from the newest published release) unless `--release <tag|id>`; all take `--project`, `--dir` and `--json`. Each follower whose release changed is reported as `my-project: v1.4.0 -> v1.3.2`; in JSON every `deployed[]` entry names its target as `"project": "<slug>"` or `"sandbox": "<name>"`, and `serving[]` carries `{ from, to }`.

- `yard releases publish [tag] [--file <path>]... [--name] [--notes | --notes-file] [--channel <name>] [--spec <file|->]`: uploads the files into the draft (up to 5 GiB each), then publishes it. With `--spec` (`{ project?, tag_name, release_name?, release_notes?, channel?, files? }`) the tag argument and those flags are refused. Without a terminal, the tag and flags are the whole request. A partial upload still publishes without the failed files and exits non-zero; when every file fails nothing is published. `--json`: `{ release: { id, version }, deployed, files, uploaded, failed }`.
- `yard releases promote <tag|id> --to <channel>`: moves a published release into another channel (nothing is copied); followers of the new channel serve it only if it is the newest there.
- `yard releases archive <tag|id> [--yes]` (asks first, naming each download file it deletes), `unarchive <tag|id> [--to <channel>]`.
- `yard releases list [--channel <name> | --drafts | --archived] [--page] [--limit]` (`.items[]`), `show <tag|id>` (files with size, SHA-256 and download count).

## GitHub

- Install the Yard GitHub App once per team: https://dash.yard.sh/integrations, or interactive `yard init` offers it. `yard init --json` and a failing `yard github link` print a single-use install link that expires in 15 minutes: hand it to the user and rerun. A GitHub account or org connects to one team at a time.
- `yard github link [owner/repo] [--channel <name>] [--move]` (a repo links to one project at a time; `--move` takes it from the other project), `unlink`, `list --json` (`links[]`).
- Publishing a GitHub release creates a published Yard release in the sync channel (default `Production`, so it ships): tag, title, notes and assets. GitHub drafts and pre-releases do not sync, and a private project does not sync. Editing the GitHub release re-syncs it; deleting it archives the Yard release.
- With `.yard/settings.json` at the tag, each declared section syncs too: services (each with `_service.js`), migrations, landing page, `pricing.tiers`, `products` and `downloads.buttons`, each matched exactly; an absent section is left as it was. A section that fails validation records a sync error and the release keeps what it had. Tags are immutable, so a settings fix lands with the next tag (or the dashboard's Re-sync).
- A synced release edited on Yard (dashboard, CLI, agent or API) is skipped by later syncs, new assets included, with a warning in its Sync Logs; Release Settings > Re-sync overwrites it from GitHub again.
