# ROADMAP.md — Hot Potato Phase 2: The AI Tutor Rework

The master plan for turning Hot Potato's AI from three stateless one-shot endpoints into a warm, Thai-first tutor agent that students want to talk to all day. Written for **implementing agents** (Cursor, Claude, etc.) and humans — each phase is self-contained enough to execute without this conversation.

> Read [`CLAUDE.md`](CLAUDE.md), [`server/CLAUDE.md`](server/CLAUDE.md), and [`client/CLAUDE.md`](client/CLAUDE.md) before touching code. They describe the stack, conventions, and the git layout quirk (⚠️ `client/` and `server/` are **separate git repos**; never run git from the `Hot-Potato/` root).

---

## 1. Vision & Golden Rules (non-negotiable)

These override any "best practice" instinct an implementing agent might have. If a change conflicts with a Golden Rule, the Golden Rule wins.

### 🥇 Golden Rule 1 — Never ration tokens for real students

**This project does NOT restrict how much a student uses the AI.** Students can chat as much as they want, whenever they want. There must be **no per-student token quota, no message cap, no "you've used your daily limit" UI, no paywall logic.**

Rate limiting exists **only** to stop bots and non-human activity (scripted abuse of the open endpoint burning the Gemini quota). Limits must be set so generously that no human student — even an enthusiastic one asking questions non-stop — ever hits them. When a limit *is* hit, assume it's a bot; the response is a plain 429, not a guilt-trip message.

### 🥇 Golden Rule 2 — Anonymous users keep FULL access

Students (including the owner 😂) are too lazy to log in first. **Users without an account must be able to read all public content AND use the AI fully — ask, answer, get feedback — with zero login wall.** The only difference for anonymous users: nothing is persisted for them (no history, no cross-session memory, no saved answers).

**This already works today** (`optionalAuth` on content routes, open `/api/chat/*`, client-side thread state). Every phase below must preserve it. When a phase adds a logged-in-only feature (memory, saved sessions), the anonymous path must degrade gracefully to the current client-side behavior — never to an error or a login prompt blocking the AI.

### The product principles (from the owner)

1. **Thai-first.** The primary audience is Thai students. AI output defaults to Thai (`AI_OUTPUT_LANGUAGE`), warm and casual. UI copy can stay mixed English/Thai for now.
2. **Critical thinking > correctness.** The goal is not scoring students — it's making them think, open up their ideas freely, and get dopamine from learning. Admire the idea first, then add depth. Never destroy an idea or make a student feel stupid.
3. **Students are low-tech AI newbies.** Many have never used a chatbot. The app must *navigate* them — teach "how to think with AI", not "how to follow AI". The AI should proactively suggest what to ask next, never leave a blank intimidating textbox.
4. **Teacher context drives the agent.** Teachers author the questions, guide answers, content, and guidelines. The agent must understand all of it and make the lesson fun to play through.
5. **The current code is a rough prototype.** Anything can be rewritten if it's worth it. **Phase -1 is complete** — agents verify with `npm test` first and the manual playbook (§7) second.

---

## 2. Current state (July 2026) — what exists and why it feels dumb

The full AI surface is `server/src/controllers/chat.controller.ts` (3 endpoints, mounted open/unauthenticated in `server/src/routes/chat.routes.ts`) plus the client bridge `client/src/components/editor/extensions/questionFeedbackApi.ts`.

| Endpoint | Used by | How it works today |
| --- | --- | --- |
| `POST /api/chat/ask` | `QuestionAgentView.tsx` (free "Ask AI" block) | Client serializes lesson text **above the block** (last 12,000 chars, `questionAgentContext.ts`), packs prior Q&A pairs from the answers store, sends everything as one flat text prompt. Stateless. |
| `POST /api/chat/feedback` | Choice/blank question views | Client computes `evaluationLevel` (exact-match %) and orders Gemini to obey it. Stateless. |
| `POST /api/chat/write-evaluate` | `QuestionWriteView.tsx` (open-ended) | One-shot coaching vs. teacher `guideAnswer`. Stateless. |

