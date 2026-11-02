# Notification System V2 — Design & PR Plan (2026-09-24)

> Moved from an untracked draft in `aivorah-frontend/docs/architecture/` (written 2026-09-24). Not re-verified since; check the code before relying on it.

The status line below is from 2026-09-24. As of 2026-09-25, PR 0–3 were live; PR 4 and PR 5 were not started.

Status: **PR 1 committed + migration live on Dev/Test. PR 2 (producers) committed locally, validated live on Dev/Test, not deployed.** §0b records the decisions PR 1 validation changed. Where §0b conflicts with a later section, §0b wins; the migration file is the final schema.

`NOTIFICATION_SYSTEM_AUDIT.md` describes how the system works today. §0 re-checks it against the live system and records what changed.

---

## 0. Validation of the audit (done before design)

| Check | Result | How verified |
|---|---|---|
| Audit findings F1–F16 still match the code | Yes. Line refs are unchanged. | Re-read of the routes and files cited |
| Uncommitted F1–F3 fixes still present | Yes: frontend (10 files + 1 new test), challenges-service `routes/challenges.js`, messages-service `routes/messages.js` | `git status` in all 3 repos |
| **Live DB for challenges-service** | **Test** (`xmwzifxlpmgsrxgxsgan`) | Live `GET /challenges/assigned` returns rows that exist only in Test (Dev has 0 `challenge_assignments`) |
| **Live DB for assignments-service** | **Test** | Live `GET /assignments` returns assignment `1ae01cde…`, which exists only in Test |
| **Live DB for messages-service** | **Test** | Live `notifications-summary` returns `challenges:5` for a Test student |
| Live DB for profile-service | Inconclusive (probe user exists in both Dev and Test); very likely Test | Live `/user/preferences` |
| Prod (`jwlepxmclbkrmfzrumch`) | No live service found that uses it. This matches the standing note that real production doesn't exist yet. | — |
| Local `.env` files | Dev for messages, challenges, assignments, profile, schedule and habits; Test for students | `.env` |
| **Parents in `users`** | **Test: 4 of the 8 linked parent emails have NO `users` row.** Dev: 4 of 4 do. | Read-only count, `parent_children` left join `users` |
| Email casing | 0 mixed-case emails in `users` or `parent_children` (Dev and Test) | Read-only count |
| Duplicate `(challenge_id, student_id)` pairs | 0 in Dev and Test, so a UNIQUE constraint is safe to add | Read-only count |
| Legacy parent system messages (Test) | 24 total, 14 unread, 10 unread from the last 14 days | Read-only count |
| `notifications` table exists | No (Dev and Test) | — |
| Existing auth pattern | Firebase ID token → `firebase-admin.verifyIdToken` → `users` by email. Used in learning-paths-service `middleware/auth.js`. Frontend: `auth.currentUser.getIdToken()` | Code |
| Realtime WS auth | Plain `{type:'auth', email}`, **no verification** (`messages-service/services/realtime.js`) | Code |
| pg_cron / pg_net available | **Not verified.** The Supabase management token in `aivorah-db/.env` returns 401 (expired). | `scripts/lib.js` query |
| Parent screens: child selection | Local `selectedIdx` state per screen. Nothing reads a route param. | `useParentChallenges.ts` |
| `demoLogin()` | Exists in `AuthContext`. Creates a session **without a Firebase token**. | `AuthContext.tsx:80` |

## 0b. Decisions revised during PR 1 validation

| Topic | Original proposal | Final (implemented) | Evidence |
|---|---|---|---|
| Recipient identity | lowercased email | **`recipient_id uuid NOT NULL → users.id ON DELETE CASCADE`** | The 4 Test parents with no `users` row are all `e2e…@example.com` fixtures with 0 messages; every real parent has a row. Email is **not stable**: auth-service `renameEmailAcrossData` (`accountManagement.js:54`) renames the email-keyed tables but would never update a notifications row. `users.id` survives an email change, and the cascade gives free cleanup on account delete. |
| Orphan parent links | would have been notified | skipped, counted as `orphanParentLinks` in the fan-out result + log | same |
| Auth → recipient | token email = recipient | token email → `users.id` (403 if no row) | `middleware/firebaseAuth.js` |
| Parent event key | same key as student | **base key + `:child=<childId>`** | A parent of two children in the same class-wide assignment must get one notification *per child*; with one key the second would be dropped as a duplicate. Challenge rows are per-student already, so only class-wide assignments hit this. |
| Version source | `assigned_at` | confirmed | The only path that reactivates a `challenge_assignments` row is the completed→re-assign UPDATE (`challenges.js:199-216`); unassign + assign creates a new row id. `assignments` rows are only inserted or deleted (plus a `quiz_questions` update), so `created_at` is a safe version. A sub-millisecond difference maps to the same key (tested). |
| `push_status` values | disabled/pending/sent/failed/skipped | adds **`sending`** | Needed for PR 4's atomic claim (below). Added now so PR 4 needs no constraint change. |

