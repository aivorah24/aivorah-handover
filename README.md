# Aivorah — Handover

Single place for the Aivorah engineering handover: system overview, access, runbooks, open work, KT sessions and meeting notes.

| | |
|---|---|
| **Handing over** | Saimohan R |
| **Receiving** | _TBD_ |
| **Started** | 2026-10-14 |
| **Status** | 🟡 In progress — overview, access, runbooks, open work documented; KT not started |

> **No secrets in this repo.** Only *where* a secret lives and *who* can grant it. See [02-access](02-access/access-inventory.md).

## Contents

| Folder | What's in it | Status |
|---|---|---|
| [01-overview](01-overview/) | Architecture, repos, environments, request flow | ✅ v1 |
| [02-access](02-access/) | Every account/platform, owner, what must be transferred | ✅ v1 |
| [03-runbooks](03-runbooks/) | Local dev, deploy, rollback, migrations, E2E, key rotation, incident triage | ✅ v1 — needs receiver dry-run |
| [04-open-work](04-open-work/) | Open PRs, known bugs, tech debt, risks | ✅ v1 — service repos not yet listed |
| [05-decisions](05-decisions/) | ADRs — one file per significant decision | ⏳ todo |
| [06-kt](06-kt/) | KT plan, session notes + recordings, sign-off | ⏳ todo |
| [07-meetings](07-meetings/) | `YYYY-MM-DD-topic.md` meeting notes | template ready |
| [status](status/) | Weekly handover status updates (newest = latest) | ✅ 2026-10-16 |

## Handover checklist

- [x] System overview written
- [x] Access inventory written
- [ ] Access transferred / granted to receiver (per platform, see access inventory)
- [ ] Monolith repo moved from personal GitHub account to `aivorah24` org
- [x] Runbooks written
- [ ] Runbooks dry-run by receiver
- [x] Open work list written
- [ ] Supabase Dev/Test ref discrepancy resolved (see 04-open-work)
- [ ] Open work list reviewed together
- [ ] KT sessions completed (see `06-kt/`)
- [ ] Receiver deploys one change end-to-end unaided
- [ ] Sign-off by receiver and manager

## How updates are shared

Each update is a new file in [`status/`](status/) named `YYYY-MM-DD.md` (what shipped, in progress, risks/blockers, next). Meeting notes go in [`07-meetings/`](07-meetings/) using the [template](07-meetings/_template.md).
