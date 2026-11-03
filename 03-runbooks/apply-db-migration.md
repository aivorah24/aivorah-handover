# Runbook: apply a database migration safely

Every schema change is a numbered SQL file in `aivorah-db/supabase/migrations/` (e.g. `0019_notifications_assignment_feedback.sql`). Never change tables in the Supabase dashboard directly.

| Project | Ref | Used by |
|---|---|---|
| Dev | `evlqkcozespsfrnuhevf` | local development |
| **Test** | `xmwzifxlpmgsrxgxsgan` | **all live Render services (real users today)** |
| Prod | `jwlepxmclbkrmfzrumch` | nothing yet (future production) |

The ledger table `public.schema_migrations` records which files ran (version + checksum).

## Known trap: the ledger is incomplete
Some migrations were applied by hand without a ledger row (on Test, roughly `0007`–`0018`; `0019` is on Test but not on Dev). So **`node scripts/migrate.js test` would try to re-run old files** and can fail or duplicate data. Until the ledger is repaired, apply **one file at a time**, as below.

## Setup (once)
```bash
cd aivorah-db
npm ci
cp .env.example .env   # set SUPABASE_ACCESS_TOKEN (supabase.com/dashboard/account/tokens), DEV_REF, TEST_REF, PROD_REF
```

## Steps for a new migration
1. **Write** `supabase/migrations/00NN_short_description.sql` (next number). Make it safe to re-run where possible (`if not exists`, `on conflict do nothing`).
2. **Check the ledger** on the target project: Supabase → SQL Editor:
   ```sql
   select version from public.schema_migrations order by version;
   ```
3. **Apply to Dev first:** SQL Editor on Dev → paste the file → Run. Then record it:
   ```sql
   insert into public.schema_migrations (version, checksum)
   values ('00NN_short_description', '<sha256 of the file>')
   on conflict (version) do nothing;
   ```
   Get the checksum with: `node -e "const c=require('crypto'),f=require('fs');console.log(c.createHash('sha256').update(f.readFileSync(process.argv[1],'utf8').replace(/\r\n/g,'\n')).digest('hex'))" supabase/migrations/00NN_short_description.sql`
4. **Test** the feature against Dev.
5. **Apply to Test** the same way (step 3 on the Test project). Test is live: do it before deploying the service that needs it.
6. **Run the repo tests:** `npm test`. Open a PR with the file, merge.
7. **Deploy the service** that uses the change ([deploy-backend-service.md](deploy-backend-service.md)).

`migrate.js` is safe to use **only** on a project whose ledger lists every file already applied.

## Undo
There are no automatic down-migrations. Write a new migration that reverses the change (e.g. drop the new column) and apply it the same way. Before risky changes, take a backup: Supabase → Database → Backups (paid plans), or export the affected tables.

## Gotchas
- **Notification types:** a new type must be added to the `notifications_type_check` constraint first (like `0019`), or inserts fail silently in the producing service.
- **Migrations are immutable** once applied anywhere. Never edit an applied file; add a new one.
- **Prod** is unused today; apply there only as part of the production build-out (`migrate.js prod --confirm-prod`).
