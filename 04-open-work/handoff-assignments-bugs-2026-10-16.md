# Handoff — Teacher Assignments bugs (manual test, 16 Oct 2026)

Repos (GitHub org aivorah24): `aivorah-frontend` (Expo RN web, branch base `master`),
`aivorah-assignments-service` (Node/Express on Render, base `main`).
Rules: never edit `ai-mentor` (monolith). No Claude/Co-Authored-By attribution in commits/PRs.
Show the fix locally before pushing; open PRs, user merges.

## Bugs to fix (all must be done)

1. **Generate Assignment asks for topic again** — FIXED (frontend), pushed to `wip/assignments-bug-1`.
   - Cause: AI topic used `title || chapter`, so "Chapter 5 Worksheet" beat the picked Chapter/Topic;
     the non-IB "Theme / Unit (optional)" box was never sent to the AI.
   - Fix: `hooks/useTaskSteps.ts` + `hooks/useIBGeneration.ts` → topic = `chapter || title`;
     `components/AIAssignmentFlow.tsx` shows the extra box only for IB PYP ("Unit of Inquiry").
     Test added in `__tests__/unit/useTaskSteps.test.tsx`. 116/116 pass (run `--runInBand`).
   - Path: `src/screens/teacher/assignments/`.

2. **First "Generate" tap gives only "Brainstorm"; second tap gives all 3 steps** — NOT fixed.
   - Service `routes/assignments.js` `POST /assignments/generate-steps`: Groq `openai/gpt-oss-120b`,
     `max_tokens: 500` (reasoning model → output often truncated/partial), and the loop keeps only
     step types whose label+instructions came back. So 1 of 3 steps is common.
   - Fix idea: raise max_tokens (~2000), retry once if < 3 steps, fill any missing step type
     from defaults; frontend `useTaskSteps.ts` can also backfill missing types from `STEP_DEFAULTS`.
   - Also: for non-IB boards only steps come back (no Purpose/Outcomes/Word Bank) though UI promises them.

3. **Drawing answers not graded** ("Could not fully read this submission – your teacher will review it")
   — PARTLY fixed in service, pushed to `wip/assignments-bug-3`, NOT yet tested.
   - Cause (verified): Groq model `meta-llama/llama-4-maverick-17b-128e-instruct` returns 404
     model_not_found, so every image grade fails. Also old JSON cleanup regex
     ```` /```json[\s\S]*?```|```/ ```` deleted the whole fenced JSON block.
   - Fix in progress: new `services/visionGrader.js` (`callVisionGrader(prompt, imageUrls)`:
     Gemini via `geminiModel` from `services/quizVision.js` → Mistral → Qwen HF). Wired into
     `gradeVisionSubmission`, worksheet-page image grading (~line 2745) and photo grading (~line 2837).
   - `node --check` passes (earlier error was only missing env vars when loading the module). TODO:
     update tests `tests/integration/b04-drawing-evidence.test.js` and `b04-quiz-drawing-question.test.js`
     (they mock groq for vision; mock `services/visionGrader` instead); add unit tests for visionGrader;
     test with a real drawing (Render has GEMINI_API_KEY + GEMINI_MODEL fallback list).

4. **Normal written answers not validated properly** — NOT investigated.
   - Text step grading also in `POST /assignments/:id/submit` (stepAnswers / quizAnswers branches), Groq gpt-oss-120b.
   - Screenshot: a written answer "Airforce" got 0% with generic feedback; a fully blank step showed "--".

5. **Upload Worksheet → interactive HTML: ">, < or =" (comparison) activity marks correct choices wrong** — NOT investigated.
   - Start: `src/screens/teacher/assignments/utils/quizHtml.ts` (frontend) and service quiz conversion
     (`services/quizVision.js`, `normaliseAnswerToOptionText`/`normalizeQuizQuestion` in routes/assignments.js).
     Suspect: symbol normalisation (`>` vs `&gt;` vs "greater than", `≥`/`>=`) when comparing selected vs correct.

6. **Student result/preview still uses old teal theme** (e.g. teal "Done" button) — NOT fixed.
   - Change to Aivorah gold: `colors.accentYellow` #BAA467 / `colors.accentOrange` #C9A24E (`src/theme/index.ts`).
   - Teal hex values to find: `#C3DCD8`, `#3D7A74` etc. in `src/screens/student/assignments/constants.ts`,
     `src/screens/teacher/assignments/constants.ts`, `utils/quizHtml.ts`, `AIAssignmentFlow.tsx`.

## Pushed WIP branches (no PRs)
- `aivorah-frontend` → `wip/assignments-bug-1` (commit 910ba95) — bug 1, tested locally.
- `aivorah-assignments-service` → `wip/assignments-bug-3` (commit ad4a47f) — bug 3, `node --check` passes, NOT tested.

## Which repo owns grading
Bug 3 lives in **aivorah-assignments-service**, not the monolith. The frontend's student submit
(ScaffoldedModal / QuizModal) posts to `ASSIGNMENTS_SERVER_URL` → `POST /assignments/:id/submit` in this service.
No ai-mentor change needed.
