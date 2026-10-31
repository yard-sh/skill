# Hosted services

A service is a project's server-side code, deployed in a release and run by Yard under a path of the project's URL, with an optional database (database.md), secrets, rooms (rooms.md) and sign-in through Yard Auth (yard-auth.md). Docs: https://docs.yard.sh/v1/platform/services/overview.md, https://docs.yard.sh/v1/platform/services/bundle.md (layout, settings, static assets, limits), https://docs.yard.sh/v1/platform/services/secrets.md, https://docs.yard.sh/v1/platform/projects/allowances.md. Needs `.team_permissions.service`; any `access` other than `public` also needs `yard_auth`.

## Runtime

There are no ports: never scaffold Express, `app.listen()` or any listening server. The backend is one file, `_service.js`, a single pre-bundled ES module exporting a fetch handler:

```js
export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    if (url.pathname.startsWith("/api/")) return handleAPI(request, env, url);
    return env.ASSETS.fetch(request); // bundle files, falling back to index.html
  },
};
```

- Route by path. No filesystem and no long-lived process: state goes in `env.DB` or a room. About 50 ms of CPU per request (awaiting does not count).
- No Node.js compatibility: `node:*` imports, `Buffer` and `process.env` do not exist. Bundle dependencies (esbuild) and pick ones that run in browsers or Workers.
- `ctx.waitUntil(promise)` keeps work alive after the response (a log write, a webhook call, warming a room); an unawaited promise without it may be cut off.
- Hosted `request.url` is `https://<team>.yard.sh/<service-relative path>`: no slug, no mount. Never build absolute URLs from it (redirects, email links, OAuth callbacks). Outbound requests to private networks and localhost answer `403 Egress blocked by Yard`.
- `env.<SECRET>` holds the secrets, `env.DB` the database when `database_access` is on, `env.<BINDING>` each room type.

## Paths and files

The service sees its paths rooted at `/` whatever its `url`: `/<slug>/s/abc` arrives as `/abc` (`/<slug>/s` redirects to `/<slug>/s/`). For `GET` and `HEAD`, a path naming a bundle file (`/app.js`) or a folder with an `index.html` is served without running `_service.js`; everything else, and every other method, reaches the handler with the path unchanged. There is no extensionless `.html` mapping. Use relative URLs in frontend code (`fetch("api/notes")`, `href="styles.css"`) so the service works under any mount and in sandboxes; `yard service check` lints root-absolute ones.

One directory per service: `_service.js` plus static files. Dot-files, dot-folders and `node_modules/` are never uploaded. Every other file must fit the rules or the whole push fails: extensions `.html .css .js .mjs .json .svg .png .jpg .jpeg .webp .gif .woff .woff2 .ttf .otf .txt .md .ico .map .wasm .webmanifest`, names of letters, digits and `._-`, `__yard/` reserved, at most 7 nested folders and 200-character paths. `.sql` is rejected (migrations are project-level) and `ignore_files` does not apply, so keep build inputs outside the directory. Counts and sizes come from the plan (`service_max_*`); `_service.js` is at most 10 MiB.

## Settings

Each entry of `services` in `.yard/settings.json` is the whole declaration; a change is an edit plus `yard push` (project-files.md):

```json
{ "dir": "api", "name": "api", "url": "/api", "access": "authenticated", "database_access": true, "rooms": [{ "type": "Room", "binding": "ROOMS" }] }
```

- `dir`: the bundle directory (directories never nest). `name`: 1-30 lowercase letters, digits and inner hyphens, unique in the release.
- `url`: default `/<name>`; up to 64 characters of lowercase `/`-separated segments of letters, digits and `._-`, no trailing slash, unique; `/` takes the whole site; the longest match wins; `/__yard` and `/@...` are reserved.
- `access`: `public` (default), `authenticated` (sign-in required), `users` (a tier holder, trial or subscriber; products do not count): a complete paywall with no code, or `team` (members of the owning team only) (yard-auth.md).
- `database_access`: binds `env.DB`; the release's migrations create the database (a service flagged before the first migration is redeployed with `env.DB` once it exists). `rooms`: rooms.md.

Served at `https://<team>.yard.sh/<slug><url>/` (custom domain: `https://<domain><url>/`), in a sandbox at `/<slug>/@<name><url>/`; `yard service open [--sandbox <name>] [--service <name>]` prints the URL. A path no service claims belongs to the landing page. Service traffic counts toward the plan's request and bandwidth allowance.

## Secrets, logs, embedding

- `yard service secrets set KEY=VALUE... | list | rm <name> [--sandbox <name>]`: `env.<NAME>` for every service of the project or of one sandbox (separate sets; a new sandbox starts empty). Write-only; each change redeploys the services there at once (rooms follow within seconds to minutes). Names `^[A-Z][A-Z0-9_]{0,63}$`, not `DB`, `ASSETS` or a room binding; up to 32 per scope, 4 KiB each. There are no plain environment variables. Locally, `yard dev` reads `.yard/dev/secrets.env`.
- `yard service logs [--service <name>] [--sandbox <name>] [--since 2h] [--limit <n>]`: console output, uncaught exceptions and CPU-limit outcomes from the last 24 h (at most 500 lines), visible within about a minute. `yard service check`: validates every bundle offline as a deploy would, lints root-absolute URLs, warns about imports in `_service.js` and about declared room types with no exported class.
- Any site may frame the landing page or a `public` service; an `authenticated`, `users` or `team` service answers with `Content-Security-Policy: frame-ancestors 'self'`. To keep a public service out of frames, answer `frame-ancestors 'none'` from the handler (a directly served file carries no header of yours).

## Workflow

`yard service init <name> [--url <path>] [--realtime]` scaffolds a working bundle (notes API, frontend, a migration; `--realtime` a room-backed chat instead) and records its entry; then `yard dev` (local-dev.md), `yard service check && yard push`, `yard releases publish <tag>` or the sandbox recipe (sandboxes.md), `yard sandbox list` for deploy status. Editing a served release redeploys it (`stale`, `updating`, then up to date). There is no `window.yard` inside a service; that exists only on landing pages.