**Cron responsibilities (PR 4 / PR 5), with one scheduler and one endpoint:**

| Trigger | Calls | Does | Ships in |
|---|---|---|---|
| `AFTER INSERT` on notifications → `pg_net` | `POST /internal/notifications/dispatch {ids}` | realtime WS + push attempt for those ids (fast path) | PR 4 |
| `pg_cron` every 1 min → `pg_net` | `POST /internal/notifications/tick` | (a) claim + send `pending` pushes whose `push_next_at <= now()` (retries) · (b) fetch Expo receipts, prune dead tokens | PR 4 |
| same tick | same | (c) due scan → `notify_event(challenge_due / assignment_due)` | PR 5 (adds a stage; no new job) |

- **No double sends.** The dispatch path and the sweep can hit the same row. Both claim it with one `UPDATE … SET push_status='sending', push_next_at = now()+2min WHERE id IN (SELECT … FOR UPDATE SKIP LOCKED) RETURNING`. If a lease expires, the row returns to the sweep.
- **Due creation is idempotent** through `event_key`, so overlapping ticks are harmless.
- **Not verified yet:** whether `pg_cron`/`pg_net` are enabled. The management token is still returning 401. This is needed before PR 4, not PR 1.

---

## 1. Final architecture

```
Producer service (challenges / assignments / scheduler)
   │  domain write succeeds
   ▼
DB function  notify_event(event)            ← one fan-out implementation, atomic, idempotent
   │  resolves student(s) + linked parents, excludes actor,
   │  INSERT … ON CONFLICT (recipient_email, event_key) DO NOTHING
   ▼
notifications (one row per recipient)        ← single source of truth
   │ AFTER INSERT trigger → pg_net POST (wakes messages-service)
   │ pg_cron every minute → same endpoint (sweep / retry safety net)
   ▼
messages-service  /internal/notifications/dispatch   ← single delivery owner
   ├─ Realtime WS  {type:'notification'}  (web + mobile, token-authenticated sockets)
   ├─ Expo push    (mobile; preferences checked here; retries; receipts; token pruning)
   └─ (future) email
   ▼
Frontend  NotificationContext (API + WS, polling fallback)
   ├─ Bell / Badge / Toast
   └─ Push tap (foreground / background / cold start)
          ▼
     openNotification(n)  ← one router
          ▼
     exact screen + child + entity
```

**Ownership:**
- **messages-service** owns the table's APIs, delivery, realtime, and the scheduled due scan.
- **The database** owns fan-out and idempotency, through the SQL function.
- **Producers** own only "what happened" (type, entity, affected students, title/body/deep-link data).

