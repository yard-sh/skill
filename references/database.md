# Database (Yard DB)

Every project and every sandbox has its own SQLite-compatible database, shared by all services there that set `database_access: true` (services.md), as `env.DB`. Docs: https://docs.yard.sh/v1/platform/database/overview.md (migrations, failures, the SQL dialect, lifecycle, the dashboard console).

```js
const stmt = env.DB.prepare("SELECT * FROM notes WHERE user_id = ?1").bind(user);
const { results } = await stmt.all();   // rows
const row = await stmt.first();         // first row, or null
const { meta } = await env.DB.prepare("DELETE FROM notes WHERE id = ?1").bind(id).run(); // meta.changes, meta.last_row_id
await env.DB.batch([stmt1, stmt2]);     // the only atomic multi-statement write
```

- SQLite SQL, `RETURNING` and `INSERT ... ON CONFLICT DO UPDATE` included; the dialect notes are on the docs page. Store times as integer milliseconds (`Date.now()`). Tables starting with `_` are Yard's (`_yard_migrations`, `_cf_KV`): leave them alone.
- Inspect with `yard db query "<sql>" [--file <path>] [--sandbox <name>] --json` (up to 10,000 bytes of SQL, no bind parameters, results truncated at 1000 rows). Automation uses https://docs.yard.sh/v1/api/services/query-database.md with `db:query`.

## Migrations

- Flat numbered `.sql` files in `.yard/migrations/` (`0001_init.sql`, `0002_add_column.sql`; `migrations.dir` in settings.json), one ordered set for the whole project, never inside a service directory. Names of letters, digits and `._-`, no subdirectories; counts and sizes come from the plan (`migrations_max_*`).
- A deploy applies pending files in filename order before the new services go live; the first migration creates the database, services or not. Each file runs once per database, recorded by filename in `_yard_migrations`, so never edit an applied file and never rename one (a renamed file runs again): add a new file.
- Files are not transactional. A failing statement aborts the deploy (the previous version keeps serving), the statements before it stay applied and the file is not recorded, so the next deploy re-runs it from the top. Make statements re-runnable (`CREATE TABLE IF NOT EXISTS`), keep one statement per file, or record a hand-fixed file with `yard db migrations mark-applied <file> [--sandbox <name>]`.
- `yard db migrations list [--sandbox <name>] --json`: applied migrations merged with local files still pending (`database: false` when none exists yet). `yard service init` numbers its scaffold migration after the highest one present.
