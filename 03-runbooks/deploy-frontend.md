# Deploy — frontend (`aivorah-frontend`)

_Last verified: 2026-10-16 against `vercel.json`, `.circleci/config.yml`, `eas.json`._

## How it works

| Trigger | What happens |
|---|---|
| Push to any PR branch | Vercel builds a **preview** (`npx expo export --platform web` → `dist/`). CircleCI runs `build_and_test`, then `trigger-e2e` sends the preview URL to `aivorah-e2e-tests`, which posts an `e2e/smoke (aivorah-e2e-tests)` status back on the commit. |
| Merge to `master` | Vercel builds and promotes to **production** automatically. |

Vercel: team `aivorah24-9470s-projects`, project `aivorah-frontend`. SPA rewrites (`/(.*)` → `/index.html`) are in `vercel.json`.

> ⚠️ CircleCI is billing-blocked / unreliable. If it does not run, run the same checks locally (see [local-dev.md](local-dev.md)) and say so in the PR. A missing `e2e/smoke` status is **not** a pass.

## Steps — web release

1. Branch from `master`, make the change, push, open a PR (fill in the PR template).
2. Wait for the Vercel preview. Open it and check the screens you touched — the preview calls the **live** Render services, so it shows real behaviour.
3. Check PR status: `build_and_test` green (or local run noted), `e2e/smoke` green.
4. If the change needs a backend change too, **deploy the service first** ([deploy-service.md](deploy-service.md)), then merge the frontend. Previews and prod both hit the same live services, so a frontend that relies on an un-deployed endpoint breaks immediately.
5. Merge to `master`. No direct pushes.
6. Vercel → Deployments: wait for the production build to show **Ready**.

## Verify

- Load production, hard refresh, log in as a test account for the affected persona, check the changed screen.
- Browser console has no new red errors / failed requests to `*.onrender.com`.

## Android build (EAS) — only when releasing the mobile app

Profiles in `eas.json`:

| Profile | Output | Use |
|---|---|---|
| `development` | APK, dev client | local debugging |
| `preview` | APK, internal | share a test build |
| `production` | App Bundle, version auto-increment | Play Store |

```bash
npx eas build --platform android --profile preview      # or production
npx eas submit --platform android --profile production  # Play Store
```

⚠️ Unverified: which Expo account owns the project and whether a Play Console listing exists. Confirm with the handover owner and add to the [access inventory](../02-access/access-inventory.md).

## If it goes wrong

→ [rollback.md](rollback.md#frontend-vercel)
