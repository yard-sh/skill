# Hosted services

A service is a project's server-side code, deployed in a release and run by Yard under a path of the project's URL, with an optional database (database.md), secrets, rooms (rooms.md) and sign-in through Yard Auth (yard-auth.md). Docs: https://docs.yard.sh/v1/platform/services/overview.md, https://docs.yard.sh/v1/platform/services/bundle.md (layout, settings, static assets, limits), https://docs.yard.sh/v1/platform/services/secrets.md, https://docs.yard.sh/v1/platform/projects/allowances.md. Needs `.team_permissions.service`, the landing page included (landing-pages.md); any `access` other than `public` also needs `yard_auth`. When the plan stops including services, every service stops, the landing page among them, and the team is notified; they run again once it does.

## Runtime

There are no ports: never scaffold Express, `app.listen()` or any listening server. The backend is server code: an entry, `_service.ts` or `_service.js` at the bundle root, exporting a fetch handler, and the modules it imports from a `_server/` folder beside it. Yard builds them into one module when it deploys and never serves them.

```ts
import { handleAPI } from "./_server/api.ts";

export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    if (url.pathname.startsWith("/api/")) return handleAPI(request, env, url);
    return env.ASSETS.fetch(request); // bundle files, falling back to index.html
  },
};
```

- Route by path. No filesystem and no long-lived process: state goes in `env.DB` or a room. About 50 ms of CPU per request (awaiting does not count).
- Server code is the entry plus `_server/` (any depth, only `.ts .js .mjs .json`); `.ts` is allowed nowhere else. One entry: never keep both `_service.ts` and `_service.js`. Import server files by relative path (`./_server/db.ts`, extension optional); importing a public file (`./app.js`) is refused, so shared code goes under `_server/`. The build strips TypeScript types without checking them; a syntax error or broken import fails the deploy with file and line, and the previous deployment keeps serving.
- No packages are installed: a bare import (`"zod"`) is refused. Bundle a dependency into a file under `_server/` (esbuild) and import that, picking ones that run in browsers or Workers. No Node.js compatibility: `node:*` imports, `Buffer` and `process.env` do not exist.
- `ctx.waitUntil(promise)` keeps work alive after the response (a log write, a webhook call, warming a room); an unawaited promise without it may be cut off.
- Hosted `request.url` is `https://<team>.yard.sh/<service-relative path>`: no slug, no mount. Never build absolute URLs from it (redirects, email links, OAuth callbacks). Outbound requests to private networks and localhost answer `403 Egress blocked by Yard`.
- `env.<SECRET>` holds the secrets, `env.DB` the database when `database_access` is on, `env.<BINDING>` each room type.

A service with no entry is static: its files and nothing else, with its `404.html` (or a plain Not Found) for paths it has no file for. A landing page is usually one. Deleting the entry makes a service static and adding one back makes it code again. A static service has no database, secrets, logs or rooms; one that declares rooms is not deployed.

## Paths and files

The service sees its paths rooted at `/` whatever its `url`: `/<slug>/s/abc` arrives as `/abc` (`/<slug>/s` redirects to `/<slug>/s/`). For `GET` and `HEAD`, a path naming a public bundle file (`/app.js`) or a folder with an `index.html` is served without running the server code; everything else, and every other method, reaches the handler with the path unchanged. There is no extensionless `.html` mapping. Use relative URLs in frontend code (`fetch("api/notes")`, `href="styles.css"`) so the service works under any mount and in sandboxes; `yard service check` lints root-absolute ones.

One directory per service: the entry and `_server/` plus static files; `_astro/`, `_next/` and other `_` folders elsewhere are public. Dot-files, dot-folders and `node_modules/` are never uploaded. Every other file must fit the rules or the whole push fails: extensions `.html .css .js .mjs .json .svg .png .jpg .jpeg .webp .gif .woff .woff2 .ttf .otf .txt .md .ico .map .wasm .webmanifest .mp4 .webm .vtt`, and `.ts` in server code, names of letters, digits and `._-`, `__yard/` reserved, at most 7 nested folders and 200-character paths. `.sql` is rejected (migrations are project-level); keep build inputs outside the directory, or leave them out with `ignore_files` (project-files.md). Counts and sizes come from the plan (`service_max_*`), the server code's total included. Video is served with byte ranges, before the server code runs, so players can seek.

