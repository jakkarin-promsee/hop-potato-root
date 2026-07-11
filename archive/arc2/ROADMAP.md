# ROADMAP.md — Hot Potato Universal Roadmap

The single plan for **all** remaining work on Hot Potato — AI tutor, UI/UX, pages, hardening, and delight. Written for implementing agents (Claude, Cursor, …) and humans. The previous AI-tutor-only roadmap lives in [`archive/`](archive/) — it is **superseded by this file**; its Phases -1 → 3 shipped and their completion notes remain valid history.

> Read [`CLAUDE.md`](CLAUDE.md), [`server/CLAUDE.md`](server/CLAUDE.md), and [`client/CLAUDE.md`](client/CLAUDE.md) before touching code. ⚠️ `client/` and `server/` are **separate git repos**; never run git from the `Hot-Potato/` root.

---

## 1. Vision & Golden Rules (non-negotiable, carried forward)

These override any "best practice" instinct. If a change conflicts with a Golden Rule, the Golden Rule wins.

### 🥇 Golden Rule 1 — Never ration tokens for real students
No per-student quotas, no message caps, no "daily limit" UI, no paywall logic. Rate limiting exists **only** to stop bots, with env-tunable thresholds so generous no human ever hits them (`AI_RATELIMIT_PER_10MIN=60`, `AI_RATELIMIT_PER_DAY=1000`).

### 🥇 Golden Rule 2 — Anonymous users keep FULL access
No-login users read all public content **and** use the AI fully. The only logged-out difference: nothing persists (no history, no memory, no saved sessions). **Never add `protect` to an AI or public-content read path.** When a phase adds a logged-in-only feature, the anonymous path degrades gracefully — never to an error or a login wall.

### Product principles (from the owner)
1. **Thai-first.** AI output defaults to Thai, warm and casual. UI copy may stay mixed English/Thai.
2. **Critical thinking > correctness.** Admire the idea first, never say "ผิด", give dopamine. No scoring students against each other.
3. **Students are low-tech AI newbies.** The app *navigates* them — teach "how to think with AI", not "follow AI". Never leave a blank intimidating textbox.
4. **Teacher context drives the agent.** Questions, guide answers, and (soon) teacher settings shape every reply.
5. **Phone-first.** The primary student device is a phone. Every UI decision is judged at ~390 px width first, desktop second.

---

## 2. Current state (verified against code, 2026-07-10)

**Healthy baseline.** `cd server && npm test` → 203/203 green (offline). `cd client && npm test` → 52/52 green.

**Server — the old roadmap's Phases -1 → 3 are complete:**
- `POST /api/chat/tutor` (`tutor.controller.ts`): unified agent, 4 modes (`free_chat | question_feedback | write_evaluation | followup`), server-side lesson context (`lessonContext.service.ts`), `ChatSession` persistence (logged-in), `clientThread` pass-through (anonymous).
- Phase 2 persona (`services/tutor/persona.ts`, `TUTOR_NAME = "น้องมันฝรั่ง"`), `[SUGGESTIONS]` parser (`parse.ts`), model split (`AI_FAST_MODEL` / `AI_TUTOR_MODEL`).
- Phase 3 `StudentMemory`: async updates every N turns, digest injection, `GET`/`DELETE /api/chat/memory` (the only protected chat routes).
- Bot-only rate limiting, `@google/genai` SDK, `generateWithRetry`.

**Client — rewired to the unified tutor (Tier 0.A, 2026-07-10).** All six AI surfaces call `POST /api/chat/tutor` via `tutorApi.ts` / `callTutorStream`; the legacy endpoints and the `requestFeedbackFollowup()` hack are deleted. [`asking-flow.md`](asking-flow.md) documents the final per-mode context flow with measured sizes.

