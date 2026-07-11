# Tier 0 — Conversation Core: Execution Plan

> **Who this is for:** an implementing agent (any model) executing Tier 0 of [`../ROADMAP.md`](../ROADMAP.md).
> **How to use:** read §0–§2 completely before writing any code — they are the verified ground truth (traced against the actual code on 2026-07-10, file-by-file). Then execute Phases 0.A → 0.B → 0.C → 0.D in order. Each phase ends with green tests in every repo it touched, updated docs, and a separate commit per repo.
> **Why this matters:** this is the AI core of an AI-powered learning platform. If the conversation core is wrong, the whole product is wrong. Do not improvise beyond this plan; when a real ambiguity appears, prefer the smallest change that satisfies the acceptance criteria and note it in the relevant `CLAUDE.md`.

---

## 0. Ground rules (read once, obey always)

### Golden Rules (override "best practice" — non-negotiable)
1. **Never ration tokens for real students.** No quotas, caps, or paywall logic. Rate limiting exists only for bots (already implemented, generous, env-tunable).
2. **Anonymous users keep FULL access.** Reading public content and using the AI never requires login. Logged-out = no persistence only. **Never add `protect` to an AI or public-content route.** Every change must work logged-in AND logged-out (incognito).

### Product principles
- Thai-first AI output (`AI_OUTPUT_LANGUAGE`, default thai). Warm, casual, friend-tone. Tutor name: **น้องมันฝรั่ง**.
- Critical thinking > correctness. Never say "ผิด". Praise the specific good part first.
- Students are low-tech AI newbies — the app navigates them (suggestion chips, no blank intimidating textboxes).
- **Phone-first: judge every UI change at ~390 px width first.**

