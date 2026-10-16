# Database migrations (`aivorah-db`)

_Last verified: 2026-10-16 against `aivorah-db@c13935a` (latest migration `0018_board_syllabus_nodes.sql`)._

All Aivorah services share **one** Supabase Postgres per environment. The schema belongs to `aivorah-db`, not to any service.

## Rules (from the `aivorah-db` README)

1. No schema changes in the Supabase dashboard — every change is a migration file.
2. Migrations are **immutable** once applied. `migrate.js` checksums every file and refuses to run if an applied one changed.
3. Promotion order is always **Dev → Test → Prod**.
4. Every change goes through a PR.
5. Seeds are **synthetic only** — never copy prod data into dev/test (real students' records).

## Environments

| Env | Ref (per `aivorah-db/.env.example`) | Plan |
|---|---|---|
| Dev | `ernsrofvmszhmfhxeehx` | Free — wipe and reseed freely |
| Test | `vgqrrseziwnljkencqpy` | Free — E2E runs here; schema must mirror prod |
| Prod | `jwlepxmclbkrmfzrumch` | Pro — real users |

> ⚠️ The [system overview](../01-overview/system-overview.md) lists different Dev/Test refs (`evlqkcozespsfrnuhevf`, `xmwzifxlpmgsrxgxsgan`). One of the two is out of date. Check in the Supabase dashboard before running anything, and fix the wrong doc. Tracked in [open work](../04-open-work/README.md).

## Setup

```bash
git clone git@github.com:aivorah24/aivorah-db.git && cd aivorah-db
cp .env.example .env    # add SUPABASE_ACCESS_TOKEN (personal token, Supabase → Account → Access Tokens)
```

Node 18+. No `npm install` needed for the scripts (`npm ci` only for `npm test`).

The access token can create and drop projects. Treat it as a password and revoke it when you leave.

## Steps — making a schema change

1. Branch. Create `supabase/migrations/00NN_short_description.sql` with the **next** number (numbers have gaps; just use max + 1).
2. Write it to be safe on a live DB:
   - additive first (new tables/columns nullable or with defaults);
   - `create ... if not exists` where possible;
   - enable RLS on every new table and add grants/policies explicitly (see `0002_enable_rls_syllabus_tables.sql`).
3. Add a test in `tests/` (they run migrations in PGlite: `npm test`).
4. `node scripts/migrate.js dev` — applies cleanly?
5. `node scripts/compare-schema.js` — Dev now differs from Prod. Confirm the diff is **exactly** your change.
6. `node scripts/migrate.js test`, then run the E2E suite against an app pointed at Test ([e2e.md](e2e.md)).
7. Open a PR; paste the compare output in the description.
8. After merge: `node scripts/migrate.js prod --confirm-prod`.
9. `node scripts/compare-schema.js` — all three must match again.
10. Only now merge/deploy the service code that depends on it ([deploy-service.md](deploy-service.md)).

## Seeds

```bash
node scripts/seed.js dev
node scripts/seed.js test     # dedicated per-test E2E accounts
```

`seed.js` refuses prod. Auth users are **not** seeded — create logins through signup or the Supabase/Firebase dashboards.

## Not tracked in this repo

Edge functions, auth providers, SMTP, redirect URLs and storage file contents are configured per project in the dashboard. Change them by hand in all three projects and note what you changed.

## If a migration fails half-way

The Management API runs each file as one request. A failure usually leaves nothing applied and nothing recorded in `schema_migrations`. Check that table: `select * from public.schema_migrations order by applied_at desc;`. Fix the SQL in a **new** commit (the file was never recorded, so editing it is fine) and run it again. If it **was** recorded, see [rollback.md](rollback.md#database-supabase).
