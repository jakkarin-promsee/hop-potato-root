# GLOSSARY.md — the vocabulary the codebase assumes

> Updated 2026-07-11. Every term the code uses without explaining. Alphabetical. Each entry: what it means · where it lives. When you meet an unfamiliar word in the code, it should be here — if it isn't, add it.

### access_type
A lesson's visibility: **`public`** (listed in Explore, anyone reads), **`link-only`** (unlisted, anyone with the link reads), **`private`** (owner/collaborators only). Field on `Content`; enforced in `content.controller` on load/search. Default `private`.

### agent_settings
Per-lesson teacher controls for the AI tutor, stored on `Content.agent_settings`: `persona_note`, `allow_direct_answers` (bool), `scope` (`lesson_only` | `lesson_plus_general`), `custom_guidelines`. Injected into the tutor's system instruction *above* the student's personality (teacher wins). Set in the Publish modal; normalized in `lessonContext.service` and `canvas.store`.

### ai_judge
An evaluation `level` meaning "let the model decide leniently" instead of a client-computed verdict. Used for open-ended answers (write questions, blank-write) where there's no deterministic right/wrong. See **diagnostics**, **level**.

### AppLayout
The shared UI chrome (`TopNav` + `<Outlet/>`) wrapping the "app" routes (`/`, `/login`, `/explore`, `/profile`, `/settings`, …). The full-bleed routes (`/canvas/:id`, `/view/:id`, `/status`) render **outside** it. See [`ui-structure.md`](ui-structure.md).

### block / block id
A node in the TipTap document. Question nodes and the Ask-AI node carry a stable `id` attr ("block id") used to key answers (`UserContent.answers[blockId]`) and tutor sessions (`ChatSession.block_id`). The lesson-wide Ask-AI modal uses the pseudo-id `"__lesson_ai_assistant__"`.

### ChatSession
Server model: one tutor conversation per **(user, content, block)**. Capped at 50 messages (pre-save hook). Logged-in only — anonymous users have no ChatSession and pass **clientThread** instead.

### clientThread
The last ~30 messages of a conversation, sent by the client on every `/chat/tutor` request. The server uses it **only when the user is anonymous** (logged-in users get their server-side `ChatSession` instead). This is how Golden Rule 2 keeps AI context for logged-out users.

### cold start
The multi-second delay on the first request after Render's free tier has idled the server to sleep. Adds to AI latency. Handled with loading UI (`useColdStartHint`), never treated as an error.

### creator (copilot) / creatorApi
The **teacher-side** AI. One endpoint `POST /api/creator/assist` with 11 actions; the single client bridge is `lib/creatorApi.ts` (`callCreator`). Teacher-only (`protect` + owner check). Distinct from the **tutor** (student AI). See [`data-flow.md`](data-flow.md) §teacher copilot.

### denormalized author snapshot
`Content.author_name` / `collaborator_names[]` are *copies* of user display names stored on the lesson so lists render without a join. Kept in sync by `contentAuthorSnapshot.service`; a profile rename propagates via `refreshAuthorSnapshotsForUser`. `updateContent` deliberately ignores client-sent values for these.

### diagnostics
Deterministic per-choice / per-blank check results the client computes (`questionEvaluation.ts`) and passes in `questionContext.evaluation.diagnostics` for feedback modes — e.g. `[Q-2] expected=… got=…`, missed-correct / wrong-selected. Lets the tutor give precise feedback without re-grading.

### feedbackMode
How verbose a question's feedback is: **`quick_check`** (2–4 sentences, routes to the fast model) or **`full_reflection`** (2–4 paragraphs of coaching). A per-question attr; see `questionMode.ts`.

### Golden Rules
The two non-negotiables that override "best practice": **(1)** never ration tokens for real students; **(2)** anonymous users keep full access. See [`ARCHITECTURE.md`](ARCHITECTURE.md) §3 and root [`CLAUDE.md`](../CLAUDE.md).

### guide answer
The teacher's reference answer for a question (`QuestionWrite.answer`, or derived from the correct choices for choice types). Fed to the tutor so it can coach toward the intended idea. The copilot's `guide_answer` action drafts one.

### distractor
A deliberately-wrong multiple-choice option that reflects a *common misconception* (not a trick). The copilot's `distractors` action suggests them; appended as `correct: false`.

### lessonContext
The server's serialized view of a lesson (plain text + a question list + `agent_settings`), produced by `lessonContext.service.ts` and cached by `updatedAt`. This — not client-sent text — is what the AI "reads." See **② server owns lesson context** in [`ARCHITECTURE.md`](ARCHITECTURE.md).

