# Runbook — Teacher Bulk Upload / worksheet → quiz

Owner repo: `aivorah24/aivorah-assignments-service` (Render, aivorah24 workspace, service `srv-d9uvv1u417fc7399o59g`, **free plan, 512 MB**).
Frontend: Teacher → Assignments → Bulk Upload (`aivorah-frontend`, `src/screens/teacher/assignments/hooks/useBulkUpload.ts`).

## How an upload works
1. Word files: `POST /assignments/docx-to-pdf` (mammoth → headless Chrome → PDF).
2. Each file: `POST /quiz/generate` (PDF) or `/quiz/generate-image` (image). The frontend sends files **one at a time**.
3. Inside `/quiz/generate` (`services/quizVision.js → generateQuizFromDocument`):
   1. **Document model** (local, no AI): text layer + page images (headless Chrome) + Tesseract OCR. Max `DOCUMENT_MAX_PAGES` pages (default 3).
   2. **Gemini** reads the whole PDF (models from `GEMINI_MODEL`, tried in order).
   3. If Gemini fails: **Mistral** vision on the page images, then **Hugging Face (Qwen)**.
   4. Last resort: OCR transcript → **Groq** text model.

## Configuration on Render (names only — values live in Render)
| Variable | Value / meaning |
|---|---|
| `GEMINI_API_KEY` | Google AI Studio key (replaced 2026-10-28; old prepaid key had no credit) |
| `GEMINI_MODEL` | `gemini-3.6-flash,gemini-3.5-flash` — `gemini-2.5-flash` is closed to new keys |
| `GEMINI_VISION_TIMEOUT_MS` | `90000` |
| `GEMINI_TIMEOUT_WITH_FALLBACK_MS` | `90000` (was 30 s; Gemini's budget is the **smaller** of these two) |
| `DOCUMENT_MAX_PAGES` | not set → default 3 (PR #18). Do not raise above 3 on the 512 MB plan |
| `MISTRAL_API_KEY` | set, but account allows 0 requests/min (plan inactive) |
| `HF_TOKEN` | set, free monthly credit used up |
| `GROQ_API_KEY` | set (free tier) |
| `POC_FALLBACK_ENABLED` | `true` — test-only `/poc` routes, not used by Bulk Upload |

## Known limitations (as of 2026-10-28)
- **Speed:** 2–3 minutes per document while Gemini is busy/unavailable (free tier often answers "overloaded"). Fast (seconds) only when Gemini responds.
- **AI availability:** Mistral inactive (0 req/min), Hugging Face out of credit, Gemini free tier overloaded at times. These are account/billing issues, not code.
- **Pages:** only the first 3 pages go through local OCR. Gemini still sees every page when it works.
- **Thin worksheets:** a 1-page worksheet with almost no text returned HTTP 200 with **0 tasks**. The UI should treat 0 tasks as a failure (pending, see below).
- **Memory:** free plan is 512 MB. Two fixes are live (#17 one Chrome at a time + 1 warm OCR worker; #18 page cap).

## Troubleshooting
| Symptom | Likely cause | Check / fix |
|---|---|---|
| Browser says "blocked by CORS" / "Failed to fetch" | Service crashed (OOM) mid-request; crash page has no CORS headers | Render → Events: `server_failed` with `oomKilled`. Lower `DOCUMENT_MAX_PAGES` or move to a 2 GB instance |
| `docx-to-pdf` 500, "Timed out … WS endpoint" | Chrome couldn't start (no memory) | Same as above |
| Every upload takes 2–3 min | Gemini failing → OCR + Groq | Render logs: `[gemini] … unavailable`, `[vision-provider] gemini: …`. Check key/billing at aistudio.google.com |
| Logs: `402 … prepayment credits are depleted` | Gemini key out of credit | New key in AI Studio, or add billing; update `GEMINI_API_KEY` |
| Logs: `models/… is no longer available` | Model retired for this key | List models (`GET https://generativelanguage.googleapis.com/v1beta/models` with the key) and update `GEMINI_MODEL` |
| Logs: `Mistral … 429` | Mistral plan inactive / rate limit | console.mistral.ai → activate "Experiment" plan or add billing |
| Logs: `HF router 402` | Hugging Face credit used up | Buy credit or HF PRO; resets monthly (small) |

## If the AI quota is exhausted — steps
1. Render → assignments service → **Logs**: find the provider error (402 / 429 / 503 / "no longer available").
2. Fix the account (top up, new key, activate plan) on the provider's site.
3. Render → **Environment** (`https://dashboard.render.com/web/srv-d9uvv1u417fc7399o59g/env`) → update the variable → **Save, rebuild, and deploy**. (Not "New Environment Group".)
4. Re-test (below).

## How to test
```bash
# Word → PDF
node -e "require('fs').writeFileSync('d.json',JSON.stringify({fileBase64:require('fs').readFileSync('tests/fixtures/tables.docx').toString('base64')}))"
curl -s -H "Content-Type: application/json" --data @d.json https://aivorah-assignments-service.onrender.com/assignments/docx-to-pdf | head -c 100
# PDF → quiz (field name is "pdf")
curl -s -F "pdf=@tests/fixtures/b02-ib-pyp-worksheet.pdf;type=application/pdf" -F "grade=Grade 4" -F "subject=Mathematics" -F "school=" https://aivorah-assignments-service.onrender.com/quiz/generate | head -c 300
```
Then Render → **Events** (no `server_failed`) and **Logs** (which provider answered).

## Pending (not done)
- **Provider circuit breaker:** skip a provider for a while after quota/overload errors instead of trying it on every upload.
- **0-task result:** frontend should show "couldn't read this worksheet" when the quiz has no tasks.
- **Store the teacher's chosen language** on assignments (no `language` column; typing language is inferred).
- **Account/billing:** Gemini paid tier or stable key, Mistral plan, Hugging Face credit — company decision.
- **Rotate the Gemini key:** it was shared in a chat on 2026-10-28.
