# ideas.md — why it's built this way (decision records)

> Updated 2026-07-11. Lightweight ADRs (Architecture Decision Records): the *non-obvious* choices, why they were made, and **what breaks if you "fix" them.** These are the traps that make a stranger revert something the owner did on purpose. When you make a new load-bearing decision, add one.
>
> Format per entry: **Context → Decision → Why → Consequence (what breaks if violated).** Sources: [`../ROADMAP.md`](../ROADMAP.md) §"Careful-not-to-break", the `CLAUDE.md` gotchas, `prompt-notes.md`, and the code.

---

### ADR-001 — Two separate git repos, not a monorepo
**Context:** `client/` and `server/` deploy to different platforms (Vercel/Render) and were built at different times. **Decision:** keep them as two independent git repos; `Hot-Potato/` is an untracked workspace. **Why:** each host auto-deploys from its own repo; no monorepo tooling to maintain on a solo project. **Consequence:** never run `git` from the workspace root — a stray `git init` in the home dir surfaces unrelated files. Cross-cutting docs (this folder, `ROADMAP*.md`) live at the root and belong to neither repo. Change an endpoint? Update both repos. (Root [`CLAUDE.md`](../CLAUDE.md) §"Git reality".)

### ADR-002 — The server owns lesson context; the client never sends lesson text to the AI
**Context:** every AI call needs the lesson's content. **Decision:** the client sends only `contentId`; the server loads + serializes the lesson (`lessonContext.service`, cached by `updatedAt`) and injects it. **Why:** the lesson is trusted server data, not user input — keeping it server-side blocks forged context/prompt-injection, shrinks payloads, and centralizes "what the AI sees" in one file (which also renders formulas/videos/tables/drawings into text). **Consequence:** re-introducing client-sent lesson text re-opens trust holes and drifts the two halves. (Detail: [`../asking-flow.md`](../asking-flow.md); idea ② in [`ARCHITECTURE.md`](ARCHITECTURE.md).)