### Git reality ⚠️
- `client/` and `server/` are **two separate git repositories**. There is **no repo** at `Hot-Potato/` root (a stray repo exists at the user's home dir — if `git status` shows foreign files like `ChronoForge-FPGA-Engine`, you are in the wrong directory; stop).
- Always `cd client` or `cd server` before any git command. One commit per repo per phase, e.g. `feat(tier0.A): rewire client to unified tutor endpoint`.

### Commands & verification
```bash
cd server && npm install && npm run dev   # API on :5000 (nodemon)
cd client && npm install && npm run dev   # web on :5173 (vite)
cd server && npm test                     # vitest, OFFLINE (in-memory Mongo + mocked Gemini) — must stay green
cd client && npm test                     # vitest + happy-dom — must stay green
cd server && npm run test:ai              # LIVE Gemini smoke — phase boundaries only, costs tokens, needs real GEMINI_API_KEY
```
- Manual smoke: open `/view/69e39d0b60d467bd515a4945` (test lesson), test **both auth states**.
- Render free tier cold-starts; slow first response ≠ broken.
- **Definition of done per phase:** new tests for the phase's acceptance criteria written and passing, `npm test` green in every repo touched, relevant `CLAUDE.md` updated, prompt changes logged in `server/prompt-notes.md`.
- Env vars: **add none.** Model defaults stay as-is (`AI_FAST_MODEL=gemini-2.5-flash-lite`, `AI_TUTOR_MODEL=gemini-2.5-flash` — the 3-series slug re-check is Tier 3.B, not yours).

---

## 1. Mission — what Tier 0 fixes

Owner-verified problems (2026-07-10), all confirmed by code trace:

| # | Problem | Verified root cause |
| --- | --- | --- |
| P1 | Typing "สวัสดี" in a follow-up thread gets **graded as an answer** | Every follow-up goes through `requestFeedbackFollowup()` (`client/src/components/editor/extensions/questionFeedbackApi.ts:77`) which re-calls `POST /chat/feedback` with hardcoded `evaluationLevel: "almost", accuracyPercent: 0` — the legacy `buildFeedbackPrompt` frames ALL input as `=== User Answer ===` with a forced evaluation. No intent concept exists anywhere. |
| P2 | Replies show raw `**bold**` — hard to read | Model outputs markdown; every client surface renders it as plain text in a `<p>`. No renderer, and no format rules telling the model what the client can display. |
| P3 | Robotic headers leak (e.g. `**Action Plan:** …`) | Legacy prompts order "End with one actionable next step" / "suggest one concrete improvement plan" (`chat.controller.ts:105,127,180`) — the model formats that as a labeled section. Nothing strips it. |
| P4 | Tone/behavior inconsistent between question cards and the main Ask chat | **Four prompt systems live at once**: `buildLearningPrompt` + `buildFeedbackPrompt` + `buildWriteEvaluationPrompt` (legacy, still used by ALL client flows) + the new `persona.ts` (used by nobody). Context assembly differs per flow (client-truncated text vs server-side lesson). |
| P5 | Follow-up chat nests box-in-box-in-box; x-margins stack; unusable on phone | `FeedbackDiscussionPanel` = bordered box (`rounded-lg border px-3 py-2`) inside the feedback card (`border bg-violet-50 px-3 py-2`) inside the question card (`rounded-xl border p-4`), and bubbles get `max-w-[90%]` of the already-shrunken column. |

Also owner-decided: **P6** all pages stay (audit lives in ROADMAP §9 — Tier 2, not yours). **P7** student-selectable personalities come in Tier 1.C, **after** the default persona is optimized here.

**The strategy:** the good server already exists (`POST /api/chat/tutor` — persona, sessions, memory, suggestions — shipped, tested, unused). Tier 0 = rewire the whole client onto it (0.A), make output render beautifully and never leak report-speak (0.B), de-nest the chat UI for phones (0.C), then tune the single remaining context flow deliberately (0.D).

---

## 2. Verified current state (the "100% image" — trust this over guesses; re-verify against code if something looks off)

### 2.1 The unified tutor endpoint (server — EXISTS, WORKS, UNUSED BY CLIENT)

`POST /api/chat/tutor` → `tutorChat()` in `server/src/controllers/tutor.controller.ts`. Guards: `optionalAuth` + bot rate limiter (`chat.routes.ts:14,20`).

**Request body** (all validation in `tutor.controller.ts`):
```jsonc
{
  "contentId": "<mongo id>",        // REQUIRED, must be valid ObjectId → else 400
  "blockId": "<string ≤200>",       // optional; non-string coerced to ""
  "mode": "free_chat | question_feedback | write_evaluation | followup",  // REQUIRED → else 400
  "message": "<string 1..8000>",    // REQUIRED (trimmed) → else 400
  "clientThread": [                  // optional; anonymous users' history
    { "role": "student" | "tutor", "text": "<≤2000>", "createdAt": "iso?" }
  ],                                 // sanitized, last 30 kept
  "questionContext": {               // REQUIRED for question_feedback / write_evaluation → else 400
    "question": "<string, required non-empty>",
    "guideAnswer": "<string>",
    "evaluation": { "level": "correct|almost|incorrect|ai_judge", "accuracyPercent": 0-100 },
    "feedbackMode": "quick_check | full_reflection"   // default quick_check
  }
}
```
**Response:** `{ reply: string, suggestions: string[], sessionId: ObjectId | null }` (sessionId null when anonymous). Errors: 400 validation, 404 unknown content, 401/403 private-content access, 500 `MISSING_GEMINI_API_KEY`, 502 AI failure.

**Behavior pipeline** (order in `tutorChat`):
1. `getLessonContext(contentId)` (`services/lessonContext.service.ts`) — loads `Content`, serializes the **whole TipTap doc** to plain text lines + extracts every question node (`blockId`, kind, question, choices, guideAnswer, feedbackMode). Cached in-memory by `updatedAt` (max 100 entries). `QuestionAgent` nodes are excluded from the text.
2. Access check: public/link-only OK for everyone; private requires owner/collaborator.
3. History: **logged-in** → `ChatSession` keyed `(user_id, content_id, block_id)` (messages capped at 50 by pre-save hook); **anonymous** → the request's `clientThread`. Mapped to Gemini roles (`student→user`, `tutor→model`).
4. **Logged-in only:** `StudentMemory` digest (`buildMemoryDigest`) injected into the system instruction.
5. `buildSystemInstruction({ lesson, memoryDigest })` (`services/tutor/persona.ts`) — persona sections: character (warm Thai friend, specific praise required, never "ผิด"), teach rules (hint ladder, no answer-dumping), length rules per mode, output language, **`[SUGGESTIONS]` STRICT format contract** (3 short follow-up questions in the student's voice), optional memory/teacherNote slots, then **the full lesson text + all teacher questions with guide answers** ("never reveal a guide answer verbatim before the student attempts").
6. For feedback modes only, the final user turn is wrapped by `buildFeedbackUserTurn(...)` — starts with the exact tag `[งานของเธอตอนนี้: ให้ฟีดแบ็กคำตอบของเพื่อน]`, includes question, guide answer, `ผลการตรวจ` (either `level=X, accuracy=Y% — ต้องยึดตามนี้` or, for `ai_judge`, "ให้เธอประเมินเองอย่างใจดี เน้นคุณภาพการคิด"), `โหมดความยาว`, and the student's answer. **For `free_chat`/`followup` the message passes through untouched.**
7. Model routing (`resolveTutorModel`): `question_feedback` + `quick_check` → `AI_FAST_MODEL`; everything else → `AI_TUTOR_MODEL`.
8. `callTutorModel` (`services/tutor/genaiClient.ts`) — `@google/genai`, multi-turn `contents`, `config.systemInstruction`, wrapped in `generateWithRetry` (`services/tutor/retry.ts` — retries transient network errors because the owner's ISP drops connections; **never remove**).
9. `parseTutorReply` (`services/tutor/parse.ts`) — splits at the **last** `[SUGGESTIONS]` line; collects up to 3 `- ` lines (≤120 chars each).
10. Logged-in: append student+tutor messages (with `mode`) to the session, save; fire-and-forget `runMemoryUpdate` every `AI_MEMORY_EVERY_N_TURNS` (default 3) student turns, or always after `write_evaluation`.

### 2.2 The legacy endpoints (server — WHAT THE CLIENT ACTUALLY USES TODAY; to be deleted in 0.A)

All in `server/src/controllers/chat.controller.ts`, routes in `chat.routes.ts:17-19`:
- `POST /chat/ask` → `buildLearningPrompt(prompt, context, userContext)` — client sends the lesson text itself; single-shot, no persona, no suggestions.
- `POST /chat/feedback` → `buildFeedbackPrompt(...)` — forced evaluation level; "End with one actionable next step" (P3's origin).
- `POST /chat/write-evaluate` → `buildWriteEvaluationPrompt(...)` — separate long-form prompt.

### 2.3 The five client AI surfaces (current wiring — every one must move to `/chat/tutor`)

The bridge is `client/src/components/editor/extensions/questionFeedbackApi.ts` (throws `AiUnavailableError`; callers render `AiErrorRetry` with Thai copy).

| # | Surface | File | Today | Target mode (0.A) |
| --- | --- | --- | --- | --- |
| 1 | Choice card, first submit | `QuestionChoiceView.tsx` `handleSubmit` (~line 427) — computes `evaluateChoiceAnswer(choices, selectedIndices)` → `{accuracyPercent, evaluationLevel, missedCorrect, wrongSelected, correctAnswer, userAnswer}` then `requestQuestionFeedback` | `/chat/feedback` | `question_feedback` (keep client-computed deterministic level) |
| 2 | Blank-choice card, first submit | `QuestionBlankChoiceView.tsx` `handleSubmit` (~line 453) — computes level/accuracy + per-blank diagnostics `[Q-n] expected=… got=…` | `/chat/feedback` | `question_feedback` (client-computed level) |
| 3 | Blank-write card, first submit | `QuestionBlankWriteView.tsx` `handleSubmit` (~line 407) — **hardcodes** `evaluationLevel:"almost", accuracyPercent:0` (a hack; typed answers can't be exact-matched fairly) | `/chat/feedback` | `question_feedback` with `level: "ai_judge"` |
| 4 | Write card, first submit | `QuestionWriteView.tsx` `handleSubmit` (~line 192) → `requestWriteEvaluation({question, guideAnswer: answer, studentAnswer: input, feedbackMode})` | `/chat/write-evaluate` | `write_evaluation` |
| — | Follow-up thread on cards 1–4 | `FeedbackDiscussionPanel.tsx` (pure UI; parents own `handleSendThreadMessage`) → `requestFeedbackFollowup` (the P1 hack) | `/chat/feedback` | `followup` — plain conversation turn, NO evaluation payload |
| 5a | Ask-AI modal (lesson-wide FAB) | `TiptapViewer.tsx` `handleAsk` (~line 246) — local `askAi()` posts `{prompt, context: <client-serialized lesson + viewport>, userContext: <all block chat histories>}`. **On failure returns a fake English fallback string that gets saved into history** (`buildFallbackReply`, line 80) | `/chat/ask` | `free_chat` |
| 5b | Embedded Ask-AI block | `QuestionAgentView.tsx` `handleAsk` (~line 137) — own local `askAi()` + own duplicate `AiUnavailableError` class; context = lesson text **above the block** (`getQuestionAgentContextAbove`) | `/chat/ask` | `free_chat` |

Verified: **nothing in `client/src` references `/chat/tutor` today** (grep = 0 hits).

### 2.4 Client-side state & persistence (must stay shape-compatible)

- `content-answer.store.ts` (Zustand, **no persist middleware** — anonymous state is in-memory per SPA session; that is current, accepted behavior). `setAnswer(blockId, answer)` = instant local write; `syncAnswers()` = `PUT /content-answer/:contentId/bulk` — **called only when logged in** (`TiptapView.tsx:24,30-43`: 30 s interval + beforeunload + visibility-stale reload, all gated on `token`).
- Per-block answer shapes (stored in `UserContent.answers: Map<blockId, any>` — **breaking these breaks saved student work**):
  - Cards 1–4: `{ selected/input/…, submitted, aiFeedback?, feedbackThread?: {role:"student"|"ai", text, createdAt}[], threadOpen? }` — note thread role is **`"ai"` client-side vs `"tutor"` server-side** (mapping needed).
  - QuestionAgent block: `{ chatHistory: {question, answer, createdAt}[], collapsed }`.
  - Ask-AI modal uses pseudo-block `LESSON_AI_BLOCK_ID = "__lesson_ai_assistant__"`: `{ chatHistory, open }`.
- `client/src/lib/axios.ts` attaches JWT and parses the structured 401 (`forceRelogin`/`clearToken`) — don't disturb.

### 2.5 Tests today (what you extend / what you delete)

- **Server** (`server/test/`, vitest + supertest + in-memory Mongo; Gemini mocked via `vi.hoisted` + `vi.mock("@google/genai")` returning `{ text: "MOCK_AI_REPLY" }` — copy this pattern from `test/api/tutor.test.ts:5-13`):
  - `api/tutor.test.ts` (the unified endpoint — extend), `api/chat.test.ts` + `unit/prompts.test.ts` + `unit/__snapshots__/prompts.test.ts.snap` (characterization tests **pinning the legacy endpoints/prompts — delete in 0.A step 5**), `unit/persona.test.ts` (persona snapshot-phrase tests — extend), `unit/parse.test.ts` (extend in 0.B), `unit/lessonContext.test.ts`, `unit/retry.test.ts`, `unit/tutorModel.test.ts`, `unit/memoryDigest.test.ts`, `api/memory.test.ts`, `api/rateLimit.test.ts`, `api/auth|content|answers.test.ts`, `ai/live.ai.test.ts` (live smoke).
  - Fixture lesson: `test/fixtures/lesson.json`, seeded via `test/helpers/seed.ts` (`seedUser`, `seedLesson`).
- **Client** (`client/src/components/editor/extensions/__tests__/`): `questionEvaluation.test.ts` (keep — deterministic choice math stays), `questionAgentContext.test.ts` (keep parts used), `questionFeedbackApi.test.ts` (**replace** with `tutorApi.test.ts`).

### 2.6 Docs that must stay truthful
`../CLAUDE.md`, `server/CLAUDE.md` (API table + AI section), `client/CLAUDE.md` ("How questions reach the AI" section), `../asking-flow.md` (describes the legacy wiring — rewritten in 0.D), `server/prompt-notes.md` (append-only log; every persona edit requires an entry).

---

## 3. Phase 0.A — Client rewire to `/api/chat/tutor` (kills P1, most of P4)

**Goal:** all five surfaces speak to the unified tutor with proper modes; small talk never gets graded; the legacy prompt zoo dies. Repos: **both**. Do the server prep first, then client, then retirement — in that order, testing at each step.

### Step A1 — Server: intent-aware persona rule

In `server/src/services/tutor/persona.ts`, add a new section to `buildSystemInstruction` (place it right after "== HOW YOU TEACH ==")::

```
"== เมื่อไหร่ประเมิน เมื่อไหร่คุยเฉยๆ (สำคัญมาก) ==",
"- You evaluate an answer ONLY when the latest message starts with the tag \"[งานของเธอตอนนี้: ให้ฟีดแบ็กคำตอบของเพื่อน]\". Only the answer inside that message is evaluated.",
"- EVERY other message — greetings (\"สวัสดี\", \"หวัดดี\"), jokes, small talk, questions about you, meta-questions, follow-ups after feedback — is normal friend conversation. Reply warmly and naturally. NEVER treat it as an answer, NEVER give an evaluation, NEVER restate the question rubric.",
"- In a follow-up thread the student may send a NEW attempt at the answer. Coach it conversationally (what improved, what's still missing) — do not produce a formal verdict.",
```
(Adjust wording freely for prompt quality, but the two behaviors — tag-gated evaluation, chit-chat never graded — must be explicit.)

**Tests:** extend `test/unit/persona.test.ts` with snapshot-phrase assertions that the rule text survives refactors (assert `buildSystemInstruction` output contains e.g. `"[งานของเธอตอนนี้"`-gating sentence and the "NEVER treat it as an answer" clause — same trick the file already uses). Log the persona change in `server/prompt-notes.md`.

### Step A2 — Server: two small contract extensions

In `tutor.controller.ts` (+ types):
1. **`currentSection`** (optional string, trim, cap 500 chars) on the request body. When present, append to the **final user turn** (after `buildFeedbackUserTurn` wrapping, all modes):
   `\n\n[ตอนนี้เพื่อนกำลังอ่าน/อยู่แถวส่วนนี้ของบทเรียน: {currentSection}]`
2. **`questionContext.evaluation.diagnostics`** (optional string, cap 500). Pass into `buildFeedbackUserTurn`; when non-empty append a line `รายละเอียดการตรวจ (ระบบตรวจให้แล้ว): {diagnostics}` after the `ผลการตรวจ` line. This carries the deterministic per-blank/per-choice info (`missedCorrect=…`, `[Q-n] expected=… got=…`) the legacy flow used to send.

**Tests:** extend `test/api/tutor.test.ts` — currentSection appears in the last `contents` entry; diagnostics appears only in feedback modes; both optional (absent = unchanged behavior).

### Step A3 — Client: create the typed bridge `tutorApi.ts`

New file `client/src/components/editor/extensions/tutorApi.ts` (keep `AiUnavailableError` exported from here; delete `questionFeedbackApi.ts` in step A5):

```ts
export type TutorMode = "free_chat" | "question_feedback" | "write_evaluation" | "followup";
export interface TutorThreadEntry { role: "student" | "tutor"; text: string; createdAt?: string }
export interface TutorRequest {
  contentId: string; blockId: string; mode: TutorMode; message: string;
  clientThread?: TutorThreadEntry[];
  currentSection?: string;
  questionContext?: {
    question: string; guideAnswer?: string;
    evaluation?: { level: "correct"|"almost"|"incorrect"|"ai_judge"; accuracyPercent?: number; diagnostics?: string };
    feedbackMode?: QuestionFeedbackMode;
  };
}
export interface TutorResponse { reply: string; suggestions: string[]; sessionId: string | null }
export async function callTutor(req: TutorRequest): Promise<TutorResponse>  // POST /chat/tutor via "@/lib/axios"; empty reply or axios error → throw AiUnavailableError
```

Helper in the same file: `toClientThread(...)` — maps stored shapes to `TutorThreadEntry[]`:
- feedback-card threads: prepend the original exchange so anonymous followups have full context → `[{role:"student", text: <original student answer>}, {role:"tutor", text: <aiFeedback>}, ...feedbackThread.map(m => ({role: m.role === "ai" ? "tutor" : "student", text: m.text, createdAt: m.createdAt}))]`
- Q&A histories (`{question, answer}` pairs): flatten each pair to student + tutor entries.

**contentId source:** `useCanvasStore` exposes `contentId` (see `TiptapViewer.tsx:110`). Question views don't receive it as a prop — read it from the store the same way (`useCanvasStore((s) => s.contentId)`). If it is null (shouldn't happen inside `/view/:id`), throw `AiUnavailableError` rather than sending garbage.

**Tests:** new `__tests__/tutorApi.test.ts` mirroring the axios-mock style of the old `questionFeedbackApi.test.ts` (mock `@/lib/axios`, assert URL/payload/roles mapping/error → `AiUnavailableError`, assert the ai→tutor role rename and history flattening).

### Step A4 — Client: rewire all five surfaces (per-surface spec)

Common rules for all surfaces:
- Send `blockId` = the block's TipTap node id (`attrs.id`); the Ask-AI modal keeps its pseudo-id `"__lesson_ai_assistant__"` (deliberate deviation from the roadmap's `blockId: null` — it keeps a separate `ChatSession` row per surface and matches the existing store key; the server accepts any string ≤200).
- Always send `clientThread` built from the locally stored history (cheap; the server ignores it for logged-in users and it is the ONLY history for anonymous users). Keep local storage shapes unchanged (see §2.4) — additive fields only.
- Store `suggestions` from every response additively on the block answer (`suggestions?: string[]`) — 0.C renders them; harmless in the Map.
- **Stop sending lesson text.** The server owns context now. Delete client calls to `getQuestionAgentContextFromEditor` / `getQuestionAgentContextAbove` / `buildQuestionAgentUserContext` for AI purposes. Keep `getQuestionAgentViewportContext(mainRef.current)` ONLY in `TiptapViewer` — trim its output to the first ~300 chars and send as `currentSection`.
- Keep every `AiErrorRetry` path exactly as it behaves today (retry re-invokes the same send).

Surface specifics:

1. **`QuestionChoiceView.tsx` `handleSubmit`:** `mode: "question_feedback"`, `message: userAnswer` (the joined selected-choice text from `evaluateChoiceAnswer` — if empty, fall back to `"(ไม่ได้เลือกคำตอบ)"`), `questionContext: { question, guideAnswer: correctAnswer, evaluation: { level: evaluationLevel, accuracyPercent, diagnostics: "missedCorrect=…; wrongSelected=…" }, feedbackMode }`. Reply → `aiFeedback` (unchanged flow).
2. **`QuestionBlankChoiceView.tsx` `handleSubmit`:** same as #1 with its computed level/accuracy and its per-blank diagnostics string.
3. **`QuestionBlankWriteView.tsx` `handleSubmit`:** `mode: "question_feedback"`, `evaluation: { level: "ai_judge", diagnostics: <the per-blank template/expected string it already builds> }` — replaces the hardcoded `"almost"` hack. `message` = the joined `[Q-n] = value` user answer it already builds.
4. **`QuestionWriteView.tsx` `handleSubmit`:** `mode: "write_evaluation"`, `message: input` (student text), `questionContext: { question, guideAnswer: answer, feedbackMode }` (no evaluation object — server defaults to `ai_judge`).
5. **Follow-up threads (all four cards' `handleSendThreadMessage`):** `mode: "followup"`, `message` = exactly what the student typed (no wrapping, no topic/initialFeedback smuggling), `clientThread` = `toClientThread` with the prepended original exchange, same `blockId` as the first submit (so the logged-in `ChatSession` continues seamlessly — the first submit already wrote to the same session). Delete every `requestFeedbackFollowup` call site.
6. **`TiptapViewer.tsx` (Ask-AI modal):** replace local `askAi`/`buildFallbackReply` with `callTutor({ mode: "free_chat", blockId: LESSON_AI_BLOCK_ID, message: question, clientThread: toClientThread(chatHistory), currentSection })`. **Fix the fallback bug while here:** on `AiUnavailableError` show an inline retry state (reuse `AiErrorRetry`) instead of saving a fake English reply into history. Keep `{question, answer}` history shape.
7. **`QuestionAgentView.tsx`:** replace its private `askAi` + duplicate `AiUnavailableError` with `callTutor({ mode: "free_chat", blockId, message, clientThread: toClientThread(chatHistory) })`. Keep collapse/clear behavior.

### Step A5 — Retire the legacy endpoints (server + client, only after A4 is green)

- Client: delete `questionFeedbackApi.ts` and `__tests__/questionFeedbackApi.test.ts`; grep `client/src` for `chat/ask|chat/feedback|chat/write-evaluate|requestQuestionFeedback|requestWriteEvaluation|requestFeedbackFollowup` → must be 0 hits.
- Server: remove `/ask`, `/feedback`, `/write-evaluate` from `chat.routes.ts`; delete `askChat`, `askFeedback`, `askWriteEvaluation` and the three `build*Prompt` builders from `chat.controller.ts` (keep the `generateWithRetry`/`isTransientAIError` re-exports only if something still imports from there — check `rateLimit` tests and `ai/live.ai.test.ts`; prefer moving imports to `services/tutor/retry`). Delete `test/api/chat.test.ts`, `test/unit/prompts.test.ts`, `test/unit/__snapshots__/prompts.test.ts.snap`. Update `test/ai/live.ai.test.ts` if it smokes legacy endpoints (point it at `/chat/tutor` modes instead).
- Docs: update `server/CLAUDE.md` API table (+AI section) and `client/CLAUDE.md` ("How questions reach the AI").

### Phase 0.A acceptance (all must pass)
- [ ] Typing "สวัสดี" in a card follow-up thread AND in the Ask-AI modal gets a friendly greeting, zero evaluation language — logged-in and incognito. (Manual, owner-judged.)
- [ ] A 5+ turn conversation on one card remembers turn 1 — logged-in (via ChatSession) AND anonymous (via clientThread).
- [ ] All five surfaces work on `/view/69e39d0b60d467bd515a4945` in both auth states; choice cards still show deterministic correct/incorrect marks (client math untouched).
- [ ] Legacy endpoints return 404; grep confirms no client references.
- [ ] `cd server && npm test` and `cd client && npm test` green; new tests cover: tutorApi payloads/role-mapping, currentSection/diagnostics passthrough, persona intent-rule phrases.
- [ ] `server/prompt-notes.md` entry appended; both `CLAUDE.md`s updated.

---

## 4. Phase 0.B — Markdown pipeline & output-format discipline (kills P2 + P3)

**Goal:** replies render beautifully; no robotic section headers ever leak. Repos: **both**. Depends on 0.A.

### B1 — Client: markdown renderer
- Add `react-markdown` + `remark-gfm` to `client` (this is the default choice; a comparably light alternative is fine if bundle size proves problematic — do not hand-roll a parser).
- New shared component `client/src/components/editor/extensions/MarkdownMessage.tsx`: renders tutor text with an allowlist — paragraphs, `strong`, `em`, `ul/ol/li`, inline `code`, `pre`, `blockquote`, links (target=_blank, rel=noopener). **No raw HTML, no images, no headings** (map `h1–h6` to `p` with bold styling). Match current bubble typography (`text-sm`/`text-base`, inherits bubble colors); style via a small prose class in `indexTiptap.css` or inline components, consistent in dark mode (bubbles currently use explicit grays — keep readable).
- Use it for **tutor/AI text only** (student text stays plain) in all surfaces: `FeedbackDiscussionPanel` ai bubbles, the `aiFeedback` block in all four card views, `QuestionAgentView` answer bubbles, `TiptapViewer` modal answer bubbles.
- Tests: a small render test for `MarkdownMessage` (bold renders as `<strong>`, raw `<script>`/HTML is not injected, headings downgraded).

### B2 — Server: format contract in the persona
Append to `buildSystemInstruction` (near LENGTH RULES):
```
"== FORMAT (STRICT) ==",
"- You may use light markdown: **bold** for the key word, short lists (max ~4 items) when truly listing, `code` for code/formulas.",
"- You are texting a friend, NOT writing a report: NEVER use labeled sections or headers like 'Action Plan:', 'Feedback:', 'สรุป:', 'คำแนะนำ:', 'ขั้นตอนต่อไป:', 'Next steps:' — weave advice into the conversation instead.",
"- No markdown headings (#), no tables, no horizontal rules.",
```
Persona snapshot tests updated (assert the "NEVER use labeled sections" phrase). `prompt-notes.md` entry.

### B3 — Server: parse guard (defense in depth)
In `services/tutor/parse.ts`, after the `[SUGGESTIONS]` split, strip leaked report-labels from the reply: for each paragraph-leading line matching `/^\s*(\*\*)?\s*(action plan|next steps?|feedback|summary|สรุป|แผน(การ)?|ขั้นตอนต่อไป|คำแนะนำ(เพิ่มเติม)?)\s*:?\s*(\*\*)?\s*:?\s*/i` — remove the label but keep the sentence content that follows. Do NOT touch legitimate bold mid-sentence. Keep the list of labels as an exported constant so tests pin it. Extend `test/unit/parse.test.ts` with: label stripped at reply start, label stripped at paragraph start mid-reply, legit `**bold**` untouched, Thai labels covered.

### Phase 0.B acceptance
- [ ] A `full_reflection` reply renders formatted on the client (no raw `**`), in all six render sites.
- [ ] 10 manual Thai turns across card + Ask flows produce zero "Action Plan"-style labels (owner-judged).
- [ ] Parse-guard unit tests green; persona snapshot tests green; `npm test` green both repos; prompt-notes entry written.

---

## 5. Phase 0.C — Phone-first chat UI (kills P5). Repo: **client** only. Depends on 0.A.

### C1 — De-nest the follow-up thread
Redesign `FeedbackDiscussionPanel.tsx` (parents in the four card views pass the same props — change is mostly internal):
- Kill the box-in-box: the panel loses its own `rounded-lg border bg-white px-3 py-2` wrapper; the messages list loses the inner `border bg-gray-50 p-2` box. The thread becomes a flat, full-width block under the feedback card separated by a hairline (`border-t`) only — ONE visual nesting level inside the question card, total.
- Let bubbles escape the card padding: apply negative horizontal margins (e.g. `-mx-4` matching the question card's `p-4`) plus small inner padding, or restructure the parent so the thread renders as a sibling section of the card body. Bubbles: `max-w-[85%]` of the **full card width**, student right / tutor left, keep current colors.
- Input row pinned at the thread's bottom: textarea + send button with **≥44 px** touch height (`h-11`), full width.
- Long threads (> ~60vh) scroll inside the thread block, newest visible.
- Optional stretch (only if trivial): on `max-width: 480px` open the thread as a bottom sheet (fixed, `bottom-0`, `max-h-[85dvh]`); skip if it risks the schedule — flattening is the required outcome.

### C2 — Suggestion chips (the "navigator" behavior)
- New `SuggestionChips.tsx`: renders `suggestions: string[]` as tappable pill buttons (wrap in a flex-wrap row, ≥44 px hit area, violet outline style consistent with existing buttons).
- Tapping a chip **sends it immediately** as the student message in that surface (simplest consistent behavior; it disappears once a new reply arrives — always render only the latest reply's suggestions).
- Wire into: follow-up threads (under latest tutor bubble), `QuestionAgentView`, `TiptapViewer` modal, and under the first `aiFeedback` on cards (tapping there opens the thread and sends). Suggestions come from the stored `suggestions` field (0.A already saves them).
- Empty `suggestions` → render nothing (server may return none; never show placeholder chips).

### C3 — 390 px audit
On a 390 px viewport, walk `/view/69e39d0b60d467bd515a4945`: input rows, send buttons (≥44 px), Ask-AI modal margins (`p-1.5` outer is fine; verify inner paddings), card paddings, chips wrap correctly, **no horizontal scroll anywhere** (check `document.documentElement.scrollWidth <= innerWidth` on each state). Note: the viewer scales content by CSS `transform: scale()` (`TiptapViewer.tsx` zoom math) — test with real devtools device emulation, not just a narrow window.

### Phase 0.C acceptance
- [ ] At 390 px: thread bubbles use ≥85% of the card's usable width with a single visual nesting level; no horizontal scroll in `/view/:id`.
- [ ] Suggestion chips tappable in every flow; chip tap sends and gets a reply.
- [ ] `cd client && npm test` green (add a render test: panel has no nested-border DOM anymore; chips render from props and fire onSend).
- [ ] Owner eyeballs on a real phone before sign-off (leave this as the final unchecked box for the owner).

---

## 6. Phase 0.D — Context-flow & prompt consistency pass (rest of P4). Repos: **both** (mostly server prompts + docs). Depends on 0.A–0.C.

This phase is deliberately owner-in-the-loop; the agent prepares, measures, and proposes — the owner judges tone.

1. **Trace & document the final context flow** — write `../asking-flow.md` fresh (replace the legacy doc; keep the filename): per mode (`free_chat`, `question_feedback`, `write_evaluation`, `followup`) — what the system instruction contains, what the final user turn contains, what history is loaded (session vs clientThread), and measured sizes: log `systemInstruction.length`, history char total, and reply length for one real call per mode against the test lesson (a temporary `AI_DEBUG_PROMPTS=true` run locally is fine — never commit debug output).
2. **Tune deliberately, smallest-change-first, one experiment at a time:**
   - Does `question_feedback` need the whole lesson text, or the question + guide answer + nearby section? (The full text is already in the system instruction via `lessonContext`; if lessons are long this is token waste and focus dilution. Consider a `lessonScope` option in `buildSystemInstruction` — `"full" | "question_focus"` — and use `question_focus` for `question_feedback`/`followup` on question blocks: include title + the question's own entry + ±~1500 chars of surrounding lesson text. Requires mapping blockId → position in `lessonContext.service.ts`; keep `full` for `free_chat`.)
   - Does `free_chat` benefit from a recent-answers digest (what the student answered on other blocks)? Only add if a transcript shows the tutor being blind to something it should know; otherwise skip — cheaper and more focused wins.
   - Suggestions quality: are the 3 chips fun and in the student's voice? Tweak the `[SUGGESTIONS]` instruction if they read like teacher homework.
3. **Prompt-quality loop with the owner:** at least one 10-turn Thai transcript per flow (4 flows), pasted (1–2 line excerpts) into `server/prompt-notes.md` with what felt off + the tweak. Target feel: **"อยากคุยต่อ"** — playful, curious, zero report-speak. Iterate until the owner signs off.
4. Keep every change pinned by the persona snapshot tests (update phrases when wording changes — that is their job).
5. Final doc sweep: `server/CLAUDE.md` AI section + `client/CLAUDE.md` reflect the single-endpoint reality; root `CLAUDE.md`'s "current focus" line updated to point at Tier 1.

### Phase 0.D acceptance
- [ ] `asking-flow.md` rewritten and accurate (a reader can predict the exact prompt for each mode).
- [ ] Token-budget decisions written down (even the "no change" ones, with the measured numbers).
- [ ] 4 × 10-turn transcripts logged in `prompt-notes.md`; owner signs off that tone/behavior feel consistent and fun across every card type + main Ask (owner-judged — leave unchecked for the owner).
- [ ] `npm test` green both repos.

---

## 7. Careful-not-to-break (regression tripwires)

| Thing | Where | Rule |
| --- | --- | --- |
| Anonymous full AI access | `optionalAuth` on `/api/chat/*`, client guards | Golden Rule 2 — adding `protect` to an AI path = regression. `/memory` stays the only protected chat route. |
| Structured 401 contract | `auth.middleware.ts` ↔ `client/src/lib/axios.ts` | Client parses exact `{ forceRelogin, clearToken }` shape. |
| `UserContent.answers` Map shape | server model ↔ all card views | Additive fields only (`suggestions?` OK). Renaming/removing stored keys breaks saved student work. |
| `generateWithRetry` | `services/tutor/retry.ts` | Owner's ISP drops connections. Port it everywhere; never delete. |
| Error handler order | `server/src/app.ts` | `app.use(errorHandler)` stays last. |
| `[SUGGESTIONS]` contract | `persona.ts` ↔ `parse.ts` ↔ (0.C) chips | Change format only with both sides + tests in the same commit. |
| ChatSession 50-message cap | `chat_session.model.ts` pre-save hook | Long threads must keep working when truncation kicks in (server loads capped history — fine; don't "fix" it). |
| TipTap node = source of truth | editor internals | Tier 0 does NOT touch editor node schemas. Question node attrs are read-only to you. |
| Rate limiter | `rateLimit.middleware.ts` | Bot-only, generous. Do not lower thresholds. |
| Optimistic concurrency | `PUT /content/:id` | Untouched by Tier 0 — if you find yourself editing it, you've drifted out of scope. |

## 8. Explicitly OUT of scope for Tier 0
SSE streaming (1.A), teacher agent settings (1.B), student personalities (1.C — P7), Profile page (2.A), login UX (2.B), page polish/`Index.tsx` deletion (2.C), CORS lockdown (3.A), model slug upgrades (3.B), any dopamine/celebration UI (Tier 4), new env vars, per-student anything that smells like a quota.
