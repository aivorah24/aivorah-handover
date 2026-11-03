# Feature → service map

Which backend serves each part of the app, and where the screens live in `aivorah-frontend`. Checked against `src/config/*ServerUrl.ts` and the screens' imports on 2026-11-03.

"Also calls `ai-mentor`" = some calls in that feature still go to the old monolith at `aivorah.ai` (through `src/config/serverUrl.ts`). Moving those calls to the feature's own service is the way to retire the monolith.

| Feature | Frontend folder (`src/…`) | Main service | Also calls `ai-mentor` |
|---|---|---|---|
| Login, sign-up, email verify | `screens/auth/`, `context/AuthContext.tsx` | auth | Yes (verify email, password reset, role) |
| Onboarding | `screens/onboarding/` | profile | Yes (mobile OTP) |
| Profile, settings, photo | `screens/*/profile/`, `screens/*/settings/`, `screens/shared/account/` | profile | Yes (parent settings/profile, recovery email) |
| Teacher roster, parent My Kids | `screens/teacher/students/`, `screens/parent/my-kids/` | students | No |
| Parent dashboard, learning, AI report | `screens/parent/dashboard/`, `learning/`, `ai-progress-report/` | students, leaderboard | Yes (dashboard, learning) |
| Assignments | `screens/teacher/assignments/`, `screens/student/assignments/`, `screens/parent/assignments/` | assignments | Yes (quiz library, quiz preview, explain) |
| Learning paths and lessons | `screens/student/learning-paths/`, `lesson/`, `screens/teacher/learning-paths/`, `curriculum-learning-path/`, `subject-paths/` | learning-paths | Yes (curriculum lists) |
| AI Coach | `screens/student/ai-coach/`, `components/common/*AIPanel.tsx` | ai-coach | Yes (feedback, suggestions, teacher/parent AI panels) |
| Student dashboard, XP, leaderboard | `screens/student/dashboard/`, `leaderboard/`, `progress/` | leaderboard | Yes (dashboard subjects) |
| Teacher dashboard and student progress | `screens/teacher/dashboard/`, `student-progress/` | teacher-dashboard | No |
| Challenges | `screens/*/challenges/` | challenges | Yes (student quiz session) |
| Messages, channels, notifications | `screens/shared/messages/`, `features/channels/`, `context/NotificationContext.tsx` | messages | No |
| Schedule | `screens/student/schedule/`, `screens/teacher/schedule/` | schedule | Yes (student schedule) |
| Timetable and resources | `screens/shared/timetable/`, `screens/shared/resources/` | timetable-resources | No |
| Attendance and My Classes | `screens/*/attendance/`, `features/classes/` | attendance | Yes (connect to school) |
| School admin | `screens/school-admin/` | school-admin | No |
| Announcements, class portfolio | `screens/shared/announcements/`, `features/class-portfolio/` | school-admin | No |
| Aivorah (platform) admin | `screens/aivorah-admin/` | platform-admin | No |
| Parental controls, screen time | `screens/parent/parental-controls/`, `services/lockStatus.ts` | parental-controls | No |
| Daily habits, daily tasks | `screens/student/daily-habits/`, `screens/parent/daily-tasks/` | daily-habits | No |
| Bedtime stories | `screens/student/bedtime-stories/` | bedtime-stories | No |
| World and You | `screens/student/world-and-you/` | world-and-you | No |
| Help center | `screens/student/helpCenter/`, `screens/parent/help-center/`, `screens/shared/help-support/` | help-center | No |

Service addresses and branches: [repositories.md](../repositories.md). Endpoints and env vars: each service's `README.md`.