Structural problems (why the rework):

- **No conversation.** Every "turn" is a fresh one-shot prompt. The follow-up thread (`requestFeedbackFollowup()` in `questionFeedbackApi.ts`) is a hack that re-calls `/feedback` with hardcoded `evaluationLevel: "almost", accuracyPercent: 0` and stuffs the thread into the `diagnostics` field — this is the main reason the bot feels strict and robotic.
- **The server never sees the lesson.** It trusts a client-truncated text snippet even though the full `tiptap_json` sits in MongoDB. No attention across class content.
- **No memory.** The AI never knows what the student already asked, likes, or struggles with.
- **Deprecated SDK.** `@google/generative-ai` was deprecated Nov 30, 2025; the replacement is the unified **`@google/genai`** SDK. Model `gemini-2.5-flash-lite` still works but the lineup has moved on (Gemini 3.x).
- **Silent failure.** The client swallows every AI error and shows a canned Thai fallback string that *looks like real feedback*.
- **Prompt logging.** `console.log(prompt)` leaks student answers into Render logs.

What is already *right* and must be preserved: the prompt **intent** (admire first, never say "ผิด", friend tone, one actionable next step, teacher answer = reference not gospel, `quick_check`/`full_reflection` modes) and the anonymous-access model.

---

## 3. Target architecture

All AI logic consolidates into a **tutor agent service** on the server. In prose:

1. The client sends a lightweight request: *who* (JWT if present), *where* (`contentId` + `blockId`), *what mode* (free chat / question feedback / write evaluation / follow-up), and the student's message or answer. Anonymous clients also send their local thread (since the server stores nothing for them).
2. The server assembles context **itself**: it loads the lesson from MongoDB, serializes `tiptap_json` to text (including the teacher's questions and guide answers), loads the teacher's agent settings, and — for logged-in students — loads the chat session and student memory.
3. The server calls Gemini via `@google/genai` using **`systemInstruction`** (persona + teacher settings + lesson context) and a proper **multi-turn `contents` history** (real roles, not one flat string).
4. The response streams back (SSE) with the reply plus 2–3 suggested follow-up questions ("navigator" behavior).
5. For logged-in students the server persists the turn to `ChatSession` and asynchronously updates `StudentMemory`.

New server pieces:

| Piece | Location (new) | Role |
| --- | --- | --- |
| Tutor service | `server/src/services/tutor/` | Prompt building, Gemini calls, retry, streaming |
| Lesson context service | `server/src/services/lessonContext.service.ts` | Load + serialize + cache lesson text from `tiptap_json` |
| `ChatSession` model | `server/src/models/chat_session.model.ts` | Per (user, content, block) conversation history |
| `StudentMemory` model | `server/src/models/student_memory.model.ts` | Per user: interests, strengths, misconceptions |
| Rate limiter | `server/src/middlewares/rateLimit.middleware.ts` | Bot-only protection (Golden Rule 1) |

The old three endpoints stay alive during the transition and are removed only in Phase 5 when the client is fully rewired.

---

## 4. Phases

Ordering rationale: **tests first (the AI's eyes)**, then hygiene (cheap, de-risks everything), then the agent core (the foundation every other feature plugs into), then persona, memory, teacher settings, streaming/UX, and finally the dopamine layer.

### Phase -1 — Give the AI eyes: test harness before everything 👀 ✅ *Complete (2026-07-10)*

**Goal:** this project graduates from caveman-mode manual verification. AI agents will implement every later phase, so agents must be able to **prove their own work** — pull content, call the chat endpoints, check the contracts — without a human clicking through the UI each time.

**Why before Phase 0:** Phase 0 swaps the SDK and Phase 1 rewrites the AI core. Characterization tests that lock in *today's* behavior are the regression net that makes those rewrites safe. Write the net first, then jump.

- **-1.1 Tooling (server — the heavy half).** **Vitest** + **supertest** + **mongodb-memory-server** in `server/`. Three scripts in `package.json`:
  - `npm test` — unit + integration. Must pass with **no network, no real DB, no API key** (in-memory MongoDB, Gemini mocked). This is the command agents run constantly.
  - `npm run test:watch` — dev loop.
  - `npm run test:ai` — live Gemini smoke tests. Reads `GEMINI_API_KEY` from `server/.env` (via `test/setup.ts`); `vitest.ai.config.ts` sets `RUN_AI_TESTS=true`. Costs real tokens; run at phase boundaries, not every save.

  Note: mongodb-memory-server downloads a mongod binary on first run (~100MB, needs internet once; works on Windows).
- **-1.2 Fixture lesson (real teacher data, not toy data).** Script `server/scripts/export-fixture.ts`: dump the test lesson (`contents` ObjectId **`69e39d0b60d467bd515a4945`**) plus a test user from the real DB into `server/test/fixtures/*.json` (strip password hashes/secrets). All integration tests seed the in-memory MongoDB from these fixtures, so every test exercises a real authored lesson with real question blocks.
- **-1.3 Characterization tests of the current API.** Lock in today's behavior before anything changes:
  - **Auth:** register / login / recheck, and the exact structured-401 shape (`{ message, code, forceRelogin, clearToken }`) — the client's auto-logout parses this, so a test failure here means a broken client.
  - **Content:** `GET /content/load` across `access_type` (public / link-only / private) × caller (anonymous / owner / collaborator / stranger). **This is Golden Rule 2 as an executable test** — anonymous must read public content forever.
  - **Answers:** `content-answer` save / bulk / load roundtrip of the answers map (chat threads live in here).
  - **Chat endpoints (Gemini mocked):** 400 on missing fields, `evaluationLevel`/`feedbackMode` coercion rules, 200 response shapes for all three endpoints.
- **-1.4 Unit tests for AI-adjacent logic.** Prompt builders (`buildLearningPrompt` / `buildFeedbackPrompt` / `buildWriteEvaluationPrompt`) — snapshot the persona rules so a refactor that silently drops "never label the learner as wrong" **fails a test**; `isTransientAIError` classification; `generateWithRetry` with a fake model that fails N times (transient vs. fatal paths).
- **-1.5 Gemini mock seam.** In Phase -1, mock at the module boundary (`vi.mock("@google/generative-ai")`) so **zero production code changes** are needed. The proper injectable wrapper arrives naturally with Phase 1's tutor service — at which point the mocks move to that seam.
- **-1.6 Live AI smoke tests (`test:ai`).** Against the fixture lesson: `/chat/ask` returns a non-empty answer containing Thai characters (codepoint heuristic); `/chat/write-evaluate` returns non-empty feedback. These prove the key, model name, and retry wrapper still work against the real Gemini API.
- **-1.7 Client tests (light — server is the priority).** Vitest + happy-dom in `client/`: `questionAgentContext.ts` serializer (pure module — headings, lists, char cap, QuestionAgent exclusion), the accuracy-percent → evaluation-level math from the choice views, and `questionFeedbackApi.ts` fallback behavior with mocked axios. No heavy component testing yet.
- **-1.8 Working agreement + docs.** Update both `CLAUDE.md` files ("no tests yet" → test commands and where tests live). Add the standing rule for all implementing agents: **every later phase ships with tests for its own acceptance criteria, and a phase is not "done" until `npm test` is green in every half it touched.**

**Acceptance:** `cd server && npm test` passes offline with no API key (55 tests); `npm run test:ai` passes with a real key in `server/.env`; deliberately breaking the 401 shape or deleting a persona rule from a prompt builder makes the suite fail; `cd client && npm test` passes (16 tests). See [`ROADMAP-detailed.md`](ROADMAP-detailed.md) completion note for post-audit polish.

### Phase 0 — Hygiene & safety net (small)

**Goal:** stop the bleeding without changing behavior. The Phase -1 suite must stay green throughout — especially the chat-endpoint characterization tests across the SDK migration.

- **0.1 Migrate SDK** to `@google/genai` in `server/`. Keep `MODEL_NAME` behavior identical (`gemini-2.5-flash-lite` default, env overrides). Port `generateWithRetry` (the transient-error retry exists because the owner's ISP drops connections — keep it). Verify all 3 endpoints still answer.
- **0.2 Bot-only rate limiting** (Golden Rule 1). Add `express-rate-limit` on `/api/chat/*`, keyed by IP (+ user id when present). Suggested defaults, all env-configurable: `AI_RATELIMIT_PER_10MIN=60`, `AI_RATELIMIT_PER_DAY=1000` per key. A human asking 1–2 questions/minute never touches these. Render sits behind a proxy → set `app.set("trust proxy", 1)` in `app.ts` so IPs are real. **No auth requirement is added** — the routes gain `optionalAuth` (to identify logged-in users for later phases) but must keep working with no token (Golden Rule 2).
- **0.3 Gate prompt logging.** Replace `console.log(prompt)` with a check on `AI_DEBUG_PROMPTS=true`. Default off.
- **0.4 Honest errors on the client.** In `questionFeedbackApi.ts` and `QuestionAgentView.tsx`, distinguish "AI failed" from real feedback: on failure show a small retry UI (Thai copy, e.g. "ตอนนี้ AI ตอบไม่ได้ ลองกดส่งอีกครั้งนะ") instead of returning `FALLBACK_FEEDBACK` strings that masquerade as feedback. Keep the graceful-degradation *spirit* — never crash the block.

**Acceptance:** all 3 endpoints work logged-in *and* anonymous; a scripted 200-requests-in-a-minute loop gets 429s; a normal manual session never does; Render logs contain no student answers by default.

**Completion (2026-07):** Done. `cd server && npm test` — 57 tests green; `npm run test:ai` — 2 live smoke tests green (uses `--use-system-ca` like `npm run dev` on Windows). `cd client && npm test` — 16 tests green. Client shows `AiErrorRetry` box on failure instead of fake feedback strings.

### Phase 1 — Agent core: server-side context + real conversations (the foundation)

**Goal:** one unified tutor endpoint that owns context and history.

- **1.1 Lesson context service.** `lessonContext.service.ts`: given a `contentId`, load `Content.tiptap_json` and serialize to plain text. Port the block-serializer logic from `client/src/components/editor/extensions/questionAgentContext.ts` to the server (headings, paragraphs, lists, code, blockquote, image alt). **Additionally** extract question nodes (`QuestionChoice`, `QuestionWrite`, `QuestionBlankChoice`, `QuestionBlankWrite`) into a structured list: `{ blockId, type, question, choices?, guideAnswer }` — this is the teacher context the agent must know. Cache serialized results in memory keyed by `contentId + updatedAt` (lessons change rarely; cache invalidates naturally on edit).
- **1.2 `ChatSession` model.** `{ user_id, content_id, block_id, mode, messages: [{ role: "student"|"tutor", text, createdAt }], updatedAt }`, unique compound index on `(user_id, content_id, block_id)`. Cap stored messages (e.g. last 50) to bound document size.
- **1.3 Unified endpoint `POST /api/chat/tutor`** (guard: `optionalAuth` + rate limit). Request contract:

  ```jsonc
  {
    "contentId": "69e39d0b60d467bd515a4945",
    "blockId": "q-abc123",            // the anchoring block; null = lesson-wide chat
    "mode": "free_chat",              // free_chat | question_feedback | write_evaluation | followup
    "message": "student text / answer",
    "clientThread": [                  // anonymous users only — server-side session wins if logged in
      { "role": "student", "text": "...", "createdAt": "..." }
    ],
    "questionContext": {               // only for feedback/evaluation modes
      "question": "...",
      "guideAnswer": "...",
      "evaluation": { "level": "correct", "accuracyPercent": 100 },  // level may be "ai_judge"
      "feedbackMode": "quick_check"    // quick_check | full_reflection
    }
  }
  ```

  Response: `{ "reply": "...", "suggestions": ["...", "..."], "sessionId": "..." | null }`.

  Behavior: logged-in → load/append `ChatSession`, ignore `clientThread`; anonymous → use `clientThread` as history, persist nothing, `sessionId: null`. Both paths get identical AI quality (Golden Rule 2).
- **1.4 Gemini call shape.** Use `systemInstruction` for persona + lesson context + teacher question list; use the `contents` array with alternating roles for the conversation; the new message is the last user turn. Evaluation metadata (`questionContext`) goes into the final user turn, not the system prompt, so one session can span multiple questions.
- **1.5 Evaluation fix.** For choice questions the client still computes exact-match level (it's deterministic — fine). For open/blank-write answers, send `level: "ai_judge"` and let the model assess reasoning quality itself (this removes today's hardcoded `"almost"/0%` contradiction). The prompt rule becomes: *if a level is provided, follow it; if `ai_judge`, judge generously with critical-thinking priority.*
- **1.6 Keep the old endpoints untouched** so the current client keeps working until Phase 5.

**Acceptance:** with the test lesson (`contents` ObjectId **`69e39d0b60d467bd515a4945`**): a logged-in student chats 5+ turns on one block and the tutor remembers turn 1 without the client re-sending it; the same flow works anonymous (client sends thread); the tutor correctly references lesson sections that appear *below* the block (proving server-side full-lesson context); asking about a teacher question by number ("ข้อ 2 คืออะไร") works.

**Completion (2026-07-10):** Done (server only — client still calls legacy endpoints until Phase 5). `cd server && npm test` — 93 tests green; `npm run test:ai` — 2 live smoke tests green. New: `lessonContext.service.ts`, `ChatSession` model, `POST /api/chat/tutor`, `services/tutor/{retry,genaiClient,persona}.ts`. Retry helpers moved to `services/tutor/retry.ts` (re-exported from `chat.controller.ts` for Phase -1 test compatibility). Owner should manually smoke `/api/chat/tutor` against the real test lesson for full-lesson context + 3-turn name memory before Phase 2.

### Phase 2 — Persona & prompt system (Thai-first, the soul 🔥)

**Goal:** the tutor people want to talk to all day.

- **2.1 One persona, one file.** `server/src/services/tutor/persona.ts` exports the system-instruction builder. Design pillars (preserve the intent already in today's prompts, then go further):
  - Warm Thai friend-tutor: casual, playful, emoji-light; formulas/technical terms may stay in original language. Never "ผิด/wrong"; frame gaps as "เกือบแล้ว / ขาดอีกนิด".
  - **Admire first, always.** Find the genuinely good part of the student's idea and name it specifically *before* adding depth. Praise must be specific (not generic "เก่งมาก") so the dopamine feels earned.
  - **Navigator mode** for AI-newbie students: every reply ends with a light nudge on what to explore next; the model also emits 2–3 short suggested follow-up questions (the `suggestions` field — request JSON-structured output, or fall back to a `---SUGGESTIONS---` delimiter parse).
  - **Think-with-AI, not follow-AI:** when a student asks for a direct answer to a teacher question, the tutor coaches toward it (hint ladder: nudge → bigger hint → walk-through) instead of dumping the answer — *unless* the teacher's settings (Phase 4) allow direct answers.
  - Length discipline: `quick_check` short (2–4 sentences), `full_reflection` deeper (2–4 flowing paragraphs, no scorecards/bullet-walls). Free chat: match the student's energy; short question, shortish answer.
- **2.2 Model strategy** (env-configurable, with sensible 2026 defaults):

  | Task | Env var | Suggested default | Why |
  | --- | --- | --- | --- |
  | Quick checks, suggestions | `AI_FAST_MODEL` | `gemini-2.5-flash-lite` | Cheapest ($0.10/1M in); fine for short structured turns |
  | Free chat, reflection, write-eval | `AI_TUTOR_MODEL` | `gemini-2.5-flash` (override: `gemini-3.5-flash`) | Much better warm/nuanced Thai than Flash-Lite; still cheap |

  Both are on the free tier (Flash/Flash-Lite only since Apr 2026), which suits Golden Rule 1 — usage costs stay tiny even with unlimited student chatting. Keep `AI_WRITE_EVAL_MODEL`/`AI_HEAVY_MODEL` as legacy aliases.
- **2.3 Prompt quality loop.** Create `server/prompt-notes.md` logging real transcripts (Thai) + what felt off + the fix. The owner will iterate on tone here; implementing agents should make prompt text easy to edit without touching logic.

**Acceptance (manual, owner-judged):** a 10-turn Thai conversation on the test lesson feels like a friend, opens with specific admiration on every answered question, never dumps answers unprompted, and every reply surfaces tappable next questions.

**Completion (2026-07-10):** Done (server only). `cd server && npm test` — 116 tests green. `npm run build` green. New: full Phase 2 persona in `persona.ts` (`TUTOR_NAME = "น้องมันฝรั่ง"`), `parse.ts` with `[SUGGESTIONS]` parser, `resolveTutorModel` in `tutor.controller.ts` (`AI_FAST_MODEL` / `AI_TUTOR_MODEL`), `prompt-notes.md` with first entry. Live: `npm run test:ai` adds `/api/chat/tutor` free-chat smoke test. **Default override:** `AI_TUTOR_MODEL` defaults to `gemini-2.5-flash` (not `gemini-3-flash`) because the stable `gemini-3-flash` slug returned 404 from the API on 2026-07-10; `gemini-3.5-flash` and `gemini-3-flash-preview` work — set via env when desired. **Owner gate:** run a 10-turn manual conversation on test lesson `69e39d0b60d467bd515a4945` and sign off on tone before Phase 3.

### Phase 3 — Student memory (logged-in only)

**Goal:** "knows what I know, what I like, what I prefer."

- **3.1 `StudentMemory` model.** One doc per user: `{ user_id (unique), interests: string[], strengths: string[], growth_areas: string[], preferences: string[], recent_topics: [{ content_id, summary, updatedAt }], updatedAt }`. Free-text-ish short strings, capped counts (e.g. ≤10 each) — a sketch, not surveillance.
- **3.2 Async memory update.** After a tutor turn completes (logged-in only), fire-and-forget a cheap `AI_FAST_MODEL` call: given the last few turns + current memory, return updated memory JSON (or "no change"). Never block or fail the student's reply because of memory writes.
- **3.3 Injection.** The persona builder includes a compact memory digest in `systemInstruction`: "สิ่งที่รู้เกี่ยวกับผู้เรียนคนนี้: ..." with an explicit instruction to use it for warmth and continuity (e.g. connecting the lesson to a stated interest), never to pigeonhole or limit the student.
- **3.4 Anonymous path:** skipped entirely, silently. No prompts to "log in to unlock memory" inside the chat flow (a subtle hint elsewhere in the UI is fine).
- **3.5 Privacy floor:** memory is only ever injected into that same student's sessions; add a way to wipe it (a simple `DELETE /api/chat/memory` for the owner to call is enough for now).

**Acceptance:** student mentions liking football in lesson A; next day in lesson B the tutor naturally references football in an example. Anonymous flow unchanged.

**Completion (2026-07-10):** Done (server only). `cd server && npm test` — 132 tests green. New: `StudentMemory` model, `services/tutor/memory.ts` (`runMemoryUpdate` + `buildMemoryDigest`), memory digest injected in `tutor.controller.ts`, `GET`/`DELETE /api/chat/memory` (🔒 protect — the only protected chat routes). Memory updates fire-and-forget every 3 student turns (or on `write_evaluation`); caps enforced in code. **Owner gate:** live two-lesson manual test — mention an interest logged in, confirm cross-lesson warmth; `DELETE /api/chat/memory` clears it; incognito flow unchanged.

### Phase 4 — Teacher agent settings (per lesson)

**Goal:** teachers steer the tutor without writing prompts.

- **4.1 Schema.** Add optional `agent_settings` to `Content` (`content.model.ts`): `{ persona_note?: string, allow_direct_answers?: boolean (default false), scope?: "lesson_only" | "lesson_plus_general" (default lesson_plus_general), custom_guidelines?: string }`. All optional — every existing lesson works unchanged.
- **4.2 Editor UI.** A small "AI Tutor" section in the existing publish/settings modal (`PublishSettingsModal.tsx`) — plain textareas + toggles, teacher-friendly language, no prompt jargon.
- **4.3 Injection.** `persona_note` and `custom_guidelines` are appended to the system instruction under a clearly-labeled teacher section that ranks **below** the Golden-Rule persona (a teacher can make the tutor stricter about scope, but not cruel).
- **4.4 Contract discipline:** this touches `PUT /content/:id` payloads — update both halves together (root `CLAUDE.md` rule) and respect the optimistic-concurrency `clientUpdatedAt`/409 flow.

**Acceptance:** setting `allow_direct_answers: false` on the test lesson makes the tutor coach instead of answer; `persona_note: "พูดเหมือนโค้ชกีฬา"` visibly changes tone; lessons without settings behave exactly as Phase 2.

### Phase 5 — Streaming + client UX rewire

**Goal:** modern chatbot feel; retire the old endpoints.

- **5.1 SSE streaming.** `POST /api/chat/tutor` gains `"stream": true` → `text/event-stream` (`generateContentStream` in `@google/genai`). Events: `token` (text delta), `suggestions`, `done { sessionId }`, `error`. Non-stream mode stays for fallback.
- **5.2 Client rewire.** Point all four flows (`QuestionAgentView`, choice/blank feedback, write-eval, `FeedbackDiscussionPanel` follow-ups) at `/api/chat/tutor`. The follow-up hack in `questionFeedbackApi.ts` dies here — follow-ups become plain `mode: "followup"` turns in the same session. Keep block-level thread state in the answers store for anonymous users (it *is* their history).
- **5.3 Chat polish.** Streaming text with typing indicator (respecting Render cold-start: show a friendly "ปลุก AI แป๊บนึงนะ..." state when the first byte is slow); render tutor replies as markdown (bold, lists, code) — a light renderer like `react-markdown`; suggestion chips under each reply that fill the input on tap.
- **5.4 Remove `/ask`, `/feedback`, `/write-evaluate`** from `chat.routes.ts` + controller once nothing calls them. Update `server/CLAUDE.md` API table.

**Acceptance:** replies stream visibly within ~1s of first token; suggestion chips are tappable; anonymous + logged-in both work end-to-end on the test lesson; old endpoints return 404 and nothing in the client breaks.

### Phase 6 — Dopamine layer (design-heavy, do last, optional)

Rough ideas only — needs owner design input before implementation: celebration moments on completed questions (confetti-level, not casino), lesson progress feel ("คิดมาแล้ว 3 ข้อ 🔥"), streak-ish gentle nudges, teacher-visible engagement signals (which questions sparked the most chat). **No leaderboards, no scores compared between students** — that's the correctness-measuring trap this project explicitly avoids.

---

## 5. Careful-not-to-break list

| Thing | Where | Why it's fragile |
| --- | --- | --- |
| Anonymous full access | `optionalAuth` on content routes, open chat routes, client guards | **Golden Rule 2.** Adding `protect` anywhere in the AI/content read path is a regression. |
| Structured 401 contract | `auth.middleware.ts` ↔ `client/src/lib/axios.ts` | Client auto-logout parses the exact `{ forceRelogin, clearToken }` shape. |
| TipTap node = source of truth | `client/src/components/editor/*` | Never move canvas/question state into React state/context expecting persistence. See `client/src/components/README.md`. |
| Autosave + optimistic concurrency | `PUT /content/:id`, `clientUpdatedAt` → 409 | Phase 4 touches this payload. |
| Answers map shape | `UserContent.answers` (Map<blockId, any>) | Question views + agent blocks store their state here; changing shapes breaks saved student work. |
| Transient-error retry | `generateWithRetry` in chat controller | Exists because the owner's ISP drops connections; port it to the new service, don't delete it. |
| Error handler order | `app.ts` | `app.use(errorHandler)` must stay last. |

---

## 6. Env variables added by this roadmap

| Variable | Phase | Default | Purpose |
| --- | --- | --- | --- |
| `RUN_AI_TESTS` | -1 | `false` | Enable live Gemini smoke tests (`npm run test:ai`) — costs real tokens |
| `AI_RATELIMIT_PER_10MIN` | 0 | `60` | Bot guard, per IP/user — generous (Golden Rule 1) |
| `AI_RATELIMIT_PER_DAY` | 0 | `1000` | Same |
| `AI_DEBUG_PROMPTS` | 0 | `false` | Log full prompts only when debugging |
| `AI_FAST_MODEL` | 2 | `gemini-2.5-flash-lite` | Quick checks, suggestions, memory updates |
| `AI_TUTOR_MODEL` | 2 | `gemini-3-flash` | Free chat, reflection, write evaluation |

`GEMINI_API_KEY`, `AI_OUTPUT_LANGUAGE` unchanged. `AI_WRITE_EVAL_MODEL`/`AI_HEAVY_MODEL` become aliases for `AI_TUTOR_MODEL`.

---

## 7. Verification playbook

**After Phase -1 lands, automated tests come first:** `cd server && npm test` and `cd client && npm test` must be green before any manual checking, and `npm run test:ai` (live Gemini) at phase boundaries. The manual steps below are the second layer — for tone, UX feel, and anything a test can't judge.

1. Run both halves: `cd server && npm run dev` (port 5000) and `cd client && npm run dev` (port 5173).
2. Test lesson: MongoDB `contents` ObjectId **`69e39d0b60d467bd515a4945`** — a complete real lesson; open it via `/view/<id>`.
3. Always test **both auth states**: logged in, and a private/incognito window with no account.
4. Direct endpoint checks with `curl`/PowerShell `Invoke-RestMethod` against `http://localhost:5000/api/chat/tutor` (no token → must still answer).
5. AI answers take seconds (plus Render cold start in prod) — slow ≠ broken. Judge tone in Thai; the owner is the tone referee.
6. After changing any client↔server contract, grep the other half for the old shape before calling it done.

---

## 8. Suggested execution order & sizing

| Phase | Size | Depends on | Repo(s) |
| --- | --- | --- | --- |
| **-1 Test harness** | M | — | both (server-heavy) |
| 0 Hygiene | S | -1 | server (0.4: client) |
| 1 Agent core | L | -1, 0 | server |
| 2 Persona | M | 1 | server |
| 3 Memory | M | 1–2 | server |
| 4 Teacher settings | M | 2 | both |
| 5 Streaming + rewire | L | 1–2 | both |
| 6 Dopamine | ? | 5 | client |

Phases 3 and 4 are independent of each other and can swap order. Phase 5 can start (SSE server-side) in parallel with 3/4 if desired. **Every phase inherits the Phase -1 working agreement: ship tests for your own acceptance criteria, and you're not done until `npm test` is green in every half you touched.**
