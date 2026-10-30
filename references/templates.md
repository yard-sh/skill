# Project templates

A template is a public GitHub repository anyone can turn into their own Yard project in one click. The official ones are `github.com/yard-sh/yard-*`; `yard-sh/yard-hello-world` is the minimal example (one public service, a custom landing page, a free tier). The API behind it: https://docs.yard.sh/v1/api/projects/create-project-from-repo.md and https://docs.yard.sh/v1/api/projects/preview-project-from-repo.md.

## The Create in Yard button

In the template's README, with the repository URL URL-encoded in `repo`:

```html
<a href="https://dash.yard.sh/projects?action=create&repo=https%3A%2F%2Fgithub.com%2Fowner%2Frepo"><img src="https://yard.sh/create-in-yard.png" width="200" alt="Create in Yard" /></a>
```

It opens the dashboard's create dialog on a preview of the repository (services, landing page, migrations, pricing, products). `repo` must be `https://github.com/<owner>/<repo>` (`.git` or a trailing slash is fine); the same URL can be pasted into the dialog's "Create from GitHub URL" field.

## What the repository needs

- Public, with at least one commit; submodules are not included. `.yard/settings.json` at the root (at most 16 KiB, `"version": 9`); everything else is optional.
- Every declared service directory holds `_service.js`; every product `icon` exists (256x256, at most 1 MiB); a `custom` landing page has files; a `migrations` block has `.sql` files. Files under the default `.yard/landing-page/` and `.yard/migrations/` count without a block. The creating team's plan limits apply (bundle sizes, project count, `service`, `custom_project_pages`, `service_rooms`, `yard_auth` for any non-public service).
- Everything is validated before anything is created, so a broken template creates nothing.

## What happens when someone uses it

- A new project owned by the user's active team, in `draft`, titled after the repository (made unique with ` (2)`), with its slug generated from the title; `project_slug` in the template is ignored (the owner's `yard init --project <slug>` writes theirs).
- Pricing comes from the `pricing` block (validated against the plan; without one the project has no tiers), products from `products` (every `requires` key must be one of the template's tier keys). Give each tier and product a `key`.
- Services, the landing page, migrations and download buttons become release `1.0.0`, published to `Production` and deployed. Secrets are not carried: the owner sets them with `yard service secrets set`. The project is not linked to the template repository.

Keep the README short: the button, the URLs the project serves, the layout, the local loop (`yard dev`) and shipping (`yard push`, `yard releases publish`). Test a template by creating a project from its URL before sharing the button.
