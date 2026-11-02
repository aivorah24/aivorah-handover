# Architecture maps

Moved here from `aivorah-frontend/docs/`. Items marked "unconfirmed" or "sampled" were not fully verified.

| File | What it covers |
|---|---|
| `REPO_MAP.md` | Every repo: purpose, tech, where it deploys |
| `FEATURE_SERVICE_MAP.md` | Each frontend feature → backend service, DB, external APIs |
| `SERVICE_MAP.md` | Each backend service: routes, dependencies, who calls it |
| `SYSTEM_ARCHITECTURE.md` | High-level diagram, current vs legacy architecture |
| `DEPLOYMENT_MAP.md` | What runs where (Vercel, Render, Supabase) |
| `DATA_FLOW.md` | Frontend → service and legacy → monolith request flows |
| `API_MAP.md` | API documentation status |
| `SYSTEM_ARCHITECTURE_DETAILED.md` | Detailed architecture diagram + hybrid monolith/microservice state (2026-09-21) |
| `SERVICE_DEPENDENCY_MAP.md` | Backend-to-backend calls between services |
| `LEGACY_MIGRATION_AUDIT.md` | What still depended on the monolith (2026-09-21) |
| `notifications/` | Notification system audit and V2 design / PR plan (2026-09-24) |
| `diagrams/` | Mermaid sources (`.mmd`) for the architecture diagrams |