**Tier 1 — Tutor Experience Upgrade shipped 2026-07-10 (agent-executed; owner tone-gates pending):**
- **1.A:** SSE streaming (`stream: true` → `token`/`suggestions`/`done`/`error` events); `createSuggestionsGate`; client `callTutorStream` with axios fallback; cold-start hint ("ปลุก AI แป๊บนึงนะ…") on all six surfaces.
- **1.B:** `Content.agent_settings` (teacher persona note, coach/direct-answer policy, scope, custom guidelines); injected in `persona.ts` below Golden-Rule pillars; **AI Tutor** section in `PublishSettingsModal`.
- **1.C:** Six student personality presets (`personality.ts`); `StudentMemory.tutor_personality` sync; `PersonalityPicker` in Ask-AI modal + `/settings`; localStorage for anonymous users.
- **Remaining owner gates:** live tone transcripts per preset + teacher-settings lesson in `server/prompt-notes.md`; `npm run test:ai` at tier boundary. See [`plan/tier1.md`](plan/tier1.md).

### Owner-gate findings (manual testing, 2026-07-10)

The owner verified Phases 1–3 manually. Tone is good overall, but five concrete problems surfaced — they drive Tier 0:

| # | Finding | Root cause (traced) |
| --- | --- | --- |
| P1 | Small talk is impossible — "สวัสดี" gets evaluated as an answer | Every follow-up goes through `buildFeedbackPrompt` (legacy `/chat/feedback`), which frames *all* input as `=== User Answer ===` with a forced evaluation level. No intent concept anywhere. |
| P2 | Replies show raw `**bold**` asterisks — plain-text rendering is hard to read | Model outputs markdown; client renders it as plain text. No renderer, and no format rules telling the model what the client can display. |
| P3 | Robotic headers leak into replies (e.g. `**Action Plan:** …`) | Legacy prompts order "End with one actionable next step" / "suggest one concrete improvement plan" — the model formats that as a labeled section. No output-format guard strips it. |
| P4 | Prompt behavior is inconsistent across question cards vs. the main Ask chat | Three different legacy prompt builders + one new persona = four prompt systems live at once. Context assembly differs per flow (client-truncated text vs. server-side lesson). |
| P5 | Follow-up chat renders nested inside the question card — x-margins stack up, unusable on phone | `FeedbackDiscussionPanel` is a bordered box inside the feedback card inside the question card; bubbles get `max-w-[90%]` of an already-shrunken column. |

Plus two product decisions from the owner:
- **P6:** All pages stay — audit each and finish what's missing (see §9 Pages audit).
- **P7:** New feature — student-selectable AI personalities — **shipped in Tier 1.C** (2026-07-10); owner tone-gate pending.

---

## 3. How this roadmap is organized

- **Tiers = priority bands.** Finish a tier before starting the next (exceptions noted per phase). **Current focus: Tier 4** (Tiers 0–3 shipped; owner tone-gates pending on Tiers 0–1 and 3.B tutor default).
- **Phases = shippable units** inside a tier, each with goal, work items, and acceptance criteria. Phases inside a tier may run in the listed order unless marked independent.
- **Working agreement (inherited):** every phase ships tests for its own acceptance criteria; a phase is not done until `npm test` is green in every half it touched, and the relevant `CLAUDE.md` is updated.
- **The owner is the tone referee.** Any phase touching prompts ends with a manual Thai conversation gate, logged in [`server/prompt-notes.md`](server/prompt-notes.md).

---

## 4. Tier 0 — Conversation Core 🔥 — ✅ SHIPPED 2026-07-10 (owner tone-gate pending)

**Theme: make the default tutor feel right on a phone.** Fixes P1–P5. One endpoint, one persona, one context flow, markdown that renders, UI that fits a phone. Everything later builds on this.

> **Status (2026-07-10, agent-executed overnight):** 0.A → 0.D implemented and committed (server `b4f91d8`+`7cad1b4`, client `7a7424e`+`dbcb1c5`+`e69a655`). Suites green (server 120, client 28). Live-verified at 390 px on the test lesson, anonymous: small talk gets greetings (P1 dead), markdown renders, chips send, no horizontal scroll — proof screenshots in `plan/tier0-audit/`. **Remaining owner gates:** real-phone eyeball (0.C) and 4 × 10-turn Thai tone transcripts + sign-off (0.D) — see `server/prompt-notes.md`. `asking-flow.md` rewritten with measured context sizes.

