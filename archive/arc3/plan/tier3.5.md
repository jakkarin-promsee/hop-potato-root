# Tier 3.5 — Teacher AI Helper: Execution Plan

Written 2026-07-11 for an implementing agent (Cursor or any capable model) to execute **without access to the planning conversation**. Everything you need is in this file plus the repo docs. Read [`../CLAUDE.md`](../CLAUDE.md), [`../client/CLAUDE.md`](../client/CLAUDE.md), and [`../server/CLAUDE.md`](../server/CLAUDE.md) before touching code. The roadmap summary of this tier is `ROADMAP.md` §8.

> Recheck pass 2026-07-11 (same day): every file/line/signature claim below was re-verified against the code. Corrections folded in: `QuestionChoice` has an `answerType: "single" | "multi"` attr; guide-answer AI fill is `QuestionWrite`-only (choice views derive `guideAnswer` from the correct choices automatically); the sidebar category key is `special`; tiptap-markdown's `insertContentAt` markdown parsing is **confirmed**, as is BubbleMenu availability; `getLessonContext` carries ownership + existing questions, which several actions now exploit.

**What this tier builds:** an AI copilot inside the lesson editor (`/canvas/:id`) so low-tech teachers can author lessons *with* AI — generate questions + guide answers, write formulas without knowing LaTeX, proofread/format Thai prose, draft/import lessons, get an AI content review, and auto-fill publish metadata + tutor settings. Owner context: the target teachers got their first ChatGPT training a week ago; Canva is already hard for them; LaTeX is a non-starter.

---

## 0. Ground rules (non-negotiable)

### The two Golden Rules (they govern the STUDENT side — do not regress it)

1. **Never ration tokens for real users.** No quotas for students *or* teachers. Rate limiting exists only to stop bots (reuse the existing generous limiter).
2. **Anonymous users keep full access on the read/AI side.** This tier does not touch any student surface. **Never** add `protect` to `/api/chat/tutor` or content read paths while working here — and conversely, every route this tier adds is `protect`ed (creator surface is teacher-only by nature; the editor route `/canvas/:id` is already `ProtectedRoute`).

### Owner decisions shaping this tier (do not re-litigate)

| # | Decision |
| --- | --- |
| T1 | **Formula AI uses the existing KaTeX path.** `FormulaBlock` already has a `latex` string attr and `FormulaCanvas` is already LaTeX-first (textarea + KaTeX render). AI fills `latex` through the same write path as manual edits. Do NOT build a LaTeX→`FormulaNode`-tree parser; the legacy `formula` tree stays as fallback for old lessons. |
| T2 | **Preview → accept, never auto-apply.** Every AI result is shown to the teacher first; it enters the document only via a normal editor transaction when the teacher explicitly accepts. Reject = document byte-identical. |
| T3 | **AI never emits raw TipTap document JSON.** Prose = markdown. Blocks = typed JSON matching the node attrs exactly, validated server-side. The client converts to nodes itself. |
| T4 | **No new env vars.** Reuse `AI_FAST_MODEL`, `AI_TUTOR_MODEL`, `AI_OUTPUT_LANGUAGE`, `AI_RATELIMIT_*`. Free-tier constraints apply (principle 6): no paid SaaS, no queue infra. |
| T5 | **Thai-first.** AI output defaults to Thai (reuse the `AI_OUTPUT_LANGUAGE` convention from the tutor). All new UI copy is bilingual via `useAppI18n` `t("English", "ไทย")`. |
| T6 | **One phase = one commit**, tests green in every half touched before the commit. Same working agreement as every previous tier. |

### Scope fence

- Touches: `server/src` (new creator routes/controller/services + tests), `client/src` (editor AI panels, `lib/creatorApi.ts` + tests), docs.
- **Do NOT touch:** `services/tutor/persona.ts` or any tutor prompt (owner tone-gates those), the tutor endpoint behavior, the autosave/`clientUpdatedAt` 409 machinery, student-facing pages, `optionalAuth` on any existing route.
- Creator prompts are NEW prompts — they live in `services/creator/prompts.ts` and are logged in `server/prompt-notes.md` (a short "creator prompts added" entry per phase that changes them; they are not tone-gated like the tutor persona but keep the same warm, no-shame spirit).

### Things you will see and must NOT fix (owned elsewhere)

- `client/vite.config.js` lint error `__dirname` no-undef → Tier 4.
- `og:image` relative URL TODO in `client/index.html` → Tier 5.
- `OWNER_FACEBOOK_URL` placeholder in `client/src/lib/contact.ts` → Tier 5.
- If `formulaReducer.ts` shows a pre-existing TS complaint, fix it **only if** it actually blocks your 3.5.C work; otherwise leave it and note it in the phase notes.

### Git reality ⚠️

`client/` and `server/` are **separate git repos**; `Hot-Potato/` is not a repo (and the user's home dir has a stray `git init` — foreign files in `git status` mean you're in the wrong directory). Run all git commands inside `client/` or `server/`. **Do not push** — deploys happen in one shot at Tier 5 (owner decision D1).

### Commands & definition of done

```bash
cd server && npm test        # vitest run — offline (in-memory Mongo, mocked Gemini)
cd server && npm run build   # tsc
cd client && npm test        # vitest run — happy-dom
cd client && npm run build   # vite build
cd client && npm run lint    # 1 known pre-existing error (__dirname) is OK; no NEW errors
```

A phase is done when: its acceptance criteria pass, both suites are green, builds pass, docs are updated (§6), and the phase is committed with the message given in its section.

