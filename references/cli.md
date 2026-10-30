# The yard CLI

The tool itself: install, sign-in, the conventions every command follows, account and team, maintenance. Every command, grouped by feature: https://docs.yard.sh/v1/platform/getting-started/yard-cli/commands.md. Install, sign-in, updating: https://docs.yard.sh/v1/platform/getting-started/yard-cli.md. Each feature's own commands are in that feature's reference file (SKILL.md table).

## Install and sign in

- `curl -fsSL https://cli.yard.sh | sh` (Windows: `irm https://cli.yard.sh/install.ps1 | iex`). Never pipe the installer into `sudo sh`: it would install the skill for root.
- `yard login` prints a nine-digit code and opens `https://yard.sh/login/device`. Show the user the code; they paste it there and click Authorize, and the command exits signed in. Works over SSH and in remote workspaces (open the link on any device). Run it in the background if your shell only returns output on exit. The code expires after 15 minutes; cancelling in the browser ends the command at once.
- A session lasts 90 days (the access token renews itself, the session does not). An account keeps at most 10 CLI sessions: an eleventh sign-in evicts the oldest, which then fails with `401 unauthorized`. There is no token environment variable, so the CLI cannot sign in on CI; automate from CI over the REST API with an API key (api.md).
- `yard logout` revokes the session and deletes `~/.yard/config.json`.

## Conventions

- `--json`: the result as JSON on stdout, messages on stderr. Errors are always plain text on stderr (`Error: ...`) with exit 1 and nothing on stdout, except push and pull conflicts (`{ release, conflicts }`), `coupons validate` and a partial publish.
- `--spec <file|->`: the request body as JSON (`-` reads stdin). Unknown fields are rejected. `yard <command> --help` prints the shape and is always current.
- No terminal: nothing can answer a prompt. `init` needs `--spec` or `--project`, `projects edit` needs `--spec`. Commands that delete or overwrite refuse without `--yes` and quote what they would have asked (`push` names only file counts; `yard status` lists the files), so a run without `--yes` is a preview.
- Which project: commands walk up from the cwd to the directory holding `.yard/settings.json`; `--dir <path>` starts elsewhere, `--project <slug-or-id>` names it outright. `--sandbox <name>` targets a sandbox where it applies (default: the project itself). `--release <tag|id>` picks a release; without it, writes go to your open draft (or a new draft from the newest published release) and reads use the draft, else the newest published release.
- Lists refuse an unknown `--sort` or `--direction`, `--page` below 1 and `--limit` outside 1-100; `--json` lists put rows in `.items`.
- Exit codes: `0` success, `1` any error, `2` partial push (some files failed), `130` Ctrl-C and `143` SIGTERM (in-flight work is cancelled; a push keeps what it already sent).
- `cli_outdated` (`426`): the CLI version does not match the server, and every command refuses until `yard update`. Try it first on an unexplained refusal.

## Account and team

- `yard me --json`: `.team` (`id`, `username`, `role`; `null` with no team), `.team_permissions` (the active team's entitlement: booleans carry `granted`, limits carry `limit` or `unlimited`; `null` with no team) and `.permissions` (the user's own, account-level only). Check `.team_permissions.<name>.granted` before promising a feature; `plan` is a display label.
- `yard team [--json]`: the team the CLI acts as. `.active_team.username` is the segment projects live under (`https://<username>.yard.sh/<slug>`); it is a team username, not the user's. `yard team use <username>` switches. The choice is stored on the account and shared with the dashboard, so re-check it rather than trusting an earlier answer. `owner` and `admin` run every project command; payouts and billing are owner-only (`403 not_team_owner`).
- No team: every team command fails with `403 no_team`. Not a plan problem: send the user to https://yard.sh/finish, then `yard team`.

## Maintenance

- `yard update [--check]`: installs the latest CLI (verified against its published SHA-256) and updates the skill it installed. `yard --version` prints the version.
- `yard skill install | update [--check] | uninstall [--yes]`: this skill, in `~/.agents/skills/yard/` plus `~/.claude/skills/yard/` and `~/.codex/skills/yard/` when those directories exist.
- `yard uninstall [--yes]`: ends the session, removes `~/.yard/`, the skill copies and the binary. Refuses without a terminal unless `--yes`.