### Phase 0.A — Client rewire to `/api/chat/tutor` (kills the follow-up hack)

**Goal:** all four client flows speak to the unified tutor; the legacy prompt zoo dies. This single move eliminates the root cause of P1 and most of P4.

- Point the four flows at `POST /api/chat/tutor` with proper modes:
  | Flow | File | Mode |
  | --- | --- | --- |
  | First submit, choice/blank cards | `QuestionChoiceView.tsx`, `QuestionBlank*View.tsx` | `question_feedback` (client still computes exact-match level — deterministic, keep) |
  | First submit, write card | `QuestionWriteView.tsx` | `write_evaluation` (server judges via `ai_judge` semantics) |
  | "Submit another answer?" thread | `FeedbackDiscussionPanel` via parents | `followup` — a plain conversation turn, **no forced evaluation** |
  | Ask AI (lesson-wide modal) | `TiptapViewer.tsx` | `free_chat` with `blockId: null` |
- Delete `requestFeedbackFollowup()` from `questionFeedbackApi.ts`; replace the bridge with a thin `tutorApi.ts` (request/response typed to the tutor contract: `{ reply, suggestions, sessionId }`).
- Client stops sending lesson text (server owns context). Keep sending a **light reading-position hint** (the current `getQuestionAgentViewportContext` heading) as an optional `currentSection` field; server appends it to the final user turn. Server change is small — extend the tutor request contract.
- **Intent-aware persona rule (server, `persona.ts`):** in `followup`/`free_chat`, classify intent implicitly — greeting/chit-chat/meta-question gets a friendly conversational reply, never an evaluation. Only `question_feedback`/`write_evaluation` turns evaluate, and only the submitted answer. Add unit tests asserting the rule text survives refactors (same trick as the old Phase -1 persona snapshots).
- Anonymous path: block-level thread state stays in `content-answer.store` and is sent as `clientThread` (it *is* their history). Logged-in path: server session wins.
- **Retire legacy endpoints** once nothing calls them: remove `/ask`, `/feedback`, `/write-evaluate` from `chat.routes.ts` + `chat.controller.ts`, delete their prompt builders, update `server/CLAUDE.md` API table and the Phase -1 characterization tests that pinned them.

**Acceptance:** typing "สวัสดี" into any thread gets a friendly greeting, not feedback (P1 dead). A 5+ turn conversation on one card remembers turn 1 both logged-in and anonymous. All four flows work on the test lesson (`/view/69e39d0b60d467bd515a4945`) in both auth states. Legacy endpoints return 404; client + server suites green.
**Size:** L · **Repos:** both

### Phase 0.B — Markdown pipeline & output format discipline

**Goal:** replies read beautifully; no robotic section headers ever leak (P2 + P3).

- Client: render tutor replies as markdown (`react-markdown` or a comparably light renderer) in all four surfaces — bold, lists, inline code; sanitize/limit to a safe subset. Match existing bubble styling.
- Server persona: add an explicit **format contract** to `buildSystemInstruction` — "you may use light markdown (bold, short lists); NEVER use section labels like 'Action Plan', 'Feedback:', 'สรุป:' — you are texting a friend, not writing a report."
- Parse guard in `parse.ts`: strip any leading/trailing labeled-section artifacts the model still emits (defense in depth, mirrors the `[SUGGESTIONS]` parser).
- Log a prompt-notes entry for the change.

**Acceptance:** a `full_reflection` reply renders formatted (no raw `**`); 10 manual turns produce zero "Action Plan"-style headers; unit tests cover the parse guard.
**Size:** S–M · **Repos:** both

### Phase 0.C — Phone-first chat UI (de-nest the thread)

