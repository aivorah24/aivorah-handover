# Repositories

_Last verified: 2026-10-14. GitHub org: `aivorah24` (all private)._

## Apps and tooling

| Repo | What it is | Deploys to |
|---|---|---|
| `aivorah-frontend` | Expo / React Native app (web + mobile), all 5 personas | Vercel (prod from `master`, preview per PR) |
| `aivorah-e2e-tests` | Playwright E2E suite — the **only** E2E repo | Triggered from frontend CI against the preview URL |
| `aivorah-db` | Supabase schema, migrations, seeds (Dev/Test/Prod) | Not deployed — migrations are run by hand |
| `ai-mentor` | Legacy monolith: `server/` (Node/Express) + `backend/` (Python RAG/Qdrant) | ⚠️ Lives in a **personal** GitHub account (`saimohan24092003/ai-mentor`), not the org. Must be transferred. Deploy target: GCP Cloud Run (unconfirmed if still serving) |

## Microservices (all Node/Express on Render, auto-deploy on merge)

| Repo | Responsibility | Branch → Render URL |
|---|---|---|
| `aivorah-auth-service` | Login, signup, email verify, password reset, change email, `/user/sync` | `main` → aivorah-auth-service-g27s.onrender.com |
| `aivorah-profile-service` | Profile, preferences, notification settings | `main` → aivorah-profile-service-ug5c.onrender.com |
| `aivorah-leaderboard-service` | XP, rankings, student dashboard stats, My Progress | `main` → aivorah-leaderboard-service-0nou.onrender.com |
| `aivorah-learning-paths-service` | Curriculum/syllabus, lessons, Listen/Read Aloud, practice quiz, content pipelines (NCERT, ICSE/CISCE) | `main` → aivorah-learning-paths-service-b5bv.onrender.com |
| `aivorah-assignments-service` | Assignments, worksheet → quiz (AI) | `main` → aivorah-assignments-service-qd20.onrender.com |
| `aivorah-challenges-service` | Challenges, quiz sessions | `main` → aivorah-challenges-service-qf47.onrender.com |
| `aivorah-ai-coach-service` | AI Coach chat | `main` → aivorah-ai-coach-service-7ikx.onrender.com |
| `aivorah-schedule-service` | Calendar / events, event notifications | `main` → aivorah-schedule-service-5lxh.onrender.com |
| `aivorah-messages-service` | DMs, channels, notifications (V2) | `main` → aivorah-messages-service-9qql.onrender.com |
| `aivorah-students-service` | Teacher roster: create/link/promote/remove/bulk import, status tags | `main` → aivorah-students-service-h3se.onrender.com |
| `aivorah-teacher-dashboard-service` | Teacher dashboard, insights, student-progress heatmap | `main` → aivorah-teacher-dashboard-service-e8bh.onrender.com |
| `aivorah-attendance-service` | Attendance (teacher/parent/student), My Classes, class posts | `main` → aivorah-attendance-service.onrender.com |
| `aivorah-timetable-resources-service` | Timetables (incl. PDF import), resources | `main` → aivorah-timetable-resources-service.onrender.com |
| `aivorah-parental-controls-service` | Parental controls, subject filter, screen time | `main` → aivorah-parental-controls-service.onrender.com |
| `aivorah-school-admin-service` | School Admin: students/parents/teachers, grades & sections, bulk import, announcements | `master` → aivorah-school-admin-service.onrender.com |
| `aivorah-platform-admin-service` | Aivorah Admin: overview, users, map-to-teacher, bulk import, LLM stats | `main` → aivorah-platform-admin-service.onrender.com |
| `aivorah-help-center-service` | Help chat, support tickets | `main` → aivorah-help-center-service-tmmh.onrender.com |
| `aivorah-bedtime-stories-service` | Bedtime stories + narration | `master` → aivorah-bedtime-stories-service-vx04.onrender.com |
| `aivorah-daily-habits-service` | Daily habits, parent tasks | `master` → aivorah-daily-habits-service-rdn9.onrender.com |
| `aivorah-world-and-you-service` | Daily tidbits + quiz | `master` → aivorah-world-and-you-service-xmqw.onrender.com |

## Not on GitHub

| Folder | Note |
|---|---|
| `aivorah-school-admin-dashboard-service` | Exists only on the handover laptop — not a git repo, not deployed. Decide: push to org or delete. |

## Local layout

All repos are cloned side by side (e.g. `Desktop/<repo>`). Extra `wt-*` folders are temporary git worktrees, not separate repos.
