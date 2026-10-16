# Rollback

Goal: get users back to the last working version **first**, investigate **after**. Every rollback below is followed by a revert PR so `master`/`main` matches what is actually running.

## Decide what to roll back

| Symptom | Likely culprit | Go to |
|---|---|---|
| Blank page, JS error, layout broken, everything else fine | Frontend deploy | [Frontend](#frontend-vercel) |
| One feature returns errors / spinners forever; network tab shows 4xx/5xx from one `*.onrender.com` host | That service | [Backend service](#backend-service-render) |
| Errors mention a missing column/table, or many services fail at once right after a migration | Database | [Database](#database-supabase) |

Check Vercel → Deployments and each Render service's **Events** tab: what changed in the last hour is usually the cause.

## Frontend (Vercel)

1. Vercel → project `aivorah-frontend` → **Deployments**.
2. Find the last production deployment that worked (before the bad one).
3. `⋯` → **Instant Rollback** (or **Promote to Production**). Takes seconds; no rebuild.
4. Verify: hard-refresh production and check the broken screen.
5. Open a PR that reverts the bad commit on `master` (`git revert <sha>`). ⚠️ Until that merges, the next merge to `master` will re-deploy the broken code. Vercel may also pause auto-promotion after an instant rollback — check the project banner and re-enable it once the revert is merged.

## Backend service (Render)

1. Render → the service → **Events**.
2. Find the last good deploy → **Rollback** (redeploys that build).
3. Watch it go **Live**, check **Logs**, re-test the feature.
4. Open a revert PR on the service's deploy branch. Merging it redeploys the same code the rollback restored.
5. If the bad deploy also changed **env vars**, rolling back the code does not roll them back — restore the old values in Environment by hand.

## Database (Supabase)

There is **no automatic down-migration**. `migrate.js` only applies forward, and migrations are immutable.

1. Do **not** edit or delete the applied migration file.
2. Prefer rolling back the **service** that uses the new schema if the schema change is additive (new table/column) — additive changes rarely break old code.
3. If the schema itself must be undone: write a new migration `00NN_revert_<thing>.sql` that reverses it, and apply it Dev → Test → Prod as in [db-migrations.md](db-migrations.md). In an emergency you may go to Prod after Dev only, and run Test afterwards — write down that you did.
4. Data loss: Supabase Prod is on the **Pro** plan, which includes daily backups (and point-in-time recovery only if that add-on is enabled — ⚠️ unverified). Restoring a backup replaces the **whole database** and loses everything written since. Only do it with the product owner's agreement.

## After any rollback

- Post in the team channel: what broke, what you rolled back, when.
- Write a short note in `07-meetings/` or the next `status/` file. Add a regression test before re-shipping the fix.