**Goal:** the follow-up conversation feels like a real chat, not a box-in-a-box-in-a-box (P5).

- Redesign `FeedbackDiscussionPanel` so the thread does **not** nest inside stacked bordered cards: flatten to a full-width thread under the question card (drop the inner border+background layers; bubbles span the usable column). Consider expanding into a bottom-sheet / full-height takeover on phone widths for long threads.
- Suggestion chips: render the tutor's `suggestions[]` as tappable chips under the latest reply that fill (or send) the input — the "navigator" behavior the persona already emits. (Moved up from the old Phase 5.3 because it pairs with this redesign.)
- Audit all AI surfaces at ~390 px: input row, send button targets (≥44 px), Ask-AI modal margins.
- Owner eyeballs on a real phone before sign-off.

**Acceptance:** on a 390 px viewport, chat bubbles use ≥85% of the card's usable width with a single visual nesting level; suggestion chips tappable in every flow; no horizontal scroll anywhere in `/view/:id`.
**Size:** M · **Repos:** client

### Phase 0.D — Context-flow & prompt consistency pass

**Goal:** one documented, deliberate context flow; the tutor feels consistent and *fun* everywhere (rest of P4). Input: [`asking-flow.md`](asking-flow.md) + post-0.A reality.

- Trace and document the **final** context flow (replacing `asking-flow.md`, which describes the legacy wiring): per mode — what the system instruction contains, what the final user turn contains, what history is loaded, token budget per section.
- Tune deliberately: does `question_feedback` need the *whole* lesson or the question + nearby section? Does `free_chat` benefit from recent-answers context? Trim what doesn't earn its tokens (faster + cheaper + more focused replies).
- Prompt-quality iteration loop with the owner: at least one 10-turn transcript per flow in `prompt-notes.md`, tweaks applied, re-tested. Target feel: "อยากคุยต่อ" — playful, curious, zero report-speak.
- Update `asking-flow.md` (or fold it into `client/CLAUDE.md` + `server/CLAUDE.md`) so docs match the new single-endpoint reality.

**Acceptance (owner-judged):** the owner plays through the full test lesson — every card type + main Ask — and signs off that tone/behavior feel consistent and fun across all of them.
**Size:** M · **Repos:** both (mostly server prompts + docs)

---

## 5. Tier 1 — Tutor Experience Upgrade — ✅ SHIPPED 2026-07-10 (owner tone-gates pending)

**Theme: from "works" to "feels modern".** Shipped agent-executed; owner sign-off on tone and teacher-settings behavior still pending (`server/prompt-notes.md`).

> **Status (2026-07-10):** 1.A → 1.C implemented and committed (server + client). Suites green (server 174, client 28). **Remaining owner gates:** live Thai tone transcripts (6 presets + settings-enabled lesson), real-phone smoke at ~390 px, `npm run test:ai`. Spec: [`plan/tier1.md`](plan/tier1.md).

### Phase 1.A — SSE streaming + cold-start UX — ✅

- `POST /api/chat/tutor` gains `"stream": true` → `text/event-stream` via `generateContentStream`. Events: `token`, `suggestions`, `done { sessionId }`, `error`. Non-stream mode stays as fallback.
- Client: streaming text with typing indicator; friendly cold-start state ("ปลุก AI แป๊บนึงนะ…") when first byte is slow (Render free tier).
- **Acceptance:** first token visibly streams within ~1s of server response start; suggestions arrive at the end; fallback path still works.
- **Size:** L · **Repos:** both

### Phase 1.B — Teacher agent settings (per lesson) — ✅

- `Content.agent_settings` (all optional): `{ persona_note?, allow_direct_answers? (default false), scope? ("lesson_only" | "lesson_plus_general", default lesson_plus_general), custom_guidelines? }`.
- Editor UI: an "AI Tutor" section in `PublishSettingsModal.tsx` — teacher-friendly language, no prompt jargon.
- Injection: teacher section ranks **below** the Golden-Rule persona (teachers can steer scope/strictness, not cruelty).
- Touches `PUT /content/:id` — update both halves together; respect the `clientUpdatedAt`/409 optimistic-concurrency flow.
- **Acceptance:** `allow_direct_answers: false` makes the tutor coach instead of answer; `persona_note: "พูดเหมือนโค้ชกีฬา"` visibly changes tone; lessons without settings behave exactly as before.
- **Size:** M · **Repos:** both

