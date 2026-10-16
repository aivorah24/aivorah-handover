# Runbooks

_Last verified: 2026-10-16_

Step-by-step procedures for routine and emergency operations. Each one says **when** to use it, **what you need**, the **steps**, and **how to check it worked**.

| Runbook | Use it when |
|---|---|
| [local-dev.md](local-dev.md) | Setting up a laptop / running the app and a service locally |
| [deploy-frontend.md](deploy-frontend.md) | Shipping a frontend change (web via Vercel, Android via EAS) |
| [deploy-service.md](deploy-service.md) | Shipping a change to one of the Render microservices |
| [rollback.md](rollback.md) | A deploy broke something and you need the previous version back **now** |
| [db-migrations.md](db-migrations.md) | Any database schema change (Supabase, via `aivorah-db`) |
| [e2e.md](e2e.md) | Running Playwright E2E locally, on a PR, or the full suite |
| [key-rotation.md](key-rotation.md) | Rotating / re-issuing a secret (handover, leak, provider change) |
| [incident-triage.md](incident-triage.md) | "The app is broken" — first 15 minutes |

## Conventions

- **Never paste secret values** into these files, PRs, chat or tickets. Say *which* variable and *where* it lives.
- A ⚠️ marks something known to be fragile or unverified. Treat it as unknown until you have checked it yourself, then update the runbook.
- The receiver should **dry-run every runbook once** during KT (see [`06-kt/`](../06-kt/)) and fix anything that was wrong.
