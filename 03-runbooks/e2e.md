# E2E tests (`aivorah-e2e-tests`)

_Last verified: 2026-10-16._

`aivorah-e2e-tests` is the **only** E2E repo (Playwright). Do not add E2E tests to `aivorah-frontend`.

## 1. Automatic smoke check on every frontend PR

```
aivorah-frontend CircleCI: build_and_test ─► trigger-e2e
   └─ polls GitHub Deployments for this commit's Vercel preview URL (≤5 min)
   └─ POSTs a pipeline to aivorah-e2e-tests' CircleCI (run_pr_preview_check=true, base_url, target_sha)
aivorah-e2e-tests CircleCI: pr-preview-smoke
   └─ sets status "e2e/smoke (aivorah-e2e-tests)" = pending on the frontend commit
   └─ logs in parent-a, student-a, teacher-a, parent-j; runs 7 smoke specs
   └─ sets status success / failure; HTML report in CircleCI artifacts
```

Secrets it needs (CircleCI project env vars, values never in git):

| Project | Variable | Purpose |
|---|---|---|
| aivorah-frontend | `GH_DISPATCH_TOKEN` | Read GitHub Deployments |
| aivorah-frontend | `CIRCLECI_DISPATCH_TOKEN` | Trigger the e2e pipeline |
| aivorah-e2e-tests | `FRONTEND_STATUS_TOKEN` | Write the commit status back |
| aivorah-e2e-tests | test account creds + `SUPABASE_URL`/`SUPABASE_SERVICE_ROLE_KEY` | Log in / reset test accounts |

**No status on the PR?** Check, in order: did `build_and_test` pass? Did `trigger-e2e` log "No successful Vercel preview" (then it skips on purpose)? Is CircleCI billing-blocked (pipeline never started)?

## 2. Run locally

```bash
git clone git@github.com:aivorah24/aivorah-e2e-tests.git && cd aivorah-e2e-tests
npm ci
npx playwright install chromium chrome
cp .env.example .env              # fill in test account creds
BASE_URL=<app url> npx playwright test tests/<persona>/<feature> --project=chromium --reporter=line
```

- `BASE_URL` defaults to `http://localhost:3006`. Point it at a Vercel preview to test a PR.
- Full suite setup/teardown: `node tests/ci/setup.mjs` / `node tests/ci/teardown.mjs`. They log every account in and snapshot/restore mutated profiles.
- Videos land in `recordings/` (gitignored).

## 3. Full suite on CircleCI (manual)

Trigger a pipeline on `aivorah-e2e-tests` with `run_full_e2e=true` (optional `test_filter`). It runs setup → 4 shards → teardown against `https://aivorah-frontend-git-master-aivorah24-9470s-projects.vercel.app`.

GitLab (`.gitlab-ci.yml`) is a fallback runner. It only runs when started from the GitLab UI or API.

## Rules for writing tests (from the repo's `CLAUDE.md`)

- The persona under test acts **through the UI**. API calls are only for setup or for read-only expected-value lookups. The final assertion is always a UI check.
- Every test must be parallel-safe — isolate with a dedicated mutable account, never `mode: 'serial'`.
- One PR per screen/flow. Branch protection is not enforced on GitHub Free — follow it manually.