### Phase 1.C — Student-selectable AI personalities (P7) — ✅

- A small set of curated presets layered on top of the default persona (which stays the optimized Tier-0 baseline): e.g. สุภาพ ↔ แสบๆ ขี้แซว, อธิบายละเอียด ↔ กระชับ, ชิลล์ ↔ จริงจัง. Each preset = a short persona-modifier block injected into `buildSystemInstruction` **below** Golden-Rule pillars (a personality can change flavor, never cruelty or answer-dumping rules).
- Picker UI: student-facing, in the chat surface (and/or `/settings`). Logged-in: persist choice (`StudentMemory.tutor_personality`). Anonymous: localStorage.
- Precedence: teacher sets boundaries, student sets flavor (personality block subordinated to teacher section).
- **Acceptance:** switching personality visibly changes tone mid-lesson without losing thread context; anonymous choice survives a reload; teacher restrictions still hold under every personality.
- **Size:** M · **Repos:** both

---

## 6. Tier 2 — Product Completeness (pages & accounts) — ✅ SHIPPED 2026-07-10

**Theme: no more mock pages.** All pages stay (owner decision, P6). Independent phases — any order.

> **Status (2026-07-10):** 2.A → 2.C implemented and committed (server + client). Suites green (server 181, client 52). Spec: [`plan/tier2.md`](plan/tier2.md).

### Phase 2.A — Real Profile page — ✅

- Today `Profile.tsx` is 100% mock: local state only, Save does nothing, avatar button dead. Server `UserData` model exists but **no controller/route uses it**.
- Wire it end-to-end: `GET/PUT /api/users/me/profile` (new, 🔒 protect) backed by `UserData` (`avatar`, `bio`, `metadata` for nickname/age/role-display); avatar upload via the existing Cloudinary flow; load real `user.name`/`email` from `auth.store`.
- Decide in-phase what actually matters for students (this is a learning app, not a social profile) — cut fields that serve nothing.
- **Acceptance:** edit → save → reload → data persists; avatar upload works; anonymous users never see a broken page (route already `RequireLogin`).
- **Size:** M · **Repos:** both

### Phase 2.B — Login-flow UX cleanup — ✅

- Known cleanup target (noted in both `CLAUDE.md`s): `Login.tsx`, the three route guards, and `lib/axios.ts` force-relogin handling — touched together.
- Scope (define precisely in-phase): clearer login/register switching, readable error states, `?reason=` messaging on forced logout, redirect-back-to-where-you-were after login, Thai copy polish.
- **Acceptance:** forced-logout lands on login with a human explanation and returns you to your page after re-auth; register/login validation errors are specific; owner signs off on flow feel.
- **Size:** S–M · **Repos:** client (server: register password ≥ 8 aligned with change-password)

### Phase 2.C — Page polish sweep — ✅

Close the small gaps from the pages audit (§9): delete the dead `Index.tsx` stub (+ its import in `App.tsx`), refresh `Guide.tsx` content to match current features (it predates the tutor), Landing copy check, and a pass over empty/loading/error states on Explore/Dashboard/History.
- **Acceptance:** audit table in §9 shows no ❌ left; `npm run lint` clean.
- **Size:** S · **Repos:** client

---

## 7. Tier 3 — Hardening & Ops — ✅ SHIPPED 2026-07-10

**Theme: safe to leave running unattended.** Do before promoting the app beyond friendly users.

