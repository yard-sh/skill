# Sandboxes

A sandbox is an optional private copy of the project at `https://<team>.yard.sh/<slug>/@<name>/`: its own page, services, database, secrets, rooms and simulated commerce, serving whatever release it points at. Docs: https://docs.yard.sh/v1/platform/sandboxes/overview.md. A project starts with none; `max_sandboxes` caps them (Basic 1, Pro 10; `403 sandbox_limit_reached`). Code never goes to a sandbox: it goes to a release, and the sandbox is pointed at the release.

## What serves where

| State | Set by | Serves |
| --- | --- | --- |
| following a channel | `sandbox channel <name>`, `unpin` | the channel's newest published release; new releases take over |
| rolled back | `sandbox rollback <release>` | that release until the next one lands in the channel |
| pinned | `sandbox pin [release]`, `promote` | that release until `unpin` |

Pin > rollback > channel. A new sandbox follows nothing and serves nothing until `channel` or `pin`. A rollback is refused (`409`) while pinned. Deploy status after a change: `stale` (a redeploy starts shortly), `updating`, `failed` (the previous version keeps serving; the error is in the bundle, so fix it and push), or up to date.

Try a release before users see it, then ship it. This is the one recipe; publish defaults to Production, which the project follows, so the storefront is pinned first:

```sh
yard sandbox create preview                 # once
yard sandbox pin                            # hold the storefront on what it serves now
yard releases publish v1.4.0                # into Production; the project is pinned, so users see no change
yard sandbox pin v1.4.0 --sandbox preview   # browse .../<slug>/@preview/, yard service logs --sandbox preview
yard sandbox unpin                          # ships v1.4.0 (or yard sandbox rollback <old> if a bad one is already live)
```

Promotion (`yard sandbox promote preview`) pins the project to what the sandbox serves; data and secrets never move.

## Visibility and commerce

- Private (the default) admits only members of the owning team, any role, after sign-in; everyone else gets a Team only page. `yard sandbox visibility public --sandbox <name>` opens it to anyone with the URL, but only while the project itself is public and past draft. Without `--sandbox`, `visibility private` hides the project itself from strangers (the launch stage is separate, projects.md).
- Commerce is simulated: checkouts, trials, subscriptions (renewing on schedule), license keys, coupon redemptions and gifts all happen, no card is charged, no money moves, and amounts are computed as a real sale's would be. The team's books (`yard users`, `yard transactions`, earnings, payouts) never include it; a sandbox's commerce is on that sandbox's dashboard pages. Payout setup is not required.
- A sandbox's license keys validate only when the request names the sandbox (license-keys.md). Consumables bought there are fulfilled there and refunded after 3 days unfulfilled, like real ones.
- `yard sandbox delete <name> [--yes]` removes its files, database, secrets, rooms and all of its commerce at once. Releases belong to the project and survive.

## Commands

`visibility`, `channel`, `rollback`, `pin` and `unpin` act on the project itself unless `--sandbox <name>`; `create <name>` (2-60 letters, digits and hyphens, starting with a letter), `rename <name> <new-name>` (URLs change), `delete <name>` and `promote <from> [--to <sandbox>]` name the sandbox. All take `--project`, `--dir` and `--json`; a release is a tag or UUID.

`yard sandbox list --json`: `{ project: {...}, sandboxes: [...] }`, each with `slug`, `visibility`, `channel`, `pinned_release_id`, `rollback_release_id`, `serving_release`, `releases`, `deploy_status` (`up_to_date`, `stale`, `updating`, `failed`), `deploy_error`, `page_url` and `services[] { name, url }`. `rollback`, `pin`, `unpin` and `promote` answer `{ release_id, version, action, artifacts }` plus `"project": "<slug>"` or `"sandbox": "<name>"`.
