# Local development

## Prerequisites

- Node 22 (CI uses `cimg/node:22.11`); Node ≥ 18 is enough for `aivorah-db`.
- Git access to the `aivorah24` org (see [access inventory](../02-access/access-inventory.md)).
- A `.env` per service repo — copy `.env.example`, get values from Render → service → Environment (Render is the source of truth). **Point local services at Supabase Dev, never Prod.**

## Frontend (`aivorah-frontend`)

```bash
npm ci
npm run web            # Expo web, talks to the LIVE Render services by default
```

To run the frontend against **local** services instead:

```bash
EXPO_PUBLIC_USE_LOCAL_SERVICES=1 npm run web
```

Each `src/config/*ServerUrl.ts` then uses its localhost port (e.g. auth = `http://localhost:4001`). Only web + dev builds honour this flag; native builds always hit live.

Checks to run before opening a PR (same as CI `build_and_test`):

```bash
npx tsc --noEmit
npm run lint             # ⚠️ fails on master today from old lint debt — make sure you add no NEW errors
npm run test:unit
npm run test:integration
npm run vercel-build     # web export, catches build-only failures
```

Mobile: `npm run android` / `npm run ios` (Expo Go). Native builds call the live backend so they work off-Wi-Fi.

## A microservice (`aivorah-*-service`)

```bash
npm ci
cp .env.example .env     # fill from Render
npm run dev              # port is in the service's index.js
npm test                 # tests live in tests/
```

## Database (`aivorah-db`)

See [db-migrations.md](db-migrations.md). No dependencies to install; needs `SUPABASE_ACCESS_TOKEN` in `.env`.

## E2E (`aivorah-e2e-tests`)

See [e2e.md](e2e.md).
