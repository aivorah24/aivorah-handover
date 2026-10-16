# Open work

_Last updated: 2026-10-16. Re-check PR lists before each KT session — they go stale fast._

Legend for "Suggested": **merge** · **finish** (needs work, then merge) · **close** (stale / superseded) · **decide** (owner to call it).

## Open pull requests

### `aivorah-frontend`

| PR | Title | Branch | Last update | Suggested |
|---|---|---|---|---|
| [#123](https://github.com/aivorah24/aivorah-frontend/pull/123) | style(profile): pencil icon next to name replaces Edit display name row | `style/profile-name-edit-icon` | 2026-10-13 | merge after preview check |
| [#121](https://github.com/aivorah24/aivorah-frontend/pull/121) | Fix student dashboard not reflecting Learning Path progress | `fix/student-dashboard-progress-sync` | 2026-10-13 | finish — overlaps #110, review together |
| [#110](https://github.com/aivorah24/aivorah-frontend/pull/110) | fix(student): correct learning progress, mastery, and quiz tracking | `fix/student-learning-progress-b001-b005` | 2026-10-11 | decide — overlaps #121 |
| [#105](https://github.com/aivorah24/aivorah-frontend/pull/105) | feat(admin): Platform Overview — in-app, correct DB, real auth | `feat/super-admin-overview` | 2026-10-07 | decide — probably superseded by merged #107 (Aivorah Admin persona) |
| [#46](https://github.com/aivorah24/aivorah-frontend/pull/46) | fix(ci): base64-encode E2E_ENV_FILE to stop CircleCI truncation | `fix/e2e-env-file-base64` | 2026-09-08 | close — CI no longer runs Playwright here (E2E moved to `aivorah-e2e-tests`) |
| [#45](https://github.com/aivorah24/aivorah-frontend/pull/45) | fix(teacher/assignments): B01 — stop leaking internal topic codes as chips | `fix/b01-topic-code-leak` | 2026-09-08 | decide — check if still reproducible on `master` |

No open issues — bugs are tracked in `aivorah-frontend/docs/plans/BUGS_AND_FIXES.md` (see below).

### `aivorah-e2e-tests`

| PR | Title | Last update | Suggested |
|---|---|---|---|
| [#43](https://github.com/aivorah24/aivorah-e2e-tests/pull/43) | test(e2e): teacher student status tags | 2026-10-12 | merge after a local run |
| [#41](https://github.com/aivorah24/aivorah-e2e-tests/pull/41) | Port frontend PR #47 test fixes + new teacher specs | 2026-10-06 | finish |
| [#27](https://github.com/aivorah24/aivorah-e2e-tests/pull/27) | Add e2e regression test: onboarding Save & Exit losing name/school | 2026-08-21 | decide |
| [#26](https://github.com/aivorah24/aivorah-e2e-tests/pull/26) | Add CI: lint test code (GitHub Actions) | 2026-08-19 | close or port to CircleCI — Actions is billing-blocked |
| [#13](https://github.com/aivorah24/aivorah-e2e-tests/pull/13) | Update CLAUDE.md rules and progress log | 2026-08-07 | close (stale) |
| [#12](https://github.com/aivorah24/aivorah-e2e-tests/pull/12) | Add check-verified and forgot-password E2E tests | 2026-08-07 | decide |
| [#8](https://github.com/aivorah24/aivorah-e2e-tests/pull/8) | Add admin test creds + cap local Playwright workers | 2026-08-07 | decide |

### `aivorah-db`

| PR | Title | Last update | Suggested |
|---|---|---|---|
| [#3](https://github.com/aivorah24/aivorah-db/pull/3) | Add CI: schema validation on migration PRs (GitHub Actions) | 2026-09-23 | decide — Actions is billing-blocked |

### Other repos

⏳ Not yet listed: the 20 service repos and `ai-mentor`. Known in-flight branches:

| Repo | Branch | State |
|---|---|---|
| `aivorah-learning-paths-service` | `feat/icse-official-ingestion` | ICSE/CISCE ingestion: download/extract works; class/topic splitting needs fixing |
| `aivorah-platform-admin-service` | — | Backs the Aivorah Admin persona (frontend #107 merged 2026-10). Confirm it is deployed and env vars are set on Render |

## Known bugs

Source: `aivorah-frontend/docs/plans/BUGS_AND_FIXES.md` (from the 27-08-2026 bug report). All bug entries in it are marked ✅ CLOSED except:

- **#10** — avatar list scoped to grade/curriculum: classification (bug vs enhancement) still open.
- **#17** — teacher schedule alignment: listed as a bug, no entry yet.
- The **enhancements** list in the same file (filter defaults, hide-completed, location picker, etc.) is deferred.

## Tech debt

| Item | Impact |
|---|---|
| `master` lint fails on pre-existing errors in the frontend | CI `build_and_test` is red for reasons unrelated to the PR. Hides real failures |
| Monolith `ai-mentor/server` still serves legacy screens via `https://aivorah.ai` | Two backends to run. Rule: no new dependencies; move features to services. See `aivorah-frontend/docs/plans/MONOLITH_MOP_UP_PLAN.md` |
| `ai-mentor/backend` (Python RAG) deploy target unknown | Can't tell whether it is live or needed |
| No `/health` convention across services | Harder to verify deploys and set up monitoring |
| No monitoring / alerting | Outages are found by users |
| `aivorah-db` migration numbers have gaps (0003–0005, 0007 missing) | Harmless, but confirm none were applied to Prod outside the repo — check `schema_migrations` |
| `aivorah-frontend/docs/*.md` (Aug 2026) partly out of date | e.g. says 15 services, there are 20. Prefer this handover repo |
| `aivorah-school-admin-dashboard-service` only on the handover laptop | Push to org or delete |

## Risks and discrepancies to resolve

| # | Item | Action |
|---|---|---|
| 1 | **Supabase Dev/Test refs disagree.** `01-overview/system-overview.md` says Dev `evlqkcozespsfrnuhevf` / Test `xmwzifxlpmgsrxgxsgan`. `aivorah-db/.env.example` and its README say Dev `ernsrofvmszhmfhxeehx` / Test `vgqrrseziwnljkencqpy`. Prod (`jwlepxmclbkrmfzrumch`) agrees | Check in the Supabase dashboard; fix whichever doc is wrong |
| 2 | Which Supabase project each Render service actually points at | Check each service's `SUPABASE_URL` in Render; record it in `repositories.md` |
| 3 | Monolith repo in a personal GitHub account | Transfer to `aivorah24` |
| 4 | CircleCI / GitHub Actions billing-blocked | Fix billing or move CI to GitLab properly. Until then PR checks are manual |
| 5 | Third-party keys on personal free tiers | Re-issue under company accounts ([key-rotation](../03-runbooks/key-rotation.md)) |
| 6 | `school-admin-service` trusts an `adminEmail` request param without token check | Security fix before wider rollout |
| 7 | No staging backend — PR previews hit live services | Be careful with destructive actions on previews; consider a staging Render env |
