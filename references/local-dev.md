# Local development: yard dev

`yard dev` runs the project on the machine the way Yard hosts it: the landing page, every service under its mount, the `X-Yard-*` headers from a chosen persona, secrets, rooms, and a local database with the migrations applied. Docs: https://docs.yard.sh/v1/platform/services/local-development.md (first run, personas, panel, differences, troubleshooting). Iterate here, then `yard push`; real commerce (a purchase, a trial, a license key) needs a sandbox.

```sh
yard dev                        # serves http://localhost:9875/<slug>/ ; walks up to .yard/settings.json
yard dev --as user:pro --json   # default persona; one JSON event per line for scripts and agents
```

Flags: `--port <n>` (9875), `--dir`, `--project`, `--as <persona>`, `--root` (serve at `/` like a custom domain), `--open`, `--secrets-file <path>`, `--reset-db`, `--reset-rooms`, `--allow-local-egress`, `--no-panel`, `--offline` (no Yard lookups), `--json`.

- Needs `.yard/settings.json` with a service or a landing page. Each run downloads the current local runtime (about 40 MiB, checksum-verified) into `~/.yard/runtime/`; Linux needs glibc 2.35+, macOS 13.5+. One `yard dev` per working copy (a second refuses, naming the pid); the port is bound first, so a busy port fails at once (`--port`).
- No login needed. Logged in, `window.yard.project` is the live project (pricing and products from the published release, or from settings.json until something is published: `snapshot_source` `remote_unpublished`); offline or logged out it is built from settings.json without `title`, `tagline`, `description` or images. Nothing else leaves the machine except checkout and trial links and `GET /api/v1/*` on the local origin, which is proxied to Yard.
- `--json` events: `ready` (`slug`, `runtime_version`, `landing_page`, `services[]` with `url`, `access`, `database` and `rooms` (`[]` when none), `panel`, `persona`, `database`, `migrations_applied`, `migrations_pending`, `secrets_loaded`, `missing_secrets`, `snapshot_source`, `warnings`), then `restart`, `validation_error` (`errors[]`), `migrations`, `log` (`service`, `level`, `message`), `request`, `runtime_error`, `runtime_download`, `notice`, `stopped`. A fatal refusal (lock, port, settings, runtime) prints `Error: ...` on stderr and exits 1 with no event: run it in the background and watch the exit as well as `ready`.

## Personas

There is no real Yard Auth locally: a persona decides the `X-Yard-*` headers, `__yard/auth/me`, `__yard/products` and `window.yard.ownership()`.

| Persona | Entitlement | Tier |
| --- | --- | --- |
| `anonymous` | none (no headers) | |
| `signed-in` | `none` | |
| `trial` (when a tier offers one) | `trial` | the first tier with a trial |
| `user:<tier key>` (one per tier) | `active` | that tier |
| `buyer:<product key>` (one per product) | `active` | the first tier that lets them buy it; holds that product |
| `team` (a team member) | `team` | |

Each signed-in persona has a stable UUID `X-Yard-User-Id` (the same in every project and run; read it from `__yard/auth/me` or `/__yard/dev/api/state`, never hard-code it) and an email like `user-pro@example.com`. Personas come from the project's tiers and products (the published release when logged in, else settings.json). Pick one with `--as`, with the cookie (`curl -H 'Cookie: yard_dev_identity=user:pro' http://localhost:9875/<slug>/api/notes`), with `POST /__yard/dev/api/persona {"persona":"team","default":true}`, or at the picker `/<slug>/__yard/auth/login` (`__yard/auth/logout` clears it). A persona the project no longer has serves as `anonymous` with a warning. Gating, header stripping, `__yard/auth/guest` and the cross-origin rules behave as hosted (yard-auth.md).

## Database, secrets, rooms, panel

- `env.DB` is `.yard/dev/data.sqlite`; `.yard/migrations/*.sql` are applied in order and recorded as hosted (a failing file leaves its earlier statements applied: fix it to re-run from the top, or `--reset-db`). New files apply on save. `yard db query` still targets the hosted database; query the local one from the panel or with `POST /__yard/dev/api/db/query {"sql":"...","params":[]}`.
- Secrets: `.yard/dev/secrets.env` (`KEY=value`, the hosted grammar; never pushed). A startup warning names remote secrets with no local value.
- Rooms run with the hosted contract; storage under `.yard/dev/rooms/<service>/` survives restarts (`--reset-rooms` clears it; `--reset-db` leaves rooms alone). Live connections end on every restart, which is every save, so reconnect logic is exercised early.
- The control panel `http://localhost:9875/__yard/dev/` sits over loopback-only JSON endpoints under `/__yard/dev/api/`: `GET state` (URLs, services, personas, runtime, migrations, secret names), `GET requests?since=&limit=` and `requests/stream` (SSE), `GET logs?service=&since=` and `logs/stream`, `POST persona`, `POST db/query`, `POST migrations/apply`, `POST db/reset`, `POST rooms/reset` (POSTs need `Content-Type: application/json`). An automated check: start `yard dev --json`, wait for `ready`, curl the URLs with persona cookies, read `requests` and `logs`, stop the process.

## Differences from hosted

Every service directory, the page directory, migrations, settings.json and the secrets file are watched; any save (assets included) re-validates with push's rules, restarts the runtime in under a second and reloads open landing page tabs. The 50 ms CPU budget is not enforced; egress to private networks is blocked unless `--allow-local-egress`; the Cache API is unavailable; `request.url` is `http://localhost:<port>/...`; there is no consent screen, sandbox, draft gating or `yard service logs` (use the panel). Checkout and trial links go to Yard (test buying in a sandbox); a product checkout's `returnTo` to `http://localhost:<port>` is refused unless it is a Yard Auth redirect URI origin; persona purchases fire no webhooks and are never refunded.
