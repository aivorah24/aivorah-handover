# Feature → service map

Which backend serves each frontend feature. Moved from an untracked `aivorah-frontend/docs/architecture/` draft (written 2026-09-21).

Rows marked ✱ were re-checked against `aivorah-frontend` `src/config/*ServerUrl.ts` on 2026-11-02. Other rows were not re-verified, especially the "Legacy Monolith Dependency" column.

| Feature | Frontend | API | Current Backend | Database | External Services | Legacy Monolith Dependency |
| ------- | -------- | --- | --------------- | -------- | ----------------- | -------------------------- |
| Authentication | Auth & Onboarding UI | `AUTH_SERVER_URL` | `aivorah-auth-service` | Supabase, Firebase | - | No |
| User Profile & Settings | Profile UI | `PROFILE_SERVER_URL` | `aivorah-profile-service` | Supabase | Cloudinary | No |
| Students & Parents | Students / My Kids UI | `STUDENTS_SERVER_URL` / `PARENT_SERVER_URL` | `aivorah-students-service` | Supabase, Firebase | Groq | No |
| Assignments | Assignments UI | `ASSIGNMENTS_SERVER_URL` | `aivorah-assignments-service` | Supabase, Qdrant | Groq, Gemini, Cloudinary | Partial (some validations/explain use monolith) |
| Challenges | Challenges UI | `CHALLENGES_SERVER_URL` | `aivorah-challenges-service` | Supabase | - | No |
| Leaderboard & Performance | Leaderboard UI | `LEADERBOARD_SERVICE_URL` | `aivorah-leaderboard-service` | Supabase, Qdrant | - | No |
| Learning Paths | Learning Paths UI | `LEARNING_PATHS_SERVER_URL` | `aivorah-learning-paths-service` | Supabase, Qdrant, Firebase | Groq, Cloudinary | Partial (Curriculum fetching still uses Monolith) |
| Messages & Channels | Messages UI | `MESSAGES_SERVER_URL` | `aivorah-messages-service` | Supabase | Groq, Cloudinary | Yes (Channels API uses `SERVER_URL`) |
| Schedule ✱ | Schedule UI | `SCHEDULE_SERVER_URL` | `aivorah-schedule-service` | Supabase | - | No |
| Timetable ✱ | Timetable UI | `TIMETABLE_RESOURCES_SERVER_URL` | `aivorah-timetable-resources-service` | Supabase | AI (PDF import) | No |
| Daily Habits | Habits UI | `DAILY_HABITS_SERVER_URL` | `aivorah-daily-habits-service` | Supabase | - | No |
| Bedtime Stories | Stories UI | `BEDTIME_STORIES_SERVER_URL` | `aivorah-bedtime-stories-service` | - | ElevenLabs, Groq | No |
| AI Coach / Ask | Ask UI | `AI_COACH_SERVER_URL` | `aivorah-ai-coach-service` | Supabase, Qdrant | Groq | No (Re-routed through `SERVER_URL` variable name but mapped to coach) |
| World and You | World & You UI | `WORLD_AND_YOU_SERVER_URL` | `aivorah-world-and-you-service` | Supabase | Groq | No (Tidbit quiz logic still uses monolith) |
| School Admin ✱ | Admin UI | `SCHOOL_ADMIN_SERVER_URL` | `aivorah-school-admin-service` | Supabase, Qdrant | Groq, Cloudinary | Yes (Some parts use Monolith `/school-admin/*`) |
| Teacher Dashboard | Teacher UI | `TEACHER_DASHBOARD_SERVER_URL` | `aivorah-teacher-dashboard-service` | Supabase, Qdrant | - | Partial |
| Parental Controls | Parent UI | `PARENTAL_CONTROLS_SERVER_URL` | `aivorah-parental-controls-service` | Supabase | - | No |
| Help Center | Help UI | `HELP_SERVICE_URL` | `aivorah-help-center-service` | - | Groq | No |
| Legacy Learning Modules (5E, Draw) | Legacy UI | `SERVER_URL` | `ai-mentor` (Monolith) | Supabase, Qdrant | - | Yes |
| Class Portfolio ✱ | Portfolio UI | `SCHOOL_ADMIN_SERVER_URL` | `aivorah-school-admin-service` | Supabase | Cloudinary | No |
| Resources ✱ | Resources UI | `TIMETABLE_RESOURCES_SERVER_URL` | `aivorah-timetable-resources-service` | Supabase | - | No |
| Attendance ✱ | Attendance UI | `ATTENDANCE_SERVER_URL` | `aivorah-attendance-service` | Supabase | - | No |
| Platform Admin ✱ | Aivorah Admin UI | `PLATFORM_ADMIN_SERVER_URL` | `aivorah-platform-admin-service` | Supabase, Firebase | Gemini | No |