## Settings

Each entry of `services` in `.yard/settings.json` is the whole declaration; a change is an edit plus `yard push` (project-files.md):

```json
{ "dir": "api", "name": "api", "url": "/api", "access": "authenticated", "database_access": true, "rooms": [{ "type": "Room", "binding": "ROOMS" }] }
```

- `dir`: the bundle directory (directories never nest). `name`: 1-30 lowercase letters, digits and inner hyphens, unique in the release.
- `url`: default `/<name>`; up to 64 characters of lowercase `/`-separated segments of letters, digits and `._-`, no trailing slash, unique; `/` takes the whole site; the longest match wins; `/__yard` and `/@...` are reserved.
- `access`: `public` (default), `authenticated` (sign-in required, with a yard account or a project account), `users` (people who hold one of the project's tiers, by a purchase, a subscription or a trial; a product bought on its own does not count): a complete paywall with no code, or `team` (members of the owning team only) (yard-auth.md). The service a project accounts sign-in page falls under must be `public`, or the settings file is refused (yard-auth.md).
- `database_access`: binds `env.DB`; the release's migrations create the database (a service flagged before the first migration is redeployed with `env.DB` once it exists). `rooms`: rooms.md.

Served at `https://<team>.yard.sh/<slug><url>/` (custom domain: `https://<domain><url>/`), in a sandbox at `/<slug>/@<name><url>/`; `yard service open [--sandbox <name>] [--service <name>]` prints the URL. The service at `/` is the project's landing page and answers every path no other service claims. Service traffic counts toward the plan's request and bandwidth allowance.

## Secrets, logs, embedding

- `yard service secrets set KEY=VALUE... | list | rm <name> [--sandbox <name>]`: `env.<NAME>` for every service of the project or of one sandbox (separate sets; a new sandbox starts empty). Write-only; each change redeploys the services there at once (rooms follow within seconds to minutes). Names `^[A-Z][A-Z0-9_]{0,63}$`, not `DB`, `ASSETS` or a room binding; up to 32 per scope, 4 KiB each. There are no plain environment variables. Locally, `yard dev` reads `.yard/dev/secrets.env`.
- `yard service logs [--service <name>] [--sandbox <name>] [--since 2h] [--limit <n>]`: console output, uncaught exceptions and CPU-limit outcomes from the last 24 h (at most 500 lines), visible within about a minute. `yard service check`: validates every bundle offline and builds its server code as a deploy would, listing every build problem with file and line; lints root-absolute URLs, warns about declared room types the entry doesn't export, and warns when the sign-in page is under a service that needs sign-in. `yard push` and `yard dev` run the same build first.
- Any site may frame the landing page or a `public` service; an `authenticated`, `users` or `team` service answers with `Content-Security-Policy: frame-ancestors 'self'`, and so does the service that holds the project's sign-in page while project accounts are on, the landing page included when the page is in it. To keep a public service out of frames, answer `frame-ancestors 'none'` from the handler (a directly served file carries no header of yours).

## Workflow

`yard service init <name> [--url <path>] [--realtime]` scaffolds a working bundle (notes API, frontend, a migration; `--realtime` a room-backed chat instead) and records its entry; then `yard dev` (local-dev.md), `yard service check && yard push`, `yard releases publish <tag>` or the sandbox recipe (sandboxes.md), `yard sandbox list` for deploy status. Editing a served release redeploys it (`queued`, `deploying`, then up to date). `window.yard`, `embed.js` and page analytics go on the pages of the service at `/` (landing-pages.md); elsewhere only the project's sign-in page gets `embed.js` and the project data, at its own address and without analytics (yard-auth.md), and other services read project data from the public API. While the project is a draft, and not private, the team sees a draft banner on every service's pages.
