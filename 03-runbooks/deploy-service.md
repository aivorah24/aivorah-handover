# Deploy — a microservice (Render)

_Last verified: 2026-10-16._

## How it works

Every service in [repositories.md](../01-overview/repositories.md) is a Node/Express app on Render with **auto-deploy on merge** to its deploy branch (`main` for most; `master` for school-admin, bedtime-stories, daily-habits, world-and-you). Secrets live in Render → service → **Environment**.

There is **no staging backend** — PR previews of the frontend call the live services. A merged service change is live for every user within minutes.

## Steps

1. Branch from the deploy branch, change code, add/adjust tests in `tests/`.
2. `npm test` locally. Run the service (`npm run dev`) and hit the changed route, ideally with the frontend pointed at it (`EXPO_PUBLIC_USE_LOCAL_SERVICES=1 npm run web`).
3. **Backward compatibility check:** the live frontend keeps calling the old contract until its own PR merges. Add fields/routes; don't rename or remove them in the same release. Remove the old shape in a later deploy, after the frontend has shipped.
4. If the change needs a DB change, run the migration first, at least on Dev and Test ([db-migrations.md](db-migrations.md)). Prod migration must land **before** the service deploy that depends on it.
5. If it needs a new env var: add it in Render → Environment **before** merging. Render asks whether to save only or save and redeploy — "save only" is fine here, because the merge will deploy. Add the variable *name* to `.env.example` and the [access inventory](../02-access/access-inventory.md).
6. Open PR → review → merge.
7. Render dashboard → service → **Events**: watch the deploy go to **Live**. Read the **Logs** tab for boot errors.

## Verify

- `curl -s https://<service>.onrender.com/<a health or simple GET route>` returns 2xx. ⚠️ Not every service has a `/health` route — check its `index.js` and note it here once confirmed.
- Use the feature in production for the affected persona.
- Logs show no new errors for ~10 minutes.

## Gotchas

- ⚠️ The **first request after idle can be slow** if a service is on a plan that spins down. Check each service's plan in Render before reading slowness as an outage.
- The auth service moved to a new Render account on 2026-09-18 after the old one was billing-suspended — hence the random URL suffixes (`-g27s` etc.). If you ever **re-create** a service, its URL changes and you must update `src/config/<name>ServerUrl.ts` in the frontend and redeploy it.
- Services that call each other (e.g. students → `LEARNING_PATHS_SERVICE_URL`) hold the other service's URL in env vars — update those too.

## If it goes wrong

→ [rollback.md](rollback.md#backend-service-render)