### Test conventions (match exactly)

- **Server:** vitest + supertest + `mongodb-memory-server`. New API tests go in `server/test/api/creator.test.ts`; unit tests in `server/test/unit/`. Mock Gemini the way `test/api/tutor.test.ts` does (`vi.mock` on the genai client / capture helpers — read that file first and copy its setup, including `seedUser`/`seedLesson` from `test/helpers/seed.ts` and the reset patterns in `beforeEach`).
- **Client:** vitest + happy-dom, tests next to code (`__tests__` folders). Copy patterns from `src/pages/__tests__/` and `src/components/__tests__/`. Mock `@/lib/creatorApi` in component tests; test the API helper separately with a mocked axios instance (see `src/lib/__tests__/`).

---

## 1. Verified current state (2026-07-11 — trust this over guesses, re-verify against files if something looks off)

### 1.1 Editor surface

- Route `/canvas/:id` → `TipTapCanvas` page → `TipTapEditor.tsx` (top bar `EditorHeader.tsx`, left sidebar `EditorLeftSidebar.tsx` with the block-insert buttons, right sidebar, main area). Read `client/src/components/README.md` before editor work. Lazy-loaded (Tier 2.A) — new editor-only code should stay inside the editor chunk (import from editor files only, nothing added to `main.tsx`).
- Extensions configured in `editor/config/editorExtensions.ts` via `createEditorExtensions(editable)`. `tiptap-markdown` (`Markdown` extension) is already installed and configured — this is your markdown→nodes bridge.
- **Markdown insertion (CONFIRMED 2026-07-11):** `tiptap-markdown` overrides `insertContentAt` so `editor.commands.insertContentAt(range, markdownString)` parses the string as markdown (verified in `node_modules/tiptap-markdown/dist/tiptap-markdown.umd.js` ~line 975 — the extension wraps `insertContentAt: (range, content, options) => ...`). This is the one true path for all prose insertion in 3.5.D/E. Still add one small round-trip test in the first phase that uses it (guards against a future tiptap-markdown upgrade changing behavior).

### 1.2 Question block nodes (`client/src/components/editor/extensions/`)

TipTap node names (for `insertContent` `type`) and attrs — **these are the contract for `generate_questions`; match them exactly:**