**Why fan-out lives in a DB function** and not in JS copied into every producer, or behind an HTTP call to messages-service:
- It is atomic, and idempotency happens in one statement.
- There is no second copy of the logic (the audit's `push.js` problem).
- A producer's assign request doesn't depend on messages-service being awake. A Render cold start can take 30–60 s.

The trade-off is that the SQL logic needs DB-level integration tests (§10).

Kept deliberately out: Redis, BullMQ, Kafka, a separate notification service, Web Push, email.

## 2. Final database model (migration `aivorah-db/supabase/migrations/0006_notifications_v2.sql`, additive only)

**Recipient identity = lowercased email, not `users.id`.** Half the parents in Test have no `users` row, and every parent link, the Firebase token and the legacy `messages.recipient_email` are all keyed by email. `recipient_user_id` is still stored when a row exists.

```sql
create table public.notifications (
  id                uuid primary key default gen_random_uuid(),
  recipient_email   text not null check (recipient_email = lower(recipient_email)),
  recipient_user_id uuid references public.users(id) on delete set null,
  recipient_role    text not null check (recipient_role in ('student','parent','teacher','school_admin')),
  type              text not null,          -- challenge_assigned | assignment_assigned | challenge_due | assignment_due
  entity_type       text not null,          -- challenge_assignment | assignment
  entity_id         text not null,
  child_id          uuid references public.users(id) on delete cascade,  -- set for parent rows
  title             text not null,
  body              text,
  data              jsonb not null default '{}'::jsonb,   -- {challengeId, assignmentId, subject, dueAt, ...}
  event_key         text not null,
  actor_email       text,
  actor_role        text,
  created_at        timestamptz not null default now(),
  read_at           timestamptz,
  push_status       text not null default 'disabled'
                    check (push_status in ('disabled','pending','sent','failed','skipped')),
  push_attempts     int  not null default 0,
  push_next_at      timestamptz,
  push_error        text,
  push_ticket_ids   text[],
  constraint notifications_recipient_event_uniq unique (recipient_email, event_key)
);
create index notifications_feed_idx   on public.notifications (recipient_email, created_at desc, id desc);
create index notifications_unread_idx on public.notifications (recipient_email) where read_at is null;
create index notifications_push_idx   on public.notifications (push_next_at) where push_status = 'pending';
create index notifications_entity_idx on public.notifications (entity_type, entity_id);
alter table public.notifications enable row level security;  -- no policies: service-role only

-- Versioning for legitimate re-assignment (Decision 1)
alter table public.challenge_assignments add column if not exists assigned_at timestamptz;
update public.challenge_assignments set assigned_at = created_at where assigned_at is null;
alter table public.challenge_assignments alter column assigned_at set default now();
alter table public.challenge_assignments add constraint challenge_assignments_challenge_student_uniq
  unique (challenge_id, student_id);                       -- verified 0 duplicates in Dev/Test

-- notify_event(p jsonb) returns setof notifications   (fan-out + idempotent insert)
-- due-scan helper views/functions for PR 5
```

`push_status` starts as `disabled` until PR 4. This way the legacy push and the V2 push never both fire during the dual-run.

## 3. Event key strategy (one format)

```
<type>:<entity_type>:<entity_id>:<version>[:<qualifier>]
```
- The recipient is **not** part of the key, because the unique constraint is `(recipient_email, event_key)`. This keeps "all recipients of one event" queryable by `event_key`.
- `version` is an immutable timestamp of the domain *action*, in epoch ms, so it never depends on the client clock or on formatting.

| Event | Key | Version source |
|---|---|---|
| Challenge assigned / re-assigned | `challenge_assigned:challenge_assignment:<caId>:<assigned_at ms>` | New `challenge_assignments.assigned_at`, set on insert **and** on the completed→re-assign UPDATE |
| Assignment assigned | `assignment_assigned:assignment:<assignmentId>:<created_at ms>` | `assignments.created_at` (assignments are never re-assigned in place today) |
| Challenge due | `challenge_due:challenge_assignment:<caId>:<assigned_at ms>:<due_date>` | Includes the due date, so a changed due date gives one new due event |
| Assignment due | `assignment_due:assignment:<assignmentId>:<created_at ms>:<due_date>` | Same |

How each case behaves:
- **Duplicate request or double submit:** the same row gives the same `assigned_at`, the same key, and `DO NOTHING`. The unique `(challenge_id, student_id)` also stops a second domain row: 23505 is mapped to the existing `alreadyAssigned` response.
- **Legitimate re-assignment of a completed challenge:** a new `assigned_at` gives a new key and a new notification (Test 8).

## 4. Scheduled / due notification strategy

**Definition (v1):** a due notification is created once, at the item's due time.
- It goes only to recipients whose work is **not complete**: the challenge is not `completed`, or the assignment has no submission from that student.
- There are no reminders. A "due in 24 h" reminder would be a separate future type (`*_due_soon`).

**Due time:**
- `challenge_assignments.due_date` is a DATE, and `assignments.due_date` may carry a time.
- v1: `due_at` = the given timestamp, or the end of the day (23:59:59) in **Asia/Kolkata** when only a date is given.
- ⚠️ The existing `isPastDue` in challenges-service uses the server's UTC. That is a pre-existing inconsistency, and PR 5 aligns it.

**Scheduler: Supabase `pg_cron` + `pg_net`**
- A job runs every 5 minutes and `POST`s messages-service `/internal/notifications/tick`, authenticated with a shared secret.
- The tick endpoint scans for items where `now() - 24h < due_at <= now()` and calls `notify_event` for each. The 24 h lookback limits the backlog on first run or after an outage.
- It is idempotent through `event_key`: a second run inserts 0 rows. The same tick also sweeps pending pushes.
- **Why pg_cron:** it is free and already inside the DB we use, and the call wakes a sleeping Render service. A Render Cron Job is paid and doesn't exist today.
- **Fallback if pg_cron/pg_net can't be enabled:** a GitLab scheduled pipeline (already the free CI fallback) calling the same endpoint.
- **Prerequisite:** confirm the extensions are available. That needs a fresh `SUPABASE_ACCESS_TOKEN` (the current one returns 401).

## 5. Authentication strategy (minimal boundary)

1. **messages-service** gets `middleware/firebaseAuth.js`, a copy of the verified pattern in learning-paths-service `middleware/auth.js` *without* the role gate.
   - It checks `Authorization: Bearer <Firebase ID token>`, runs `verifyIdToken`, and sets `req.auth = { email: lower(token.email), userId?, role? }`.
   - Every `/notifications*` route filters by `req.auth.email` only. **No `userEmail` parameter is accepted.**
2. Internal endpoints (`/internal/notifications/*`) need an `X-Internal-Key` header equal to env `INTERNAL_API_KEY`. The hard-coded `'aivorah-admin-2026'` in `cleanup-system-notifs` moves to the same env key in PR 1, which is a security fix only; the endpoint itself is removed in PR 6.
3. **Realtime WS:** a new `{type:'auth', token}` message is verified the same way. Notification events go **only** to token-verified sockets. The legacy `{type:'auth', email}` path stays for chat until PR 6, and is listed there as remaining legacy auth.
4. Frontend: `getAuthHeaders()` uses `auth.currentUser.getIdToken()`, the existing pattern.
5. **Documented as out of scope (legacy auth debt):** every other `?userEmail=` endpoint across the services, including legacy `notifications-summary`, `mark-read` and chat.
6. **Known gap:** `demoLogin()` sessions have no Firebase token, so V2 returns 401 for them. The frontend treats that as an empty feed. Demo accounts don't get V2 notifications unless you want that changed.

**Render env needed on messages-service:** `FIREBASE_SERVICE_ACCOUNT_B64` and `INTERNAL_API_KEY`.

## 6. Class-wide assignment strategy

- **A notification is a point-in-time event.**
  - Recipients are the students eligible **when the assignment is created**, plus exactly those students' linked parents.
  - Students who join later get no historical notification. The assignment still appears for them through `GET /assignments`, because eligibility and notifications are separate.
- **Eligibility has one definition.** `assignments-service/services/assignmentRecipients.js` implements the rule the student feed uses today (`messages.js:236-249, 353-358`):
  - curriculum is null or matches
  - grade is null or matches
  - teacher broadcasts go only to that teacher's `teacher_student` students
  - school must match the creator's school
  - PR 2 adds a parity test against `GET /assignments` filtering. The audit found that the summary's comment *claims* it mirrors `GET /assignments`, but this wasn't verified line by line, so the parity test is an acceptance gate.
- Parents are derived **only** from the resolved student list. This fixes F15, where a parent was notified about work their child never sees.
- A parent who created the assignment is excluded, the same as today's assignment path. The challenge path is aligned to match (F14).

## 7. Web / mobile delivery strategy

| | Web (testers) | Mobile |
|---|---|---|
| Persisted row | ✔ | ✔ |
| Realtime WS event → feed update + toast | ✔ (PR 4) | ✔ while foregrounded |
| Polling fallback | ✔ (slowed to 30 s once WS is proven) | ✔ |
| OS push | ✘ (Web Push is a future enhancement, not claimed) | ✔ Expo |
| Tap → `openNotification` | Bell / toast | Bell / toast / push (foreground, background, cold start) |

## 8. Push ownership strategy

- **Single owner:** `messages-service/services/notificationDelivery.js`.
- **Producers stop calling `sendPush`** for events covered by V2. From PR 4 they insert with `push_status='pending'` and the legacy calls are removed from those code paths, which is a switch rather than a file deletion. The duplicated `push.js` files are deleted in PR 6.
- **Triggers:**
  - an `AFTER INSERT` trigger does a pg_net POST to `/internal/notifications/dispatch` with the new ids, for low latency
  - the `/tick` sweep picks up `pending` rows where `push_next_at <= now()`, for reliability
- **Per row:**
  1. check preferences: `challengeUpdates` / `assignmentAlerts` / a global push switch
  2. if the user has opted out, set `skipped`
  3. if there are no tokens, set `skipped`
  4. otherwise send to Expo and store the ticket ids, then set `sent`
  5. on error: `push_attempts++`, backoff 1 → 5 → 15 → 60 min, and `failed` after 5 attempts
- **Receipts:** the tick fetches Expo receipts for rows `sent` more than 15 minutes ago. A `DeviceNotRegistered` receipt removes that token from `user_preferences.data.expoPushTokens`.
- **Logout:** a new authenticated `DELETE /user/push-token` in profile-service, called on logout. This fixes a shared device receiving the previous user's pushes.
- **Preferences model (Decision 7):** the notification row is **always** created, and preferences only gate push and email.
  - ⚠️ **This is a visible behaviour change.** Today the bell *hides* items when a toggle is off (`NotificationContext.tsx:265-272`). In V2 they stay in the bell, and only push is suppressed.

## 9. Migration / backfill strategy

- **Order:** Dev → Test. Prod only when it goes live. All statements are additive, and there is no destructive DDL. They are applied with `aivorah-db/scripts/migrate.js`, which needs a fresh access token.
- **Dual-run (PR 2 → PR 5):**
  - Producers write V2 rows **and** keep writing the legacy parent messages and legacy push.
  - The frontend keeps the legacy feed until PR 3, where it switches behind the flag `EXPO_PUBLIC_NOTIFICATIONS_V2`, default on, and can be reverted by redeploying with the flag off.
- **Backfill** (`messages-service/scripts/backfill-notifications-v2.js`, `--dry-run` by default, run once after PR 2 deploys):
  - **Window:** the last 14 days.
  - **Sources:**
    - active `challenge_assignments` and `assignments`, both personal and class-wide through the same recipient resolver
    - parents derived from the same rows, not by parsing message text
  - **Keys:** the backfill goes through `notify_event` with the same keys the producers use, so running it alongside live dual-writes cannot duplicate anything.
  - **Read state:**
    - rows whose work is finished (challenge completed, assignment submitted) are marked read
    - a parent row is marked read if the matching legacy system message is read (same recipient, same title, created within ±2 min)
    - everything else is unread
  - Test volume is tiny (16 challenge assignments, 10 unread parent alerts in 14 days).
- **Rollback per PR:** revert the deploy. The table and columns are additive and can stay. Frontend: set the flag off.

## 10. Six-PR plan (+ PR 0)

### PR 0: ship the existing F1–F3 fixes (already written and tested)
Frontend (10 files + test), challenges-service `/assigned-by`, messages-service parent-title parse + slice 20. It is independent of V2 and gives testers immediate relief. PR 3 later replaces the context/toast parts.

### PR 1: Foundation *(aivorah-db, messages-service)*
- Migration 0006: table, indexes, `assigned_at`, unique constraint, `notify_event()`.
- messages-service:
  - `middleware/firebaseAuth.js`
  - `services/notifications.js` (`buildEventKey`, `createNotificationEvent` → rpc, keyset cursor encode/decode)
  - `routes/notifications.js`:
    - `GET /notifications?cursor=&limit=` sorted `created_at desc, id desc` with an opaque cursor
    - `GET /notifications/unread-count`
    - `POST /notifications/:id/read`, which only updates when `recipient_email` = the caller, and otherwise returns 404
    - `POST /notifications/read-all`
  - admin key moved to env.
- **Tests:**
  - *Unit:* event key format and versioning, cursor, auth middleware (missing, invalid or expired token, email lowercasing).
  - *Integration (supertest, mocked Supabase):* ordering, ties on `created_at` broken by `id`, pagination with no gaps or duplicates, read-one, read-all, unread-count, cross-user access → 404.
  - *DB-level:* `scripts/test-notify-event.sql` run against Dev: duplicate → 1 row, new version → 2 rows, multiple parents, actor excluded, parent with no `users` row still gets a row.
- **No frontend changes. No producer changes.**

### PR 2: Producers *(challenges-service, assignments-service, messages-service backfill)*
- **Challenges `/assign`:**
  - set `assigned_at` on insert and re-assign
  - map a 23505 unique violation to `alreadyAssigned`
  - after the domain write succeeds, **await** `notify_event` (the student plus parents, actor excluded)
  - log failures as structured logs without failing the assign
  - legacy message and push stay
- **Assignments `POST /`:** individual (student + parents) and class-wide (resolver → students + their parents); await `notify_event`; legacy paths stay.
- **Structured log line per event:** `{evt:'notification.create', type, eventKey, entity, recipients:n, inserted:n, duplicates:n, error?}`. It contains no names, message content or tokens; recipients are logged as a count, and as a hashed email only in debug mode.
- **Tests:**
  - payload and recipient unit tests
  - integration: a rejected request (grade guard, 403, 404) creates no notification; double submit creates one; re-assign completed creates a new one; class-wide recipients match the resolver
  - resolver parity test
- Backfill script plus a dry-run report.

### PR 3: Frontend V2 feed + router *(aivorah-frontend)*
- `src/services/notificationsApi.ts`, which sends the token.
- `src/navigation/openNotification.ts`, the **single** router. It maps role × type → screen + params `{highlightId: entityId, childId, tab}` and marks only that notification read.
- `NotificationContext` V2:
  - feed from the API, badge from unread-count
  - toast when an unseen id arrives; no AsyncStorage `*SeenAt`
  - polling at 8 s for now
- `Header` and `NotificationBanner` use the context's items and `openNotification`, with pagination ("Load more"), loading, empty and error states.
- The parent Challenges and Assignments screens read `childId` (select that child) and `highlightId` (open that entity).
- One banner mount: remove `AppNavigator.tsx:115` (F13).
- **Tests:**
  - newest first, badge
  - toast shows the new item
  - tap → correct navigate params for the student and the parent
  - only the tapped item is read
  - the parent screen selects the child from `childId`
  - error and empty states
- **E2E** (`aivorah-e2e-tests`), tests 1–8, run against the preview URL plus the Test backends.

### PR 4: Delivery *(messages-service, profile-service, challenges/assignments, frontend, aivorah-db)*
- **messages-service:**
  - `notificationDelivery.js`: preferences, Expo send, retry/backoff, receipts, token pruning
  - `/internal/notifications/dispatch` and `/tick`
  - WS token auth plus `{type:'notification'}` events
- **Migration 0007:** trigger (pg_net) plus the pg_cron sweep job.
- Producers insert `push_status='pending'` and drop their own `sendPush` for these events.
- **profile-service:** authenticated `DELETE /user/push-token`.
- **Frontend:**
  - push listeners in `App.tsx` (`addNotificationResponseReceivedListener` plus `getLastNotificationResponseAsync` for cold start) → `openNotification`
  - WS subscription → feed refresh plus toast
  - unregister the token on logout
- **Tests:**
  - delivery unit tests: preference skip, no token, Expo error → retry, receipt `DeviceNotRegistered` → token removed
  - frontend listener tests with a mocked `expo-notifications`
  - WS event test
- **Real-device verification (manual, reported separately):** foreground, background and killed app on an EAS dev build. **This will not be claimed from Jest.**

### PR 5: Due events *(messages-service, aivorah-db, challenges-service)*
- A due scan inside `/tick`, `due_at` in Asia/Kolkata, skip completed or submitted work, a 24 h lookback, event keys per §3.
- pg_cron every 5 minutes. **No due notification comes from creating the item.**
- The legacy `schedule_events`-derived bell items are only in the legacy feed, which is no longer used after PR 3, so F10 is gone.
- Align `isPastDue`.
- **Tests:** the scan runs twice and gives exactly 1 notification per recipient; a completed item gets none; a changed due date gives one new event; boundary times around IST midnight.
- **E2E:** test 9, using a short due time plus a manual tick call.

### PR 6: Legacy removal — **checklist only, needs separate approval**
Each item comes with a usage search, production-traffic evidence (Render logs) and green E2E before removal:
- the legacy `system@aivorah.com` inserts
- the legacy `notifications-summary`, `mark-system-read` and `cleanup-system-notifs` routes
- the `*SeenAt` AsyncStorage keys
- the duplicated `push.js` files
- monolith notification code (**needs its own approval**, per the monolith rule)
- WS email-only auth
- a one-time cleanup of legacy system messages

## 11. Testing layers (kept separate in every report)
1. Service unit + supertest integration (Jest, mocked Supabase).
2. DB-level SQL tests for `notify_event`, constraints and the due scan (run against **Dev** only).
3. Frontend Jest (RNTL).
4. E2E in `aivorah-e2e-tests` against the Vercel preview + Test backends. CircleCI is billing-blocked, so these go through GitLab or a local run.
5. Real-device push checks (manual, EAS build).

## 12. Risks

| Risk | Mitigation |
|---|---|
| pg_cron/pg_net not enabled or not allowed | Verify first (needs a fresh token); GitLab scheduled pipeline fallback |
| messages-service lacks the Firebase service account | Add `FIREBASE_SERVICE_ACCOUNT_B64` to Render (your action); until then V2 returns 500 and the flag stays off |
| Render cold start delays push and WS by 30–60 s | The row is already persisted, so the feed is correct on the next poll; the trigger wakes the service |
| The Test DB is shared with testers | Additive migrations only; backfill dry-run first |
| WS connection map is in memory (one instance) | Fine at the current single instance; polling fallback stays |
| Preference behaviour change (bell shows items when push is off) | Called out for product sign-off |
| Class-wide eligibility drift vs `GET /assignments` | Parity test is a PR 2 gate |
| Demo sessions have no Firebase token | Empty V2 feed; documented |
| Due-time timezone (IST vs today's UTC `isPastDue`) | Decide now; align in PR 5 |

## 13. Files that will change

| PR | Repo | Files |
|---|---|---|
| 0 | frontend / challenges / messages | already modified (see git status) |
| 1 | aivorah-db | `supabase/migrations/0006_notifications_v2.sql`, `scripts/test-notify-event.sql` |
| 1 | messages-service | `middleware/firebaseAuth.js` (new), `services/firebaseAdmin.js` (new), `services/notifications.js` (new), `routes/notifications.js` (new), `index.js`/`server.js` (mount), `routes/messages.js` (admin key → env), `__tests__/notifications*.test.js` (new), `package.json` (`firebase-admin`) |
| 2 | challenges-service | `routes/challenges.js`, `services/challenges.js`, `services/notify.js` (new, thin rpc wrapper), tests |
| 2 | assignments-service | `routes/assignments.js`, `services/assignmentRecipients.js` (new), `services/notify.js` (new), tests |
| 2 | messages-service | `scripts/backfill-notifications-v2.js` (new) |
| 3 | frontend | `src/services/notificationsApi.ts` (new), `src/navigation/openNotification.ts` (new), `src/context/NotificationContext.tsx`, `src/components/common/Header.tsx`, `src/components/common/NotificationBanner.tsx`, `src/navigation/AppNavigator.tsx`, `src/screens/parent/challenges/hooks/useParentChallenges.ts` + screen, `src/screens/parent/assignments/hooks/useParentAssignments.ts` + screen, tests |
| 3 | aivorah-e2e-tests | new `notifications/*.spec.ts` (tests 1–8) |
| 4 | messages-service | `services/notificationDelivery.js` (new), `routes/internalNotifications.js` (new), `services/realtime.js`, tests |
| 4 | aivorah-db | `0007_notifications_delivery.sql` (trigger + cron) |
| 4 | profile-service | `routes/profile.js` (`DELETE /user/push-token`) |
| 4 | challenges / assignments | stop legacy `sendPush` for V2 events |
| 4 | frontend | `App.tsx`, `src/services/pushNotifications.ts`, `src/hooks/useNotificationSocket.ts` (new), tests |
| 5 | messages-service | `services/dueScan.js` (new), tests; aivorah-db `0008` (cron schedule); challenges-service `isPastDue` |
| 6 | all | checklist only |
