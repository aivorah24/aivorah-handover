# Microservices Migration Audit

> Moved from an untracked draft in `aivorah-frontend/docs/architecture/` (written 2026-09-21). Not re-verified since; check the code before relying on it.

Since then Class Portfolio, Resources and Timetable moved off the monolith (see `FEATURE_SERVICE_MAP.md`).

## Overview
This document outlines the gaps between the intended microservices architecture and the actual implementation observed in the source code.
Despite the creation of many microservices, the **legacy monolith (`ai-mentor`) remains deeply entrenched** in core application flows.

## Fully Migrated Features
The following features are correctly implemented and hit their respective microservices:
- **Authentication**: `AUTH_SERVER_URL` is fully utilized for OTP, password reset, account deletion, etc.
- **Profile / Preferences**: `PROFILE_SERVER_URL` handles avatar uploads, user preferences, and role assignments.
- **Leaderboard / Performance**: Uses `LEADERBOARD_SERVICE_URL`.
- **Challenges**: Uses `CHALLENGES_SERVER_URL`.
- **Daily Habits**: Uses `DAILY_HABITS_SERVER_URL`.
- **Messages**: Uses `MESSAGES_SERVER_URL`.
- **Parental Controls**: Uses `PARENTAL_CONTROLS_SERVER_URL`.

## Partially Migrated Features
The following features have microservices but still partially rely on the legacy monolith (`SERVER_URL`):
- **Assignments**: Uses `ASSIGNMENTS_SERVER_URL` for fetching and submitting, but uses the monolith for `/ibpyp/explain-wrong` and `/ncert/explain-wrong` in legacy screens. Some validation logic like `/assignments/validate-fill-blank` also points to the monolith in certain legacy modals.
- **School Admin**: Many dashboards use `SCHOOL_ADMIN_SERVER_URL`, but some views still call `${SERVER_URL}/school-admin/overview` and `${SERVER_URL}/school-admin/access-requests`.
- **Learning Paths & Curriculum**: Uses `LEARNING_PATHS_SERVER_URL` for `useSubjects.ts`, but the global `curriculumPrefetch.ts` and `dashboardService.ts` still query `${SERVER_URL}/curriculum`, `${SERVER_URL}/curriculum/subjects`, and `${SERVER_URL}/ncert/chapters`.

## Not Migrated (Still on Legacy Monolith)
The following features entirely rely on the `ai-mentor` monolith (`SERVER_URL`), meaning the monolith cannot be deprecated yet.
- **Schedule / Timetable**: `aivorah-schedule-service` exists, but the frontend explicitly fetches `${SERVER_URL}/schedule/upcoming` and `${SERVER_URL}/timetable`.
- **Channels**: The entire `features/channels` module hits `${SERVER_URL}/channels`.
- **Class Portfolio**: The entire `features/class-portfolio` module hits `${SERVER_URL}/class-portfolio`.
- **Resources**: The entire `features/resources` module hits `${SERVER_URL}/resources`.
- **Analytics & Tracking**: Analytics events (`/analytics/track`, `/analytics/child-report`) hit the monolith.
- **Legacy 5E Learning Screens**: `DrawingBuddyScreen`, `Learn5EScreen`, `MathHelperScreen`, `WordWizardScreen`, `ReadAloudScreen`, and `VideoPlayerScreen` directly rely on monolith endpoints (`/modules/*`, `/lesson/*`, `/read-aloud/*`).
- **Super Admin**: Features like `/admin/students`, `/admin/teachers` still point to the monolith.
- **Push Notifications**: Token registration hits `${SERVER_URL}/user/push-token`.

## Architecture Issues

### 1. Hardcoded URLs and Environment Variable Misuse
- In `services/aiMentorApi.ts`, the frontend defines an AI Mentor client that calls `${SERVER_URL}/ask`, but `SERVER_URL` is incorrectly imported from `../config/aiCoachServerUrl.ts` (meaning it actually hits the AI Coach microservice, but uses the `SERVER_URL` alias). This is confusing.

### 2. Duplicate Business Logic
- The monolith handles `/curriculum` for most grades, but Qdrant-backed Cambridge and IB curricula are intercepted and routed to `aivorah-learning-paths-service`. This splits curriculum resolution across two totally different backends depending on the student's grade.

### 3. Missing API Clients
- `aivorah-schedule-service` has no matching config in the frontend API clients (`src/config/scheduleServerUrl.ts` exists but is ignored by Dashboard services which still use `serverUrl.ts`).

## Refactoring Required

### Priority 1: Update Schedule & Timetable Service Routing
- **Issue**: Frontend `dashboardService.ts` and `timetableApi.ts` hit Monolith instead of Schedule service.
- **Required Change**: Replace imports of `serverUrl.ts` with `scheduleServerUrl.ts` in these files.
- **Risk**: Low. **Validation Required**: Verify that `aivorah-schedule-service` actually implements the endpoints.

### Priority 2: Migrate Curriculum Resolution
- **Issue**: `curriculumPrefetch.ts` splits requests between the Monolith and Learning Paths Service.
- **Required Change**: Port the remaining monolith endpoints (`/curriculum`, `/ncert/chapters`) into `aivorah-learning-paths-service` to establish single ownership of curriculum data.
- **Risk**: High. Curriculum resolution blocks dashboard rendering.

### Priority 3: Migrate Remaining Domain Features
- **Issue**: Channels, Class Portfolio, and Resources still depend on the Monolith.
- **Required Change**: Extract these into new microservices (`aivorah-channels-service`, `aivorah-portfolio-service`) or merge them into existing ones (e.g. `Messages` service).

### Priority 4: Clean up `aiMentorApi.ts`
- **Issue**: Variable shadowing in `aiMentorApi.ts`.
- **Required Change**: Rename the import to `AI_COACH_SERVER_URL` and update the file name to `aiCoachApi.ts` to reflect reality.
