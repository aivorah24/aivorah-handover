# Key rotation / re-issuing secrets

Use when: handing over (new owner, departing staff), a key leaked (committed, pasted in chat, in a screenshot), or moving a provider from a personal to a company account.

## General procedure (any key)

1. **Find every consumer.** Look up the variable in the [env var table](../02-access/access-inventory.md#env-vars-per-service-names-only), and also grep the org:
   `grep -rl VAR_NAME ~/Desktop/aivorah-*/.env.example`.
2. **Issue the new key** in the provider. Use the **company** account, not a personal one.
3. **Update every consumer** in Render → service → Environment (save and redeploy), and in CircleCI / Vercel if listed.
4. **Verify** each consumer: Render deploy is Live, logs are clean, the feature works.
5. **Revoke the old key** in the provider. Do this only after step 4 — revoking first causes an outage.
6. Update developer `.env` files and tell the team. Tick the item in the [access inventory](../02-access/access-inventory.md).

If the key **leaked**, do steps 2–5 straight away, then remove it from git history if it was committed. Removing it from history is not enough by itself — the key must be revoked.

## Key-specific notes

| Secret | Where to issue | Consumers | Notes |
|---|---|---|---|
| `SUPABASE_SERVICE_KEY` / `SUPABASE_SERVICE_ROLE_KEY` | Supabase → project → Settings → API | Almost every service; e2e CircleCI | Bypasses RLS. If the project uses the newer API keys, create a new **secret** key, swap it in, then delete the old one. If it only has the legacy JWT keys, rotating the **JWT secret** invalidates `anon` and `service_role` together. The frontend does not use Supabase directly, so only backends are affected. ⚠️ Check which key type each project uses. |
| `FIREBASE_SERVICE_ACCOUNT_B64` / `FIREBASE_SERVICE_ACCOUNT` | Google Cloud console → project `ai-learning-platform-c1308` → IAM → Service Accounts → Keys → Add key (JSON) | auth, learning-paths, schedule, attendance, messages, students, timetable-resources, platform-admin; local `firebase-service-account.json` | Encode with `base64 -w0 key.json`. Delete the old key ID in the console after the swap. |
| `FIREBASE_API_KEY` (web) | Firebase console | frontend + some services | A public identifier, not a secret. Restrict it by HTTP referrer instead of rotating. |
| `ENCRYPTION_KEY` | Self-generated | auth, profile, monolith | ⚠️ **Do not rotate blindly.** Data encrypted with the old key can't be decrypted with the new one. Find what it encrypts and plan a re-encryption first. |
| `INTERNAL_API_KEY` | Self-generated (`openssl rand -hex 32`) | messages + its callers | Change caller and callee in the same window. |
| `RESEND_API_KEY`, `EMAIL_USER`/`EMAIL_PASS` | Resend dashboard; Google account → App passwords | auth, students, platform-admin, help-center, monolith | Re-verify the sending domain in Resend if the account changes. |
| LLM keys (`GROQ_*`, `GEMINI_*`, `OPENROUTER_*`, `CEREBRAS_*`, `NVIDIA_*`) | Each provider's console | see env table | Check the free-tier rate limits on the new account — ai-coach rotates across `GROQ_API_KEY_1..4` for this reason. |
| `CLOUDINARY_*`, `ELEVENLABS_*`, `QDRANT_*` | Provider dashboards | see env table | Moving to a new Cloudinary/Qdrant **account** means migrating data, not only the key. |
| `SUPABASE_ACCESS_TOKEN` | Supabase → Account → Access Tokens | `aivorah-db` (personal, local only) | Each person uses their own. Revoke the departing person's. |
| CircleCI `GH_DISPATCH_TOKEN`, `CIRCLECI_DISPATCH_TOKEN`, `FRONTEND_STATUS_TOKEN` | GitHub fine-grained PAT; CircleCI personal API token | CircleCI project env vars | They are tied to the person who created them, so they **must** be re-issued by the receiver. Otherwise E2E status reporting stops when the departing account is removed. |

## Handover rotation order

Do this once the receiver has admin access everywhere (see [access inventory → security actions](../02-access/access-inventory.md#security-actions-before-handover)):

1. GitHub token embedded in the local `ai-mentor` remote — revoke now; it doesn't wait for anything.
2. CI tokens (re-issued by the receiver).
3. Supabase service keys, Firebase service account.
4. Third-party keys moved to company accounts (email, LLMs, ElevenLabs, Cloudinary).
5. Remove departing accounts from every platform.