### level (evaluation level)
The verdict on an answer: `correct` | `almost` | `incorrect` | `ai_judge`. For choice/blank questions the **client** computes it deterministically and the tutor must respect it; `ai_judge` hands judgment to the model. Carried in `questionContext.evaluation.level`.

### loanword rule
Tone rule (owner, 2026-07-11): every English-borrowed technical term is written in Thai with the English in parentheses on first mention — `อิเล็กตรอน (electron)` — never bare English, never dropped. Lives in `persona.ts` (`loanwordRule()`) and the creator prompts. Logged in `prompt-notes.md`.

### mode (tutor mode)
Which conversation shape a `/chat/tutor` call is: **`free_chat`** (open Q&A, message passes through raw), **`question_feedback`** (grade + coach a question answer), **`write_evaluation`** (grade an open-ended written answer), **`followup`** (a thread turn, raw message, *no* evaluation — this is why "สวัสดี" in a thread isn't graded). Chosen per surface by `tutorApi.ts`.

### น้องมันฝรั่ง (Nong Man Farang)
The tutor's persona name ("little potato"). Warm, coaching, never-shaming Thai character defined in `services/tutor/persona.ts`. The tone is owner-gated — changes must be logged in `prompt-notes.md`.

### optimistic concurrency / clientUpdatedAt / 409
The lesson-save safety net. The editor sends the `updatedAt` timestamp it loaded (`clientUpdatedAt`) with each `PUT /content/:id`; if the server copy is newer, it returns **409** and the client flags a conflict instead of clobbering. `forceSave()` omits the timestamp to overwrite. See [`data-flow.md`](data-flow.md) §lesson save.

### optionalAuth vs protect
Two auth middlewares. **`optionalAuth`** sets `req.user` if a valid token is present but never rejects (anonymous still works) — used on AI + public-read routes (Golden Rule 2). **`protect`** requires a valid JWT or returns a structured 401. `restrictTo(...roles)` runs after `protect` for admin gating.

### personality (tutor preset)
One of 6 student-selectable tutor coaching styles (`services/tutor/personality.ts`). Persisted client-side (`tutorPersonality.store`) and attached to every `/chat/tutor` call; layered *under* the teacher's `agent_settings`.

### question node (5 types)
The critical-thinking blocks: **QuestionChoice** (multiple choice), **QuestionWrite** (open-ended, AI-graded), **QuestionBlankChoice** (fill-in-blank, pick options), **QuestionBlankWrite** (fill-in-blank, typed), **QuestionAgent** (embedded free Ask-AI block). Each = a `*Node.ts` (schema/attrs) + `*View.tsx` (React render).

### StudentMemory
Server model: one warm sketch per logged-in student — interests / strengths / growth_areas / preferences / recent_topics (≤10 each, ≤120 chars) + selected `tutor_personality`. Updated **async every N student turns** (`AI_MEMORY_EVERY_N_TURNS`, default 3) by the fast model; injected into the tutor prompt as a digest. Viewable + erasable at `/profile`. Anonymous users have none.

### structured 401
The exact error body `protect` returns on auth failure: `{ message, code, forceRelogin: true, clearToken: true }` where `code ∈ TOKEN_MISSING | TOKEN_EXPIRED | TOKEN_INVALID | USER_NOT_FOUND | ACCOUNT_INACTIVE`. The client's axios response interceptor keys off this exact shape to auto-logout. A two-sided contract — change both halves together. See [`data-flow.md`](data-flow.md) §auth.

### suggestions / chips
Up to 3 follow-up questions in the student's voice that end every tutor reply. The model emits a `[SUGGESTIONS]` block; `parse.ts` splits it off; the client renders tappable chips. A format contract spanning `persona.ts` ↔ `parse.ts` ↔ the client.

### saveState (Fabric bridge)
The function wiring a Fabric canvas back into its TipTap node attr: on mount, restore from the attr; on change, serialize back. Because Fabric 7 needs manual event control, `useFabricSetup` wires this explicitly. Keeps **① TipTap = source of truth** true for drawings.

### static mode / dynamicUpdate
Toolbar performance trick: toolbar items are `memo()`'d (fast, but stale to editor state). `dynamicUpdate` flips them to "static mode" so they re-render on editor changes when needed. In `TipTapEditor`; explained in [`../client/src/components/README.md`](../client/src/components/README.md).

### zoom (viewer) vs font size (app)
**Two separate mechanisms — never merge them.** The lesson viewer scales via CSS `zoom` on the card container (chosen over `transform: scale` to avoid a double-scrollbar bug). The app-wide font-size control (`appearance.store`) sets an inline `%` on `<html>`. See [`ideas.md`](ideas.md) ADR-007/008.