### ADR-003 — The TipTap document is the single source of truth
**Context:** the editor mixes rich text, embedded Fabric canvases, formulas, and question blocks. **Decision:** all block data lives in the TipTap node's `attrs`; nothing authoritative in React state, hooks, or context. Fabric canvases sync back via a `saveState` bridge; `CanvasContext` is only a UI pointer. **Why:** one serialization path gives autosave, undo, and the read-only viewer everything for free. **Consequence:** data stashed in React state silently isn't saved (autosave serializes the doc, and it isn't in it). ([`../client/src/components/README.md`](../client/src/components/README.md).)

### ADR-004 — `optionalAuth` (never `protect`) on AI + public-read routes
**Context:** the audience is students far from good schools; login friction loses them. **Decision:** AI endpoints and public-content reads use `optionalAuth` — anonymous works fully, login only *adds* persistence. **Why:** Golden Rule 2 as an architectural invariant, not a feature. **Consequence:** adding `protect` to an AI or public-read path is a **product regression**. `/settings`, `/status`, `/view/:id` must all work logged-out. Blocked/suspended tokens degrade to anonymous under `optionalAuth` (they don't error). (Guard placement: [`data-flow.md`](data-flow.md) §1.)

### ADR-005 — Never ration tokens for real students; rate-limit bots only, per-surface
**Context:** LLM calls cost money; the instinct is per-user quotas. **Decision:** no per-student quotas/caps/paywall; rate limits are bot-only with very generous env-tunable thresholds, and **each AI surface has its own limiter bucket** (student tutor vs teacher copilot). **Why:** Golden Rule 1 — the product exists to give under-resourced students *unlimited* access. **Consequence:** any "usage limit" on the student path contradicts the product; the teacher copilot must never share the student's rate bucket.

### ADR-006 — Observability is in-memory and wiped on restart
**Context:** the owner wanted a public "is it working?" page but won't pay for a monitoring SaaS (free-tier, solo). **Decision:** an in-process error ring buffer + AI-health snapshot on `GET /status/all`; resets on every Render restart. **Why:** zero cost, zero deps; "since last restart" is honest enough for this scale. **Consequence:** never store `err.message` or request bodies in it (route + code + error name only; names admin-gated); don't build features assuming persistent metrics.

### ADR-007 — Viewer zoom uses CSS `zoom`, not `transform: scale()`
**Context:** the lesson viewer needs a zoom control. **Decision:** apply CSS `zoom` on the card container. **Why:** `zoom` scales the *layout box* (scroll extent matches what's on screen); `transform: scale()` only scaled pixels, leaving a second scrollbar / dead scroll space. **Consequence:** don't switch to `transform: scale()` — it reintroduces the double-scrollbar bug. Custom nodes (incl. Fabric) then need no own zoom logic. In the viewer, the window is the only vertical scroller.

### ADR-008 — App-wide font size and viewer zoom are separate mechanisms
**Context:** both "make text bigger" knobs exist. **Decision:** keep them distinct — `appearance.store` sets an inline `%` on `<html>` (whole app, rem-based); the viewer's CSS `zoom` (ADR-007) scales only the lesson card. **Why:** they serve different needs (global accessibility vs per-lesson reading) and merging them tangles two scroll models. **Consequence:** don't unify them; `--app-nav-height` stays px on purpose.

### ADR-009 — Teacher AI writes into the doc only via preview → accept, never as raw TipTap JSON
**Context:** the copilot can draft sections, questions, formulas. **Decision:** AI output is shown as a preview; only on explicit accept does it enter the doc as a normal editor transaction. Prose returns as **markdown**; question blocks as **typed JSON validated server-side** (malformed items dropped). **Why:** one human checkpoint keeps lessons trustworthy; trusting raw model JSON as document nodes would corrupt lessons. **Consequence:** never auto-apply; never let the model emit raw TipTap doc JSON. Autosave/409 stay untouched by the copilot. (Idea ⑤; [`data-flow.md`](data-flow.md) §6.)

### ADR-010 — Tutor replies are light markdown only, with strip guards
**Context:** early replies leaked robotic "**Action Plan:**" labels and raw `**bold**`. **Decision:** persona enforces `FORMAT (STRICT)` (light markdown, no headings/tables/labels — "texting a friend, not writing a report"); defense-in-depth `stripReportLabels` + `stripInternalTags` in `parse.ts`; client renders via `MarkdownMessage.tsx` (strict allowlist, raw HTML skipped). **Why:** tone + safety (no injected labels, no leaked internal tags reaching/persisting to the student). **Consequence:** the format is a three-file contract (`persona.ts` ↔ `parse.ts` ↔ `MarkdownMessage.tsx`) — change all three + tests together. (`prompt-notes.md` 2026-07-10.)

### ADR-011 — The structured 401 is a two-sided contract
**Context:** the client must auto-logout on a dead token but stay put on ordinary 401s. **Decision:** `protect` returns exactly `{ message, code, forceRelogin: true, clearToken: true }`; the axios response interceptor keys off `forceRelogin`/`clearToken`. **Why:** a precise shape lets the client distinguish "log in again" from other failures, and redirect only on protected paths. **Consequence:** change the body shape on one side and silent-logout breaks — update `auth.middleware.ts` **and** `lib/axios.ts` together. (Detail: [`data-flow.md`](data-flow.md) §1–2.)

### ADR-012 — Deterministic evaluation on the client; AI *coaches*, and only evaluates when tagged
**Context:** grading choice/blank questions in the LLM is wasteful and unreliable; and small talk shouldn't be graded. **Decision:** the client computes the verdict (level/accuracy/diagnostics via `questionEvaluation.ts`) and passes it in; the server gates *evaluation* on the exact `[งานของเธอตอนนี้…]` task tag — everything else (greetings, follow-ups) is plain conversation. **Why:** deterministic grading is free + correct; the tag split is why "สวัสดี" in a thread isn't graded (the P1 fix). **Consequence:** feedback modes must send the evaluation payload; `followup`/`free_chat` must not. (`prompt-notes.md`; [`../asking-flow.md`](../asking-flow.md) §2.)

### ADR-013 — Student personality is layered *under* the teacher's `agent_settings`
**Context:** both the teacher (per-lesson) and the student (preference) shape the tutor. **Decision:** inject teacher `agent_settings` first, then the student's personality preset beneath it. **Why:** the teacher owns the pedagogy of *their* lesson; the student only tunes coaching style. **Consequence:** don't let a personality override a teacher's `allow_direct_answers`/`scope`/guidelines.

### ADR-014 — Route-level code splitting to protect low-bandwidth students
**Context:** a single bundle shipped TipTap+Fabric+KaTeX to every visitor (646 kB gzip) — including students who only read. **Decision:** `React.lazy` every route except `Landing`/`NotFound`; pin heavy vendors to `manualChunks`; move editor-only CSS into the editor/viewer chunks. Entry is now ~138 kB gzip. **Why:** the mission audience is explicitly low-connectivity; downloading the teacher's editor to read a lesson is unacceptable. **Consequence:** don't statically import heavy libs into shared/entry code; keep guide images as `public/` URLs, never `import`ed; watch the entry-chunk size on every build. (ROADMAP Tier 2.)

### ADR-015 — Deploy last, in one shot (nothing is pushed until launch)
**Context:** the whole rework happened locally; both repos are many commits ahead of origin. **Decision:** no pushes until the launch tier — one deploy pass (push + env + admin promote + prod smoke) at the end. **Why:** avoid half-deployed states and repeated env churn on a solo project. **Consequence:** "it's not on prod yet" is intentional, not incomplete; production still runs the pre-rework app until Tier 5. (ROADMAP D1 / Tier 5.)

### ADR-016 — Loanwords: Thai term with English in parentheses, never bare English
**Context:** many target students can't read English, but the owner won't strip common technical loanwords. **Decision:** every borrowed term is written `คำไทย (english)` on first mention per reply (e.g. `อิเล็กตรอน (electron)`), on both the tutor and creator AI surfaces. **Why:** accessibility without dumbing down. **Consequence:** verbatim proofread presets are exempt (must not alter wording); the rule lives in `persona.ts` + `creator/prompts.ts` — keep them in sync. (`prompt-notes.md` 2026-07-11.)

### ADR-017 — `app.ts` exports the app; `index.ts` boots it
**Context:** the test suite must exercise the full HTTP surface offline. **Decision:** `app.ts` builds + exports the Express app with no `listen`/`connectDB`; `index.ts` does `dotenv → connectDB → listen`. **Why:** supertest can import the app with an in-memory Mongo and never open a port or hit a real DB. **Consequence:** keep boot side-effects out of `app.ts`; if a host's start command hard-codes `dist/app.js`, fix it to `dist/index.js`.

### ADR-018 — Two-model routing + mandatory transient-error retry
**Context:** quick feedback should be cheap/fast; coaching should be strong; and the owner's ISP drops connections. **Decision:** route `question_feedback`+`quick_check` to a fast model, everything else to the tutor model (`services/tutor/models.ts`); wrap every call in `generateWithRetry`. **Why:** cost/latency balance + resilience to flaky networks. **Consequence:** `generateWithRetry` is load-bearing, not polish — never delete/bypass it (observability hooks its outcomes). Model defaults are owner-tone-gated; changes go through `prompt-notes.md`.

### ADR-019 — Denormalized author names on `Content`
**Context:** lesson lists show author names without wanting a per-row user join. **Decision:** store `author_name`/`collaborator_names[]` copies on `Content`, kept in sync by `contentAuthorSnapshot.service`; a profile rename propagates to all the user's lessons. **Why:** cheap list rendering. **Consequence:** `updateContent` ignores client-sent values for these fields; don't "fix" them with a join — maintain the snapshot service instead.

---

*See also: [`../ROADMAP.md`](../ROADMAP.md) §"Careful-not-to-break" (the live invariant list) and [`../notes.md`](../notes.md) (known tech-debt / decisions still open).*
