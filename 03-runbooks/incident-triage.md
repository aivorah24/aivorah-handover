# Incident triage — first 15 minutes

## 1. Scope it (2 min)

- **Who:** one user, one persona, or everyone?
- **What:** whole app down, one screen, or slow?
- **Since when:** match the time against Vercel → Deployments and each Render service's **Events**.

## 2. Find the failing piece (5 min)

Open the app in the browser with DevTools → **Network**, reproduce, and look at the failing request:

| Failing request host | Look at |
|---|---|
| none — page is blank / JS error in Console | Frontend → Vercel deployment |
| `<name>.onrender.com` | That service → Render **Logs** |
| `identitytoolkit.googleapis.com` / Firebase | Firebase status; auth service `/user/sync` |
| `aivorah.ai` | Legacy monolith (`ai-mentor/server`) — ⚠️ hosting unconfirmed |

Map a service URL to its repo with [repositories.md](../01-overview/repositories.md).

In the service logs, look for:

- **Supabase errors** (missing column → a migration hasn't run or was reverted; connection errors → check status.supabase.com).
- **401/403** → Firebase token verification. Is `FIREBASE_SERVICE_ACCOUNT_B64` set and valid?
- **429 / quota** from an LLM provider → free-tier limits; switch or rotate the key ([key-rotation.md](key-rotation.md)).
- **Crash on boot** → missing env var after a deploy.

Also check the platform status pages: Render, Vercel, Supabase, Firebase.

⚠️ **Billing.** Several platforms have been billing-suspended before (the Render auth account, CircleCI, GitHub Actions). If a whole account's services are down at once, check billing first.

## 3. Mitigate (5 min)

- Recent deploy is the cause → [rollback.md](rollback.md).
- Provider outage → nothing to roll back. Tell users and keep watching the status page.
- Bad data / one user → fix through the admin UI if possible. Avoid hand-written SQL on Prod. If you must, do it in a transaction with a second person watching.

## 4. Afterwards

Write `07-meetings/YYYY-MM-DD-incident-<topic>.md`: timeline, cause, fix, and the follow-up test or alert to add.
