# Knowledge transfer (KT)

Goal: by Thu 5 Nov, Maheswar and Teja run, fix and ship Aivorah day to day on their own.

| # | Date | Mode | Theme | Who drives |
|---|---|---|---|---|
| 1 | Thu 29 Oct | Call | Product demo, every persona | Saimohan |
| 2 | Fri 30 Oct | Call | Architecture, repos, data, environments | Saimohan |
| 3 | Sat 31 Oct | Call | Local setup and the ship-a-change workflow | Saimohan, they follow along |
| 4 | Mon 2 Nov | Call | Feature deep dives: AI, notifications, content (Qdrant) | Shared |
| 5 | Tue 3 Nov | Call | Operations: deploys, DB migrations, incidents ([runbooks](../03-runbooks/)) | They drive, Saimohan reviews |
| 6 | Wed 4 Nov | In person, with founder | Access transfer and hands-on fixes | They drive |
| 7 | Thu 5 Nov | In person, with founder | Reverse demo, open work, sign-off | They present |

## Hands-on tasks
Each of them ships at least 2 real changes before sign-off.

| Level | Task | Repo |
|---|---|---|
| 1 | Fix one UI issue from their own walkthrough | `aivorah-frontend` |
| 2 | Teacher promote also moves curriculum at grade boundaries (IB PYP → IB MYP), like School Admin promote | `aivorah-students-service` |
| 3 | Write and apply a small migration `0020_…` to Dev, then Test, with `migrate.js` | `aivorah-db` |
| 4 | Review and merge or close one open PR | any |
| 5 | Fix one open issue end to end with the founder watching | any |

## Reading list
1. `01-overview/system-overview.md`, then `01-overview/architecture/` (repo, service and feature maps, Qdrant)
2. `03-runbooks/` (deploy, migrations, keys, service down)
3. Each repo's `README.md`

## Sign-off
The handover checklist in the [main README](../README.md). Session notes go in `07-meetings/`.