> **Status (2026-07-10):** 3.A → 3.B implemented and committed (server). Suite green (203 tests). Spec: [`plan/tier3.md`](plan/tier3.md). **Owner actions after deploy:** set `CORS_ORIGINS` on Render, run `npm run promote -- <email>`, production smoke from Vercel. **Owner tone-gate pending** on `gemini-3.5-flash` tutor default (3.B).

### Phase 3.A — Security & API hygiene — ✅

- CORS env-driven allowlist (`CORS_ORIGINS`); fail-open when unset with boot warning.
- `/api/users`: removed open `POST /`; `GET /` admin-only, no password hashes.
- Registration hardened: `role` ignored, validation for name/email/password.
- `admin` role gates diagnostics; `npm run promote` script for bootstrap.
- Account `status` enforced on every `protect` call; `optionalAuth` degrades blocked users to anonymous.
- Status endpoints sanitized (no Atlas host leak); `GET /api/status/` admin-only.
- Auth rate limiting on login/register (`AUTH_RATELIMIT_PER_10MIN`, default 100).
- Production 500s no longer echo internal error messages.
- `AI_DEBUG_PROMPTS` reimplemented (dev-only prompt logging).
- **Acceptance:** `cd server && npm test` green (203); decisions in `server/CLAUDE.md`.
- **Size:** S–M · **Repos:** server

### Phase 3.B — Model strategy upkeep — ✅

- `npm run check-models` script probes Flash lineup; defaults centralized in `services/tutor/models.ts`.
- Tutor default updated to stable `gemini-3.5-flash`; fast default stays `gemini-2.5-flash-lite`.
- **Acceptance:** documented in `prompt-notes.md`; owner tone sign-off pending on new tutor default.
- **Size:** XS · **Repos:** server

---

## 8. Tier 4 — Dopamine Layer ✨ (design-gated, last)

Rough ideas only — **needs owner design input before any implementation**: celebration moments on completed questions (confetti-level, not casino), lesson progress feel ("คิดมาแล้ว 3 ข้อ 🔥"), gentle streak-ish nudges, teacher-visible engagement signals (which questions sparked the most chat). **No leaderboards, no student-vs-student scores** — that's the correctness-measuring trap this project explicitly avoids.

Prerequisite: Tier 0 complete (celebrations attach to the chat/feedback moments Tier 0 rebuilds).

---

## 9. Pages audit (P6 — status as of 2026-07-10)

All pages stay. ✅ = complete for now · 🔧 = has a phase above · ❌ = broken/dead.

| Route | Page | Status | Missing / notes |
| --- | --- | --- | --- |
| `/` | `Landing` | ✅ | AI tutor card added (2.C); showcase cards still fictional — owner decision. |
| `/login` | `Login` | ✅ | Redirect-back, Thai session banner, field validation, dead buttons removed (2.B). |
| `/explore` | `Explore` | ✅ | Persistent device-local bookmarks via `bookmark.store` (2.C). |
| `/guide` | `Guide` | ✅ | Refreshed for AI tutor + critical-thinking questions (2.C). |
| `/dashboard` | `Dashboard` | ✅ | Wired. |
| `/create` | `Create` | ✅ | Wired. |
| `/canvas/:id` | `TipTapCanvas` | ✅ | Editor + AI Tutor settings in `PublishSettingsModal` (1.B). |
| `/view/:id` | `TiptapView` | ✅ | Student surface: streaming tutor, personality picker, phone-first chat (Tier 0 + 1). |
| `/history` | `History` | ✅ | Wired (`RequireLogin`). |
| `/profile` | `Profile` | ✅ | Real page: `profile.store` + Cloudinary avatar + `GET/PUT /users/me/profile` (2.A). |
| `/change-password` | `ChangePassword` | ✅ | Wired. |
| `/settings` | `Setting` | ✅ | Theme + language + personality picker (1.C). |
| `/uploadimage` | `Cloudinaryupload` | ✅ | Wired. |
| `/status` | `Status` | ✅ | Wired. |
| `*` | `NotFound` | ✅ | Static. |

---

## 10. Careful-not-to-break list