| Node name | File | Attrs (beyond `id`, `feedbackMode`) |
| --- | --- | --- |
| `QuestionChoice` | `QuestionChoiceNode.ts` | `question: string`, `choices: { text: string; correct: boolean }[]`, `answerType: "single" \| "multi"` (generation always sets `"single"` in v1). The choice **view derives** `guideAnswer` from the correct choices automatically (`QuestionChoiceView.tsx` ~486) — there is no explanation attr to fill. |
| `QuestionWrite` | `QuestionWriteNode.ts` | `question: string`, `answer: string` (**this is the teacher guide answer** — `QuestionWriteView.tsx` ~231 sends it as `questionContext.guideAnswer`) |
| `QuestionBlankChoice` | `QuestionBlankChoiceNode.ts` | `template: string`, `choices: string[]`, `correctByBlank: number[]` (index into `choices` per blank) |
| `QuestionBlankWrite` | `QuestionBlankWriteNode.ts` | `template: string`, `blankAnswers: string[]` |
| `QuestionAgent` | `QuestionAgentNode.ts` | `title`, `chatHistory`, `collapsed` — **out of scope for generation** (it's the free-chat block, nothing to generate) |

These `kind` names line up 1:1 with `LessonQuestion.kind` on the server (`"choice" | "write" | "blank_choice" | "blank_write"` in `lessonContext.service.ts`) — keep the creator action types identical.

- **Blank marker format** (both blank views, `BLANK_TOKEN_REGEX`): `/\[Q-(\d+)\]|\{\{(\d+)\}\}/g` — i.e. `[Q-1]`, `[Q-2]`, … (or `{{1}}`). Generated templates must use `[Q-n]`, numbered from 1, contiguous.
- Every node has an insert command (`insertQuestionChoice()` etc. — see `QuestionChoiceNode.ts` ~106) that sets `id: crypto.randomUUID()`, sets `feedbackMode: DEFAULT_QUESTION_FEEDBACK_MODE` (imported from `./questionMode`), and appends a trailing `{ type: "paragraph" }`. **Mirror all three** when inserting generated blocks via `insertContent` with explicit attrs (and `answerType: "single"` for choice).

### 1.3 FormulaBlock (`client/src/components/editor/FormulaBlock/`)

- Node `formulaBlock` (`index.tsx`), attrs: `{ id, formula: FormulaNode (legacy visual-builder tree), latex: string }`.
- `FormulaCanvas.tsx` is **already LaTeX-first**: `inferInitialLatex()` prefers a non-empty `latex` attr (line ~90), falls back to converting the legacy tree via `formulaToLatex.ts`; `latexInput` state + textarea; `persistLatex(nextLatex)` (~line 135) writes the attr; KaTeX `renderToString` in a `useMemo` with try/catch (~line 221) — render errors already have a fallback path.
- **3.5.C only adds an AI panel that calls `persistLatex`** (or sets the same state the textarea uses). No node-schema change needed.

### 1.4 Client API bridge patterns

- `src/lib/axios.ts` exports the single `api` instance (attaches JWT automatically — creator calls are teacher-authed for free). Creator v1 is **JSON-only, no SSE**, so plain axios is fine (the tutor's fetch-based SSE exists because of streaming; do not copy it).
- `editor/extensions/tutorApi.ts` shows the error-shape convention: a typed `AiUnavailableError`, callers render a retry UI, never fake replies. Mirror this with a `CreatorAiError` in `src/lib/creatorApi.ts`.

### 1.5 Server AI stack (all reusable as-is)

- `services/tutor/genaiClient.ts` → `callTutorModel({ model, systemInstruction, history, message }): Promise<string>` — **fully generic** (despite the name): builds contents, wraps `generateWithRetry`, records `recordAiSuccess/Failure`. Reuse it with `history: []`; do not duplicate it. **3.5.A extends its opts with an optional `responseMimeType?: "application/json"`** passed through to `config` (backwards-compatible — tutor callers unaffected; add a unit test that omitting it leaves the config identical). Gemini JSON mode + the tolerant parser together is the reliability story for the 7 JSON-output actions (`formula_latex`, `generate_questions`, `distractors`, `import_structure`, `critic`, `lesson_meta`, `agent_settings_suggest`).
- `services/tutor/models.ts` → `getFastModel()` / `getTutorModel()` (single source of model defaults).
- `services/lessonContext.service.ts` → `getLessonContext(contentId: string): Promise<LessonContext | null>` (cached by `updatedAt`, 100-entry cache, **no text-length cap** — same exposure the tutor already has, acceptable). `LessonContext` is rich and this tier should exploit it: `{ contentId, title, text, questions: LessonQuestion[], agentSettings, ownerId, collaboratorIds, accessType, updatedAt }` where `LessonQuestion = { blockId, kind, question, choices?, guideAnswer?, feedbackMode }`. Consequences baked into §2: the controller does **load + ownership check from one `getLessonContext` call** (null → 404; `ownerId`/`collaboratorIds` vs `req.user._id` → 403), `generate_questions`/`critic` see the existing questions structurally, and `agent_settings_suggest` sees the current settings.
- `middlewares/rateLimit.middleware.ts` → `createChatRateLimiter()` (array of limiters, generous, keyed by user/IP). Reuse for the creator route.
- `middlewares/auth.middleware.ts` → `protect`. `types/index.ts` → `AuthRequest`.
- Route wiring pattern: see `routes/chat.routes.ts`; app mounting in `app.ts` line ~68 area (`errorHandler` must stay last, line ~71).
- Ownership check reference: `content.controller.ts` ~121–123 (`isOwner` / `isCollaborator`) — but per above, implement it against `LessonContext.ownerId`/`collaboratorIds` so content loads only once.
- `createChatRateLimiter()` is a **factory** — calling it for the creator router creates fresh limiter instances with their own buckets, so teacher AI usage never eats the student chat budget. This is intentional; don't share instances.
- Controller style: `(req: AuthRequest, res: Response) => Promise<void>`, manual inline validation with `typeof` guards, explicit `res.status(...).json(...)` returns. Match it.

### 1.6 Publish modal + agent settings

- `components/editor/PublishSettingsModal.tsx` edits title/description/topics/access + `agentSettings` held in `canvas.store.ts` (`{ persona_note, allow_direct_answers, scope, custom_guidelines }`). Server sanitizes on save (persona_note ≤ 500, custom_guidelines ≤ 1000, scope enum `lesson_only | lesson_plus_general`). 3.5.F autofills these **fields in the modal state only** — saving stays the teacher's explicit action.

---

## 2. The contract — `POST /api/creator/assist` (Phase 3.5.A builds this; later phases consume it)

### Guards & common request

```
router.post("/assist", protect, ...createChatRateLimiter(), creatorAssist);
```

mounted at `/api/creator` in `app.ts`. Common body:

```ts
{
  contentId: string;   // required, valid ObjectId; getLessonContext(contentId) → null = 404;
                       // 403 unless req.user._id matches ctx.ownerId or is in ctx.collaboratorIds
  action: CreatorAction;  // required, one of the 11 below; 400 otherwise
  payload?: object;    // action-specific, validated per the tables below
}
```

Response: `200 { result: <action-specific object> }`. Errors: `400` (validation, with a short Thai-friendly `message`), `401` (no/invalid token — from `protect`), `403` (not owner/collaborator), `404` (content), `500` (`AI_UNAVAILABLE` after retry — same spirit as the tutor's failure path).

### Model routing

| `getFastModel()` | `getTutorModel()` |
| --- | --- |
| `formula_latex`, `proofread`, `distractors`, `lesson_meta` | `generate_questions`, `guide_answer`, `outline`, `draft_section`, `import_structure`, `critic`, `agent_settings_suggest` |

### JSON robustness (applies to every action whose result is JSON)

JSON actions call the model with `responseMimeType: "application/json"` (the `callTutorModel` extension from §1.5) **and** prompts still end with the exact schema + "ตอบเป็น JSON ตาม schema เท่านั้น". Belt-and-suspenders on top: parse with a tolerant extractor (strip ```json fences, trim to first `{`/last `}`). On parse failure: **one** re-ask appending "คำตอบที่แล้วไม่ใช่ JSON ที่ valid — ส่งเฉพาะ JSON ตาม schema เท่านั้น". Second failure → `500 AI_UNAVAILABLE`. Put the extractor + retry in `services/creator/parse.ts` with unit tests. (Plain-output actions — `proofread`, `outline`, `draft_section`, `guide_answer` — do NOT set the JSON mime type: the model returns raw markdown/plain text and the **server** wraps it in the result object.)

### The 11 actions

Caps are validation rules — reject with 400 above them. All generated text defaults to Thai.

**1. `formula_latex`** — payload `{ formulaText: string (1..300, required), description?: string (..300), usage?: string (..300) }` → result `{ latex: string, note?: string }`. Prompt: convert the human-typed formula to KaTeX-compatible LaTeX (no `$` delimiters, no `\begin{equation}`); `description`/`usage` disambiguate symbols (e.g. "ความเร็ว" → v vs u assignment); `note` = one short Thai sentence only when the AI had to guess something.

**2. `proofread`** — payload `{ markdown: string (1..12000, required), preset: "proofread" | "format" | "simplify" | "shorten" | "expand" | "reading_level", gradeLevel?: string (..40, required when preset=reading_level) }` → result `{ markdown: string }`. Presets: แก้คำผิด/วรรคตอนเท่านั้น (proofread — must NOT change wording) · จัดย่อหน้า+หัวข้อ+bullet (format — must NOT change wording, structure only) · เกลาให้อ่านง่าย (simplify) · ย่อ (shorten) · ขยายความ+ตัวอย่าง (expand) · ปรับระดับภาษาให้เหมาะกับ `gradeLevel` (reading_level). Result replaces the teacher's selection wholesale, so the prompt must forbid adding meta-commentary.

**3. `generate_questions`** — payload `{ scope: "lesson" | "selection", selectionMarkdown?: string (..12000, required when scope=selection), types: ("choice"|"write"|"blank_choice"|"blank_write")[] (1..4, non-empty), count: number (1..10), difficulty?: "easy"|"medium"|"hard"|"mixed" }`. Server injects lesson text (scope=lesson) **and always injects the existing questions** (`LessonContext.questions` — question text + kind) with the instruction "อย่าสร้างคำถามซ้ำหรือใกล้เคียงกับคำถามที่มีอยู่แล้ว". → result `{ questions: GeneratedQuestion[] }` where:

```ts
type GeneratedQuestion =
  | { type: "choice"; question: string; choices: { text: string; correct: boolean }[] }   // 3..5 choices, exactly 1 correct (v1)
  | { type: "write"; question: string; guideAnswer: string }                              // client maps guideAnswer → attr `answer`
  | { type: "blank_choice"; template: string; choices: string[]; correctByBlank: number[] } // template uses [Q-1]..[Q-n] contiguous from 1;
                                                                                            // choices 2..8; correctByBlank.length === blank count; valid indices
  | { type: "blank_write"; template: string; blankAnswers: string[] };                    // blankAnswers.length === blank count
```

Server-side validator (`services/creator/validate.ts`): enforce every constraint above (question/template 1..2000 chars, choice text 1..500, blank-token regex `\[Q-(\d+)\]`), **drop** malformed items rather than failing the request, return only valid ones; if all items were dropped, treat as parse failure (one retry, then 500). Prompt style: critical-thinking questions in the spirit of the product (คิดวิเคราะห์ ไม่ใช่ท่องจำ), distractors must be plausible misconceptions, never trick questions.

**4. `guide_answer`** — payload `{ question: string (1..2000, required) }` → result `{ guideAnswer: string }` (model returns plain text; server wraps + clamps to 2000). A model guide answer + what a good student answer contains. The guide answer is tutor context for น้องมันฝรั่ง — say so in the prompt: "ครูและ AI ติวเตอร์จะใช้คำตอบนี้เป็นแนวเฉลย". **Client scope (verified): `QuestionWrite` blocks only** — choice views derive their `guideAnswer` from the correct choices automatically and have no attr to store an explanation, so there is nothing to fill there.

**5. `distractors`** — payload `{ question: string (1..2000), correctText: string (1..500), existing?: string[] (..8) }` → result `{ distractors: string[] }` (exactly 3, plausible-misconception style, none equal to `correctText`/`existing`).

**6. `outline`** — payload `{ topic: string (1..200, required), gradeLevel?: string (..40), objectives?: string (..1000) }` → result `{ outlineMarkdown: string }` — `##` section headings, each followed by one italic line summarizing what the section will cover; 4..8 sections; ends with a "คำถามชวนคิด" placement suggestion. No body text.

**7. `draft_section`** — payload `{ heading: string (1..200, required), outlineMarkdown?: string (..4000), styleHint?: string (..500) }` + server appends current lesson context → result `{ markdown: string }` — the body for that one section only (no repeated heading, no other sections), warm Thai explainer voice, concrete examples ใกล้ตัวเด็ก.

**8. `import_structure`** — payload `{ rawText: string (1..20000, required) }` → result `{ markdown: string, suggestedQuestions: GeneratedQuestion[] }` (same schema + validator as action 3, ≤5 items). Prompt: restructure the teacher's pasted material into headings + readable sections **preserving their content** (reorganize, don't rewrite facts), fix obvious typos, and propose questions covering the material.

**9. `critic`** — payload `{}` (server loads the whole lesson) → result:

```ts
{
  summary: string;                       // 2-3 Thai sentences, warm, strengths first
  issues: { area: "accuracy"|"completeness"|"readability"|"age_fit"|"questions";
            severity: "info"|"warn"; where: string;   // section heading text or question text (first 80 chars)
            note: string; suggestion?: string }[];    // ≤ 12 items
  checklist: { item: string; pass: boolean }[];       // fixed items, see prompt
}
```

Checklist items (fixed in the prompt so output is stable): มีคำนำ/เกริ่น · เนื้อหาแบ่งหัวข้อชัดเจน · มีตัวอย่างประกอบ · มีคำถามอย่างน้อย 3 ข้อ · คำถามครอบคลุมเนื้อหาหลัก · มีสรุปท้ายบท. The server **overrides the deterministic item itself**: มีคำถามอย่างน้อย 3 ข้อ = `ctx.questions.length >= 3` (never let the AI get a countable fact wrong); the subjective items stay AI-judged. **Informational only — the client must never block publish on this.** No scores, no grades (product principle 2 applies to teachers too).

**10. `lesson_meta`** — payload `{}` → result `{ title: string (≤120), description: string (≤500), topics: string[] (≤5, each ≤40) }` from lesson content.

**11. `agent_settings_suggest`** — payload `{}` → result `{ persona_note: string (≤500), custom_guidelines: string (≤1000), scope: "lesson_only"|"lesson_plus_general", allow_direct_answers: boolean, reason: string (≤300) }`. Server includes the **current** `LessonContext.agentSettings` in the prompt so non-default teacher intent is respected ("ครูตั้งค่าไว้แล้วบางส่วน — ต่อยอด อย่าลบทิ้ง"). Prompt: read the lesson and draft tutor settings a caring teacher would write — persona_note = subject-appropriate coaching tone hints; custom_guidelines = lesson-specific dos/don'ts (e.g. "อย่าเฉลยข้อ 3 ตรงๆ ให้ชวนคิดเรื่องแรงเสียดทานก่อน"); recommend scope + direct-answers with a one-line reason. Server clamps lengths to the same limits `Content.agent_settings` sanitization uses.

---

## 3. Phases

Order: **A → B → C → D → E → F.** A blocks everything; B/C ship the highest student-facing value first; D/E/F are independent of each other after A.

---

### Phase 3.5.A — Server foundation

**Goal:** the full `/api/creator/assist` contract above, tested offline.

**Files:**

- `server/src/routes/creator.routes.ts` — router as specced; mount in `app.ts` as `app.use("/api/creator", creatorRoutes)` (before `errorHandler`).
- `server/src/controllers/creator.controller.ts` — `creatorAssist`: validate common fields → load content → ownership check → dispatch by action to a per-action handler (validate payload → build prompt → `callTutorModel` with routed model → parse/validate → respond). Keep per-action handlers small and flat; match the tutor controller's inline-validation style.
- `server/src/services/creator/prompts.ts` — one `build<Action>Prompt(...)` per action returning `{ systemInstruction, message }`. Shared preamble: Thai output (respect `AI_OUTPUT_LANGUAGE` the same way tutor persona does), audience = Thai school lessons, warm no-shame spirit, and the JSON-only footer for JSON actions.
- `server/src/services/creator/parse.ts` — tolerant JSON extractor + one-retry helper (`askJson(callModel, prompt): Promise<unknown>` style).
- `server/src/services/creator/validate.ts` — `validateGeneratedQuestions(raw): GeneratedQuestion[]` + small validators for the other JSON results (clamp lengths, enforce enums; return null → triggers the retry path).

**Tests** (`test/api/creator.test.ts` + `test/unit/creatorValidate.test.ts` / `creatorParse.test.ts`):

1. 401 without token; 403 as a logged-in non-owner; 404 bad contentId; 400 unknown action; collaborator passes.
2. Happy path per action with mocked Gemini returning canned output — assert result shape and that the right model (fast vs tutor) was requested (capture like `tutor.test.ts` does).
3. Parse robustness: fenced ```json output still parses; garbage → exactly one retry → 500 on second garbage.
4. Validator: drops choice-question with 0 or 2 correct; drops blank_choice with non-contiguous `[Q-n]` or wrong `correctByBlank` length; keeps valid siblings.
5. `generate_questions` scope=lesson injects lesson context (assert the captured prompt contains seeded lesson text via `lessonContext`).
6. Rate limiter present on the route (copy the approach in `test/api/rateLimit.test.ts` — it already tests the chat limiter with low env thresholds).
7. `callTutorModel` regression: with `responseMimeType` omitted, the generateContent `config` is byte-identical to before the change (capture the call like `tutor.test.ts` does); with it set, `config.responseMimeType === "application/json"`.

**Docs:** add the endpoint to `server/CLAUDE.md` API surface + a `services/creator/` line in the architecture tree; `prompt-notes.md` entry "2026-07-XX Tier 3.5.A creator prompts added (11 actions)".

**Commit (server):** `feat(tier3.5.A): unified creator-assist endpoint — 11 AI authoring actions`

---

### Phase 3.5.B — Question AI (client)

**Goal:** teachers generate/complete questions from the editor with preview-first UX.

**Files:**

- `client/src/lib/creatorApi.ts` — typed `callCreator<A extends CreatorAction>(contentId, action, payload)` on the shared `api` axios instance; `CreatorAiError` carries a machine code only (mirror `AiUnavailableError` semantics — `useAppI18n`'s `t()` is a hook and cannot be used in a lib file; the **components** render the shared error copy `t("AI is busy, try again", "AI ไม่ว่างแป๊บนึง ลองอีกทีนะ 🥔")`). Types for all 11 actions live here (single contract file, like `tutorApi.ts` is for students).
- `client/src/components/editor/ai/AiQuestionDialog.tsx` — dialog (shadcn `Dialog`) opened from a "✨ สร้างคำถามด้วย AI" button in `EditorLeftSidebar` — the category with `key: "special"`, label "Question" (see the category array at ~line 90). Controls: scope (ทั้งบทเรียน / วางเนื้อหาเฉพาะส่วน — a textarea for v1 selection scope), types multi-select (4 gradeable types), count (default 3, max 10), difficulty. Results render as **preview cards** (question text + choices/template/answers, per type) each with เพิ่มลงบทเรียน / ทิ้ง buttons + "เพิ่มทั้งหมด".
- Insert flow: map `GeneratedQuestion` → node attrs (`guideAnswer` → `answer`; `id: crypto.randomUUID()`; `feedbackMode: DEFAULT_QUESTION_FEEDBACK_MODE` from `./questionMode`; choice adds `answerType: "single"`) → `editor.chain().focus().insertContentAt(<pos>, [{ type: "QuestionChoice", attrs }, { type: "paragraph" }])` where `<pos>` = the block boundary after the current selection if the editor has focus, else the end of the doc — mirroring the existing `insertQuestion*` commands' trailing-paragraph behavior.
- Guide-answer fill: in `QuestionWriteView` **only** (verified — choice views derive `guideAnswer` from correct choices and have no explanation attr): a small "✨ ให้ AI ร่างแนวเฉลย" button when `answer` is empty → `guide_answer` → show result in a confirm popover → on accept `updateAttributes({ answer })`.
- Distractors: in `QuestionChoiceView` editor mode, "✨ เพิ่มตัวลวง" → `distractors` with current question + correct choice → preview → append accepted ones as `correct: false` choices.

**Tests:** `creatorApi` unit (payload/result mapping, error → `CreatorAiError`); dialog test (mocked api: renders preview cards, accept calls insert with exact attrs — assert `[Q-n]` template passthrough and `guideAnswer→answer` mapping, reject inserts nothing); guide-answer button test (empty vs filled `answer`).

**Manual pass:** real lesson on `:5173` + `:5000`, generate 3 mixed questions, insert, save, reload → blocks behave identically to hand-made ones (student view answers them, tutor feedback works — spot-check one in `/view/:id`).

**Commit (client):** `feat(tier3.5.B): question AI — generate, guide answers, distractors (preview-first)`

---

### Phase 3.5.C — Formula AI (client)

**Goal:** a teacher who has never seen LaTeX gets a rendered formula in one round-trip.

**Files:** `client/src/components/editor/FormulaBlock/FormulaCanvas.tsx` (+ a small `AiFormulaPanel` component in the same folder if cleaner).

- Collapsible "✨ ให้ AI เขียนสูตร" panel in **edit mode only**, above/beside the LaTeX textarea. Three inputs mirroring the owner's spec: สูตร (พิมพ์แบบที่ครูพิมพ์เอง เช่น `s = ut + 1/2at^2`) — required; คำอธิบายสั้นๆ (เช่น "สมการการเคลื่อนที่") — **required in the UI** (the owner's brief lists it as the teacher's job; it's what disambiguates symbols — the API keeps it optional for robustness); ใช้ทำอะไร (เช่น "ใช้คำนวณระยะทาง") — optional. One "สร้างสูตร" button.
- On result: write `result.latex` through the exact same path as manual typing (`setLatexInput` + `persistLatex`) so undo/autosave behave identically; show `note` (if any) as muted text under the panel. The teacher can then hand-edit the LaTeX — that IS the accept step here (the rendered preview + editable textarea is the preview; no extra confirm dialog needed, but the panel must not fire on mount or without click).
- KaTeX render failure of AI output: existing try/catch fallback already shows the error state — add a "ลองใหม่" hint in the panel when the current latex fails to render.

**Tests:** panel renders in edit mode only; submit calls `formula_latex` with the 3 fields; result lands in the textarea/attr (mock api); render-error path shows the retry hint.

**Commit (client):** `feat(tier3.5.C): formula AI — plain-text → LaTeX into the existing KaTeX block`

---

### Phase 3.5.D — Writing assistant (client)

**Goal:** selection-based proofread/format/level actions with a before/after preview.

- **BubbleMenu availability CONFIRMED (2026-07-11):** `@tiptap/react` v3.20 exports `./menus` (import `BubbleMenu` from `@tiptap/react/menus`) and `@floating-ui/dom` is already in `node_modules` — no new dependency. If BubbleMenu turns out to fight the existing toolbar UX in practice, the sanctioned fallback is a "✨ ปรับข้อความ" dropdown in the main toolbar acting on the current selection — same actions, same preview flow. Record the choice in phase notes.
- Actions (map to `proofread` presets): แก้คำผิด · จัดหน้า/หัวข้อ · เกลาให้อ่านง่าย · ย่อ · ขยายความ · ปรับระดับชั้น (asks for ป./ม. level in a mini-select).
- Selection serialization v1: plain text via `state.doc.textBetween(from, to, "\n\n")` — **known limitation: replacing formatted text loses inline marks.** Mitigate: preview dialog states "การจัดรูปแบบตัวหนา/สีในช่วงที่เลือกจะถูกจัดใหม่" and the returned markdown re-creates structure (headings/bold from the AI). (Note: §1.1 confirmed markdown→nodes *insertion*; selection→markdown *serialization* is a separate problem — tiptap-markdown's serializer is doc-level. If you find a clean way to serialize just the selected slice through its internals, you may use it and drop the caveat; otherwise plain text v1 stands.)
- Preview dialog: ก่อน/หลัง side-by-side (stacked at ≤390 px), ใช้เลย → `insertContentAt({ from, to }, <markdown per §1.1 path>)`; ยกเลิก → nothing. One AI call per click, no auto-rerun.

**Tests:** each preset sends the right payload; apply replaces exactly the selection range; cancel leaves doc unchanged (serialize doc before/after and compare).

**Commit (client):** `feat(tier3.5.D): writing assistant — proofread/format/reading-level with preview`

---

### Phase 3.5.E — Draft & import (client)

**Goal:** blank page → structured draft; existing material → structured lesson.

- Entry points: an empty-doc CTA card ("เริ่มบทเรียนด้วย AI ✨") shown when the doc is effectively empty (single empty paragraph), plus a toolbar/sidebar entry that's always available. Both open `AiDraftDialog` with three tabs:
  1. **ร่างโครง** — topic/gradeLevel/objectives → `outline` → preview → insert headings markdown at doc start.
  2. **เติมเนื้อหา** — lists current `##`/`###` headings from the doc (walk `editor.state.doc` for heading nodes); pick one → `draft_section` (send the heading + an outline snapshot) → preview → insert below that heading.
  3. **วางเนื้อหาเดิม** — big textarea (cap 20,000 chars with a live counter) → `import_structure` → preview markdown **and** suggested-question cards (reuse 3.5.B's preview-card component + insert mapping) → insert selected parts.
- All inserts go through the §1.1 markdown path; all previews follow T2 (nothing lands without accept).
- Loading states matter (tutor-model calls can take ~5–15 s on Render cold start): reuse the app's existing loading patterns + a playful Thai line ("น้องมันฝรั่งกำลังร่างให้... 🥔✍️").

**Tests:** empty-doc detection; outline insert produces heading nodes; fill-section inserts under the right heading; import maps questions through the shared validator/mapping; cancel paths clean.

**Commit (client):** `feat(tier3.5.E): AI draft — outline, section fill, paste-import`

---

### Phase 3.5.F — Critic + publish autofill (client)

**Goal:** pre-publish confidence for the teacher, and the tutor configured by AI.

- **Critic:** "✨ ตรวจบทเรียน" button in `EditorHeader` (near publish) → panel/dialog calling `critic` → render `summary` (top, warm), `checklist` (✓/✗ list), `issues` grouped by `area` with severity badges (`info` muted, `warn` amber — no red walls, no scores). `where` renders as plain text locator ("ในส่วน: การเคลื่อนที่แนวตรง"); v1 has **no scroll-to-block** for prose (question issues MAY deep-link later — out of scope). A "ตรวจอีกครั้ง" button re-calls. **Never gate the publish button on this.**
- **Publish autofill:** in `PublishSettingsModal`, two buttons: "✨ ช่วยกรอกข้อมูล" → `lesson_meta` → fill the title/description/topics **fields**: empty fields fill silently, fields that already have content ask one confirm ("เขียนทับของเดิมไหม?") before overwriting; "✨ แนะนำการตั้งค่าติวเตอร์" → `agent_settings_suggest` → same fill rule for the `agentSettings` fields in `canvas.store` + show `reason` as helper text. Everything stays editable; nothing saves until the teacher hits the modal's save.

**Tests:** critic renders all three sections from a canned report (incl. empty-issues state "ดูดีมาก! 🎉"); publish button unaffected by critic state; autofill fills store/fields without triggering save; overwrite-confirm on non-empty fields.

**Manual pass:** run critic on the test lesson (`/view/69e39d0b60d467bd515a4945`'s source), sanity-check the advice reads warm and useful; autofill agent settings, save, then verify the student tutor actually picks up the new `agent_settings` in one conversation.

**Commit (client):** `feat(tier3.5.F): AI critic + publish autofill (meta + agent settings)`

---

## 4. Prompt-writing guidance (applies to every action in `prompts.ts`)

- Shared system preamble (Thai): "คุณคือผู้ช่วยครูของแพลตฟอร์ม Hot Potato ช่วยครูไทยสร้างบทเรียนสำหรับนักเรียน โทนอบอุ่น ให้กำลังใจ ใช้ภาษาไทยที่อ่านง่าย" + the language rule. `persona.ts` has a private `outputLanguageRule()` (~line 30) reading `AI_OUTPUT_LANGUAGE` — replicate that tiny function in `creator/prompts.ts` (or export it from persona.ts **without touching any persona text**). Copy the mechanism, NOT the persona.
- Question prompts must carry the product's soul: คิดวิเคราะห์ > ท่องจำ, no trick questions, distractors = ความเข้าใจผิดที่พบบ่อยจริง, never shame.
- JSON actions: schema embedded verbatim in the prompt + the JSON-only footer (§2). Keep schemas minimal — every extra field is a hallucination surface.
- Log a one-line entry per phase in `server/prompt-notes.md`. If the owner later wants tone changes to creator prompts, they go through the same notes file.

## 5. Failure playbook

| Symptom | Do this |
| --- | --- |
| Gemini returns fenced/dirty JSON | Already handled by `parse.ts` (strip + one retry). Don't loosen validators instead. |
| All generated questions dropped by validator | Treated as parse failure → retry once → 500. Check the prompt schema wording first, not the validator. |
| 429 from rate limiter during dev | Thresholds are env-tunable (`AI_RATELIMIT_PER_10MIN`) — raise locally, never hardcode higher defaults. |
| Teacher pastes >20k chars | 400 with Thai message + the client caps the textarea first (counter). Don't raise the cap — lessonContext and cost. |
| KaTeX can't render AI latex | Existing fallback shows error state; panel shows "ลองใหม่" hint. Never insert invalid latex silently. |
| Markdown insert produces wrong nodes | Re-check the §1.1 verified path; fall back to explicit parser API. Do not hand-build TipTap JSON on the server. |
| Render cold start makes calls slow | Loading states (already a repo convention). Do not add timeouts shorter than ~60 s on creator calls. |

## 6. Documentation updates (required — part of done)

- `server/CLAUDE.md`: API surface table (`POST /api/creator/assist` 🔒 + one-line action list), architecture tree (`services/creator/`), gotcha line ("creator endpoint is teacher-only; student AI stays optionalAuth").
- `client/CLAUDE.md`: editor section — AI panels list, `lib/creatorApi.ts` as the single creator bridge, the markdown-insert path you verified in §1.1, the preview→accept rule.
- `ROADMAP.md`: tick each phase's ✅ + shipped line (commit hash), same style as Tiers 0–3.
- `server/prompt-notes.md`: entries per §4.
- This file: fill §7 phase notes.

## 7. Phase notes (filled by the implementing agent, 2026-07-11)

### Baseline (before 3.5.A)
- server tests: 217/217 · client tests: 113/113 · date: 2026-07-11

### After each phase
- 3.5.A: commit `2b4ab13` (server) · tests 271/271 (+54) · deviations: error contract simplified — ALL AI failures (transport + invalid-after-retry) return `500 { code: "AI_UNAVAILABLE" }` rather than mixing 502/500 like the tutor path; one machine code keeps `CreatorAiError` handling trivial. `guide_answer` and `import_structure` inject lesson context beyond the letter of §2 (grounding); `{{n}}` blank templates are normalized to `[Q-n]` instead of dropped (tolerant-parsing spirit).
- 3.5.B: commit `b45848b` (client) · tests 133/133 (+20) · markdown/insert notes: question blocks insert as node-JSON arrays via `insertContentAt` (not markdown); insert position = block boundary after selection when editor focused, else end of doc (dialog holds focus → end-of-doc is the common case).
- 3.5.C: commit `0c5f7ac` · tests 138/138 (+5) · formulaReducer touched? **No** — no pre-existing TS complaint surfaced; `persistLatex` used as-is.
- 3.5.D: commit `266e595` · tests 148/148 (+10) · BubbleMenu vs toolbar decision: **toolbar dropdown** ("✨ ปรับข้อความ" in EditorHeader). Reasons: (1) the editor card scales via CSS `zoom`, and floating-ui/getBoundingClientRect anchor math under `zoom` is unreliable in Chromium — a BubbleMenu would drift at non-100% zoom; (2) a visible button is more discoverable for the low-tech target teachers than a hidden selection popup. Selection serialization is plain-text v1 as specced (preview carries the re-format caveat). Includes the §1.1 round-trip guard test (`writingAssist.test.ts`, real headless TipTap + tiptap-markdown).
- 3.5.E: commit `d09ddb1` · tests 160/160 (+12) · markdown path used: tiptap-markdown `insertContentAt` for all three tabs (outline → pos 0, section → after-heading pos, import → doc end). Observed quirk: markdown insertion leaves a trailing empty paragraph — harmless, documented in the tests.
- 3.5.F: commit `a0d1d1b` · tests 168/168 (+8) · owner eyeball of critic tone: **transcript captured, pending owner read** — live critic ran during the Playwright manual pass; tone was warm/strengths-first, caught a real typo the draft AI made ("ดูนะขรับ"), flagged Untitled title + unfinished sections, no scores. Critic flushes unsaved edits (`saveContent`) before reviewing so the report matches the screen.

### Deviations from this plan
1. **3.5.D BubbleMenu → toolbar dropdown** (sanctioned fallback; reasons above).
2. **Creator error status**: single `500 AI_UNAVAILABLE` for every AI-side failure instead of the tutor's 502/500 split — simpler client contract; plan §2 listed only 500 anyway.
3. **3.5.F "fields only, nothing saves until modal save"**: title/description/topics/agentSettings live in `canvas.store`, and `TipTapCanvas` runs an autosave interval for *all* dirty state — so autofilled fields can be autosaved before the modal's save button, exactly like fields the teacher types by hand. Fighting this would need a parallel modal-local state copy; not worth it. The teacher-visible guarantee kept: overwrite of non-empty fields always asks first, and everything stays editable.
4. **Tolerant question validation**: `{{n}}` templates normalized to `[Q-n]` (not dropped); distractor lists with >3 valid items are sliced to 3.
5. **Manual pass: DONE (2026-07-11, Playwright + live Gemini, local dev).** Ran as a fresh teacher account (`tier35.test@hotpotato.local` / `tier35-manual-pass` — owner may delete) on a new lesson `6a51b92c015ae40864a5bc71` ("แรงเสียดทาน", left **link-only** so it stays out of explore). Verified end-to-end with real Gemini calls: empty-doc CTA → outline (6 H2 nodes inserted at doc start) → section fill (landed under the right heading, warm Thai voice) → question AI (3 blocks inserted, render as normal student cards with ส่ง/Ask AI in `/view`) → formula AI (`f = uN` + คำอธิบาย → `f = \mu N`, KaTeX rendered — the description disambiguated u→μ) → writing assistant (selection simplify replaced exactly the selection) → critic (warm, strengths-first, caught a real typo the draft AI made, no scores) → publish autofill (meta: title/description/5 topics; agent settings: lesson-specific guidelines like "ห้ามเฉลยคำถามชวนคิดทั้ง 3 ข้อ" + reason). Security spot-checks: anonymous `/view` read 200, anonymous creator 401, non-owner creator 403, anonymous tutor chat 200 (Golden Rule 2 intact). One nit found for a future pass: the critic prompt lists existing questions without their choice texts, so it can wrongly flag choice questions as "missing choices" — consider including choices in `existingQuestionsSection` (server `services/creator/prompts.ts`).