| Thing | Where | Why it's fragile |
| --- | --- | --- |
| Anonymous full access | `optionalAuth` on content + chat routes, client guards | **Golden Rule 2.** Adding `protect` to an AI/public-read path is a regression. |
| Structured 401 contract | `auth.middleware.ts` ↔ `client/src/lib/axios.ts` | Client auto-logout parses the exact `{ forceRelogin, clearToken }` shape. |
| TipTap node = source of truth | `client/src/components/editor/*` | Canvas/question state lives in node attrs; React state/context won't persist. |
| Autosave + optimistic concurrency | `PUT /content/:id`, `clientUpdatedAt` → 409 | Phase 1.B touches this payload. |
| Answers map shape | `UserContent.answers` (Map<blockId, any>) | Question views + threads store state here; shape changes break saved student work. Phase 0.A must keep anonymous thread storage compatible. |
| Transient-error retry | `generateWithRetry` (`services/tutor/retry.ts`) | Exists because the owner's ISP drops connections. Port, never delete. |
| Error handler order | `server/src/app.ts` | `app.use(errorHandler)` must stay last. |
| `[SUGGESTIONS]` output contract | `persona.ts` ↔ `parse.ts` | Client chips (0.C) will depend on it; format changes need both sides + tests. |

---

## 11. Environment variables (current + planned)

All existing vars unchanged — see [`server/CLAUDE.md`](server/CLAUDE.md) for the full table (`GEMINI_API_KEY`, `AI_OUTPUT_LANGUAGE`, `AI_FAST_MODEL`, `AI_TUTOR_MODEL`, `AI_MEMORY_EVERY_N_TURNS`, `AI_DEBUG_PROMPTS`, `AI_RATELIMIT_*`, …). This roadmap adds **no new env vars** through Tier 1; if 1.C personality presets need a kill switch, prefer a code constant over an env knob.

---

## 12. Verification playbook

1. **Automated first:** `cd server && npm test` and `cd client && npm test` green before any manual check; `npm run test:ai` (live Gemini, costs tokens) at phase boundaries only.
2. Run both halves: server `:5000`, client `:5173`. Test lesson: `/view/69e39d0b60d467bd515a4945`.
3. Always test **both auth states** — logged in and incognito (Golden Rule 2 is an executable requirement).
4. Phone check: every Tier-0/2 UI change gets a ~390 px viewport pass (real device for sign-off).
5. AI latency + Render cold start: slow ≠ broken. The owner judges tone, in Thai, and logs prompt findings in `server/prompt-notes.md`.
6. After changing any client↔server contract, grep the other half for the old shape before calling it done.

---

## 13. Sizing & suggested order

| Phase | What | Size | Repos | Depends on |
| --- | --- | --- | --- | --- |
| **0.A** | Client rewire + intent-aware persona | L | both | — |
| **0.B** | Markdown pipeline + format guard | S–M | both | 0.A |
| **0.C** | Phone-first chat UI + suggestion chips | M | client | 0.A |
| **0.D** | Context-flow + prompt consistency pass | M | both | 0.A–0.C |
| 1.A | SSE streaming + cold-start UX | L | both | Tier 0 |
| 1.B | Teacher agent settings | M | both | Tier 0 (independent of 1.A) |
| 1.C | Student personalities | M | both | 0.D (tuned default first — owner rule) |
| 2.A | Real Profile page | M | both | — (independent) | ✅ |
| 2.B | Login-flow UX cleanup | S–M | client | — (independent) | ✅ |
| 2.C | Page polish sweep | S | client | — (independent) | ✅ |
| 3.A | Security & API hygiene | S | server | before wider release |
| 3.B | Model strategy upkeep | XS | server | recurring |
| 4 | Dopamine layer | ? | client | Tier 0 + owner design |

**Recommended path:** 0.A → 0.B → 0.C → 0.D (one continuous arc — each unlocks the next), then **Tier 1 ✅** → Tier 2 phases as breathers. Tier 3.A before sharing the app beyond friendly users.
