# ROADMAP.md — Hot Potato Launch-Readiness Roadmap (v3)

The single plan for all remaining work on Hot Potato. Written 2026-07-10 for implementing agents (Claude, Cursor, …) and humans. Previous roadmaps live in [`archive/`](archive/) — `arc1/` is the 2026-07-09 tutor-only roadmap, `arc2/` is the 2026-07-10 universal roadmap (its Tiers 0–3 **shipped and owner-gated**, see §2) plus its `plan/` execution specs. **Do not execute from archive.**

> Read [`CLAUDE.md`](CLAUDE.md), [`server/CLAUDE.md`](server/CLAUDE.md), and [`client/CLAUDE.md`](client/CLAUDE.md) before touching code. ⚠️ `client/` and `server/` are **separate git repos**; never run git from the `Hot-Potato/` root.

---

## 1. Vision & Golden Rules (non-negotiable, carried forward)

These override any "best practice" instinct. If a change conflicts with a Golden Rule, the Golden Rule wins.

### 🥇 Golden Rule 1 — Never ration tokens for real students
No per-student quotas, no message caps, no "daily limit" UI, no paywall logic. Rate limiting exists **only** to stop bots, with env-tunable thresholds so generous no human ever hits them (`AI_RATELIMIT_PER_10MIN=60`, `AI_RATELIMIT_PER_DAY=1000`).

### 🥇 Golden Rule 2 — Anonymous users keep FULL access
No-login users read all public content **and** use the AI fully. The only logged-out difference: nothing persists (no history, no memory, no saved sessions). **Never add `protect` to an AI or public-content read path.** When a phase adds a logged-in-only feature, the anonymous path degrades gracefully — never to an error or a login wall.

### Product principles (from the owner)
1. **Thai-first.** AI output defaults to Thai, warm and casual. UI copy is bilingual (language toggle shipped).
2. **Critical thinking > correctness.** Admire the idea first, never say "ผิด", give dopamine. No scoring students against each other.
3. **Students are low-tech AI newbies.** The app *navigates* them — teach "how to think with AI", not "follow AI".
4. **Teacher context drives the agent.** Questions, guide answers, and per-lesson `agent_settings` shape every reply.
5. **Phone-first.** Every UI decision is judged at ~390 px width first, desktop second.
6. **Solo, free-tier project.** The owner works alone and hosts on free tiers (Vercel + Render + Atlas). Prefer simple in-app solutions over paid SaaS; no feature may assume a budget.

---

## 2. Current state (verified against code, 2026-07-10)

**The product core is done and owner-approved.** The previous roadmap's Tiers 0–3 all shipped 2026-07-10 (server HEAD `0f748eb`, client HEAD `d633da2`), and the owner **passed every tone/behavior gate** the same day (Tier 0 phone + transcripts, Tier 1 six personality presets + teacher settings, Tier 3.B `gemini-3.5-flash` default). What exists now:

- **Unified tutor:** `POST /api/chat/tutor` (4 modes, SSE streaming, suggestions, per-lesson teacher `agent_settings`, 6 student personalities, StudentMemory digest), persona "น้องมันฝรั่ง", markdown replies, phone-first flat chat threads with suggestion chips.
- **Pages:** all 15 routes wired and real — including Profile (avatar/nickname/bio via `GET/PUT /users/me/profile`), Login UX (redirect-back, Thai banners), Explore bookmarks, Status (sanitized public health, 30 s poll).
- **Hardening:** env-driven CORS, admin role + promote script, auth rate limits, sanitized status/errors, registration lockdown.
- **Extras beyond the old roadmap:** bilingual UI (`language.store` + `useAppI18n`), logo branding.
- **Health (re-verified 2026-07-11, post-Tier-3.5):** server 271/271 tests, client 168/168, `npm run build` green on both halves, lint has **one** known error (`__dirname` in `client/vite.config.js` — fixed in Tier 4).
- **Teacher AI helper (Tier 3.5) shipped 2026-07-11:** `POST /api/creator/assist` (11 actions, teacher-only) + the full editor copilot (question AI, formula AI, writing assistant, draft/import, critic, publish autofill).

**⚠️ Nothing is deployed.** Both repos are ahead of `origin/main` (server 13 commits, client 12) — production still runs the pre-rework app. **This is intentional** (owner decision): keep working locally, then do one deploy pass at the end — that is Tier 5.

### Owner decisions shaping this roadmap (do not re-litigate)

| # | Decision |
| --- | --- |
| D1 | **Deploy last, in one shot.** No pushes until Tier 5; all overhead (push, Render env, admin promote, prod smoke) happens together. |
| D2 | **Password reset / email infra last.** It is isolated from everything else — deferred to Tier 6. |
| D3 | **No PDPA/privacy page for now.** Dropped from scope by the owner. (A delete-account path still rides along with Tier 6 email infra as an optional item.) |
| D4 | **`/settings` keeps only what works.** Continue building only *Appearance* (font size etc.); Help & Support becomes a popup with the owner's Facebook contact; dead rows get cut. |
| D5 | **`/guide` untouched until last** (Tier 6). |
| D6 | **CI is nice-to-have** — owner works alone; build it cheap (Tier 4), don't gold-plate it. |
| D7 | **Landing showcase cards stay fictional** (carried from old roadmap). *Superseded 2026-07-11: the cards were removed entirely when Landing absorbed the `/guide` hub — see ROADMAP-guide.md §12.* |
| D8 | **Teacher AI helper (Tier 3.5) ships before launch** (2026-07-11). The Create page gets an AI copilot for low-tech teachers. Formula AI rides the existing KaTeX `latex` attr on `FormulaBlock`; every AI edit is preview → accept, never auto-apply; creator routes are teacher-only (`protect`). |

---

## 3. How this roadmap is organized

- **Tiers = priority bands** in the owner's stated order: SEO → observability → performance → settings/profile → **teacher AI helper** → CI → **launch** → deferred backlog. Tiers 0–4 are independent of each other and may be built in any order if it's convenient, but the listed order is the owner's priority. **Tier 5 (launch) requires 0–3.5 done** (4 optional). Tier 6 comes after launch.
- **Phases = shippable units** with goal, work items, and acceptance criteria.
- **Working agreement (inherited):** every phase ships tests for its own acceptance criteria; a phase is not done until `npm test` is green in every half it touched, and the relevant `CLAUDE.md` is updated.
- **The owner remains the tone referee** for anything touching prompts (none planned in this roadmap — but if a phase drifts into `persona.ts`, log it in [`server/prompt-notes.md`](server/prompt-notes.md)).

---

## 4. Tier 0 — SEO & Link Previews 🔗 ✅ (Phase 0.A shipped 2026-07-10)

**Theme: when a teacher shares a lesson link in LINE or Facebook, it must look real.** Today `client/index.html` has zero meta tags: no description, no Open Graph, `lang="en"`, no robots.txt — shared links render as a bare URL. This is the cheapest reach win in the whole plan.

### Phase 0.A — Static meta + OG tags ✅

- `index.html`: `lang="th"`; `<meta name="description">` (Thai, one sentence on what Hot Potato is); Open Graph tags (`og:title`, `og:description`, `og:type=website`, `og:image` — use `src/assets/logo.png` or a purpose-made 1200×630 card); `twitter:card` = `summary_large_image` (or `summary` if only the square logo exists).
- Add `client/public/robots.txt` (allow all) and a proper favicon declaration (today it points at `./src/assets/logo.png` with `type="image/svg+xml"` — wrong MIME, works by accident).
- While in the file: normalize `<script src="/src/main.jsx">` → `/src/main.tsx` (the real file; build currently resolves it by luck).
- **Known limitation (document, don't solve):** this is a client-rendered SPA — social crawlers don't run JS, so **per-lesson** OG tags (lesson title/description per `/view/:id` link) need SSR/prerendering or an edge function. Out of scope; site-wide tags only. Leave a note in `client/CLAUDE.md` so nobody wonders why lesson links share generic cards.

**Acceptance:** sharing the site URL into a LINE/Facebook/Discord preview debugger shows title + description + image; `npm run build` + `npm test` green; lesson pages still load identically.
**Size:** S · **Repos:** client
**Shipped:** client commit `e3c53ec` — 74 tests green (66 + 8 SEO), build green, lint still 1 expected error (`__dirname`). Absolute `og:image` URL deferred to Tier 5 launch smoke.

---

## 5. Tier 1 — Observability: error monitoring + Status page upgrade 📡

> ✅ Shipped 2026-07-10 — 1.A (server) + 1.B (client).

**Theme: when the app misbehaves in production, anyone — including the owner on their phone — can see what's wrong on a public page.** The `/status` page already exists and is good (Tier 3 sanitized it; polls `GET /api/status/all` every 30 s with server/database/env cards). This tier extends it into the "is everything actually working?" page the owner asked for. No paid SaaS (principle 6) — monitoring lives in the app itself.

### Phase 1.A — Server: recent-error tracking + AI health

- **Error ring buffer:** hook `error.middleware.ts` (and the tutor controller's error paths) to record the last N (≈50) server errors in memory: timestamp, method+route, status code, error name. **Never store request bodies or student messages.** In-memory is fine (free-tier Render restarts wipe it — acceptable; note it on the page as "since last restart").
- **AI health signal:** track the last tutor-call outcome (`ok`/`failed` + timestamp) in a module-level status — updated wherever `generateWithRetry` succeeds/fails. Do **not** ping Gemini from the status endpoint (costs tokens, Golden Rule 1 protects students, not health checks 😄).
- Extend `GET /api/status/all` (public, sanitized): add `ai: { status, last_success, last_failure }` and `errors: { count_since_boot, last_error_at, recent: [{ time, route, status }] }` — route paths and status codes only, **no messages/stacks** publicly. Full error names/messages go on the admin-only `GET /api/status/` if wanted.
- Tests: unit-test the ring buffer (cap, shape, sanitization) + supertest the extended `/status/all` payload in both auth states.

**Acceptance:** trigger a 500 locally → it appears in `/api/status/all` within one poll with route+code but no message; tutor call flips `ai.status`; suite green and still offline.
**Size:** S–M · **Repos:** server

### Phase 1.B — Client: Status page v2

- Add two cards to `/status`: **AI Tutor** (ok/degraded + "last success X min ago") and **Recent errors** (count + compact list of time/route/code; empty-state "ไม่มีข้อผิดพลาด 🎉").
- Bilingual copy via `useAppI18n` (page is currently English-only).
- Keep the 30 s poll and existing skeleton/error states; phone pass at ~390 px.

**Acceptance:** page renders both new cards from the live payload in both languages; no horizontal scroll at 390 px; client tests cover the new cards' render logic.
**Size:** S · **Repos:** client

---

## 6. Tier 2 — Performance: split the 2 MB bundle ⚡

**Theme: students on bad connections should not download the teacher's editor.** Today `vite build` emits a single 2,095 kB JS chunk (646 kB gzip) — every visitor downloads TipTap + Fabric + KaTeX + lowlight + the full editor even to view the landing page. The target audience is explicitly low-connectivity (principle 6 + the app's mission).

### Phase 2.A — Route-level code splitting ✅

- `React.lazy` + `Suspense` the route components in `App.tsx` — at minimum the heavy ones: `TipTapCanvas` (editor), `TiptapView` (viewer), `Cloudinaryupload`. A lightweight shared loading fallback (match existing loading states).
- `build.rollupOptions.output.manualChunks` to pin obvious vendor groups (e.g. `fabric`, `katex`, TipTap packages) so caching works across deploys.
- Watch the trap: `TiptapViewer` (student) and `TipTapEditor` (teacher) share `editorExtensions.ts` — splitting routes still leaves the viewer pulling TipTap+KaTeX+Fabric (needed to render lessons). That's fine; the win is Landing/Explore/Login/Settings not pulling any of it.
- **Acceptance:** `npm run build` shows the entry chunk for `/` + `/explore` at **≤ half of today's 646 kB gzip**; the chunk-size warning for the entry is gone; every route still renders (smoke the editor + viewer + a full tutor conversation locally); tests green.
- **Size:** M · **Repos:** client
- **Shipped:** client commit — entry JS **137 kB gzip** (was 648 kB); `manualChunks` for `tiptap`/`fabric`/`katex`; `indexTiptap.css` moved to editor/viewer lazy chunks; dead deps removed (`lowlight`, `cloudinary`, etc.); 86 tests green.

### Phase 2.B — Self-host fonts ✅

- `index.html` pulls Inter + DM Sans from Google Fonts CDN — an external dependency that stalls first paint on bad connections (and neither family has Thai glyphs; Thai text falls back to system fonts today).
- Move to `@fontsource/*` packages imported in `index.css` so fonts ship with the bundle; optionally add a Thai-capable face (e.g. Noto Sans Thai) for UI text — **owner eyeballs the look before it lands** (it changes the app's face).
- **Acceptance:** no requests to `fonts.googleapis.com`/`gstatic.com` in the network tab; visual diff approved by owner; build green.
- **Size:** S · **Repos:** client
- **Shipped:** client commit — Lora/JetBrains Mono/Inter self-hosted via `@fontsource`; CDN links removed from `index.html` + `index.css`; regression test `no-cdn-fonts.test.ts`; entry CSS 35 kB gzip (was 42 kB with CDN `@import`).

---

## 7. Tier 3 — Settings & Profile completeness 🧩

> ✅ Shipped 2026-07-11 — 3.A + 3.B (client commits `acca9ff`, `225e2ae`).

**Theme: no dead buttons anywhere a student can tap.** (Owner decisions D3/D4 apply.)

### Phase 3.A — `/settings` cleanup + Appearance

Current real controls: theme, language, AI personality picker, log out. Dead rows (render but do nothing): *Notifications*, a duplicate *Appearance* row, *Privacy & Security*, *Help & Support*, *Delete account*.

- **Cut:** *Notifications* (no notification system exists), the duplicate *Appearance* row in "Preferences", and *Delete account* (returns with Tier 6 email/account infra — don't show a dead destructive button until then).
- **Wire:** *Privacy & Security* → navigate to the existing `/change-password` page (one line; rename the row "รหัสผ่าน / Password" so it doesn't promise 2FA that doesn't exist).
- **Help & Support → popup:** a small dialog (shadcn `Dialog`/`AlertDialog` already in the repo) with a short Thai blurb ("เว็บนี้ทำฟรีโดยคนคนเดียว 😄"), the owner's **Facebook contact link** (owner supplies the URL — use a placeholder constant until then), and maybe the `/status` link for "is it down?".
- **Appearance, for real:** add a **font-size control** (e.g. เล็ก / ปกติ / ใหญ่ / ใหญ่มาก) — a persisted store (extend `theme.store` or a new `appearance.store`) that sets `document.documentElement.style.fontSize` (Tailwind sizes are rem-based, so the whole app scales — including lesson text and chat, which is the point for young/low-vision students). Note: the lesson **viewer already has its own zoom** (CSS `zoom` on the card container) — don't duplicate that; font size is the app-wide knob.
- Phone pass: the largest font size must not break the 390 px layout (chat input row, TopNav, question cards).

**Acceptance:** every visible row in `/settings` does something; anonymous users see no broken/dead rows (Golden Rule 2 — settings page stays public); font size persists across reload; largest size usable at 390 px; tests cover the store + cut/wired rows.
**Size:** M · **Repos:** client
**Shipped:** client commit — dead rows cut, Help & Support popup, app-wide font size (`appearance.store`).

### Phase 3.B — `/profile` completion

The page is already real (Tier 2.A): avatar upload, display name, nickname, bio, change-password link — and it **is** linked from TopNav. What makes it *complete* for a learning app:

- **"ความจำของติวเตอร์" card:** show what น้องมันฝรั่ง remembers about the student — `GET /api/chat/memory` (interests / strengths / growth_areas / preferences / recent_topics as chips) + a **"ลบความจำ"** button → `DELETE /api/chat/memory` with a confirm dialog. This server feature has existed since Phase 3 and **has zero UI today** — students deserve to see and control it. Empty state for students with no memory yet ("คุยกับติวเตอร์ไปก่อน เดี๋ยวเขาจะจำเธอได้เอง 🥔").
- **Personality shortcut:** show the current tutor personality with a link to `/settings` (don't duplicate the picker).
- **Light learning stats (optional, keep tiny):** lessons visited count from the existing history store. **No scores, no comparisons** (product principle 2).
- Remove the duplicated *Theme* toggle row on Profile (it lives in Settings; one source of truth).

**Acceptance:** logged-in students can view + wipe their tutor memory from Profile with confirm; anonymous users never see the memory card (route is already `RequireLogin`); tests cover the memory card states (loaded / empty / delete flow).
**Size:** S–M · **Repos:** client (server endpoints already exist)
**Shipped:** client commit — `TutorMemoryCard`, personality shortcut, Theme row removed from Profile.

---

## 8. Tier 3.5 — Teacher AI Helper 🧑‍🏫✨ (owner decision D8)

> ✅ **Shipped 2026-07-11** — all six phases (server `2b4ab13`, client `b45848b`→`a0d1d1b`), **plus the 3.5.G UX rework the same day** (owner feedback pass: sidebar AI hub — see the phase below). Server 271/271 tests, client 176/176, builds green. **Live manual pass done the same day** (Playwright + real Gemini on local dev: all 8 copilot surfaces + security spot-checks — details in `plan/tier3.5.md` §7). Remaining: owner reads the captured critic transcript for the tone gate, and may delete the test account/lesson noted in the plan file.

**Theme: teachers are exactly as low-tech as the students — the Create page must author lessons *with* AI, not assume the teacher already can.** The owner's field observation (2026-07-11): Thai teachers in the target group got their first ChatGPT training a week ago and still struggle with Canva; LaTeX is a non-starter. The AI moves into the `/canvas/:id` editor as a copilot for content creation — better teacher content directly means better tutor context for students (`lessonContext` + `guideAnswer` feed น้องมันฝรั่ง).

**Full execution spec (Cursor-executable, self-contained): [`plan/tier3.5.md`](plan/tier3.5.md).** The roadmap entries below are summaries — implement from the plan file.

Non-negotiables for the whole tier:

- **Teacher-only surface.** Every creator AI route is `protect` + owner/collaborator check on `contentId`. Golden Rule 2 governs the student/read side and is untouched; Golden Rule 1's spirit applies to teachers too — generous bot-only rate limits, no teacher quotas.
- **Preview → accept, never auto-apply.** AI output enters the document only through a normal editor transaction after the teacher accepts (the autosave + optimistic-concurrency machinery is untouched).
- **AI never emits raw TipTap doc JSON.** Prose comes back as markdown; blocks come back as typed JSON matching the node attrs, validated server-side before the client ever sees them.
- **No new env vars** (reuses `AI_FAST_MODEL` / `AI_TUTOR_MODEL` / `AI_OUTPUT_LANGUAGE`); output Thai-first, UI copy bilingual via `useAppI18n`.

### Phase 3.5.A — Server foundation: `POST /api/creator/assist`

One unified creator endpoint mirroring the tutor pattern: guards `[protect, ...createChatRateLimiter()]` + ownership check; 11 actions (`formula_latex`, `proofread`, `generate_questions`, `guide_answer`, `distractors`, `outline`, `draft_section`, `import_structure`, `critic`, `lesson_meta`, `agent_settings_suggest`); reuses `callTutorModel`, `lessonContext.service`, `getFastModel`/`getTutorModel`, `generateWithRetry`, and the observability hooks. Strict server-side validation of generated question JSON (drop malformed items); tolerant JSON parsing with one re-ask retry.

**Acceptance:** supertest suite covers 401 anonymous / 403 non-owner / happy path per action with mocked Gemini / malformed-JSON retry / validation drops; suite green offline.
**Size:** M · **Repos:** server
**Shipped:** server commit `2b4ab13` — 54 new tests (271 total). `callTutorModel` gained optional `responseMimeType` (Gemini JSON mode; config byte-identical when omitted, regression-tested).

### Phase 3.5.B — Question AI

"สร้างคำถามด้วย AI" dialog in the editor sidebar (scope: ทั้งบท or a section; pick from the 4 gradeable question types; count; difficulty) → preview cards → insert real question nodes. Plus ✨ guide-answer fill on existing `QuestionWrite` blocks (choice blocks derive their guide answer from the correct choices automatically) and distractor suggestions in the choice editor — guide answers directly upgrade the student tutor's context.

**Acceptance:** generated blocks are attr-identical to hand-made ones (incl. `[Q-n]` blank templates); nothing inserts without an explicit teacher accept; tests cover schema mapping + insert flow.
**Size:** M · **Repos:** client
**Shipped:** client commit `b45848b` — `lib/creatorApi.ts` bridge, `AiQuestionDialog`, guide-answer draft, distractor chips; 20 new tests.

### Phase 3.5.C — Formula AI

"ให้ AI เขียนสูตร" panel inside `FormulaCanvas`: teacher types the formula the human way (`s = ut + 1/2at^2`, required) + คำอธิบาย + ใช้ทำอะไร (optional context) → `formula_latex` → fills the **existing `latex` attr through the same `persistLatex` path** as manual edits. Teacher can hand-edit the LaTeX afterwards; KaTeX render-error fallback already exists.

**Acceptance:** a teacher who has never seen LaTeX produces a rendered formula in one round-trip; manual LaTeX editing still works; tests cover the panel + attr write.
**Size:** S · **Repos:** client
**Shipped:** client commit `0c5f7ac` — `AiFormulaPanel` writes through `persistLatex`; render-failure retry hint; 5 new tests.

### Phase 3.5.D — Writing assistant (auto-correct & format)

Selection-based AI actions (bubble menu): แก้คำผิด · จัดย่อหน้า/ใส่หัวข้อ · เกลาให้อ่านง่าย · ย่อ · ขยายความ · ปรับระดับภาษาตามชั้นเรียน (the Diffit-style reading-level move). Before/after preview dialog → replace selection.

**Acceptance:** each action round-trips on real Thai lesson text; reject leaves the doc byte-identical; tests cover the action → preview → apply pipeline.
**Size:** M · **Repos:** client
**Shipped:** client commit `266e595` — header dropdown (BubbleMenu rejected: CSS `zoom` breaks floating-ui anchoring; visible button beats hidden menu for low-tech teachers); real headless-TipTap round-trip test of the markdown insert path; 10 new tests.

### Phase 3.5.E — Draft & import

Empty-lesson CTA + toolbar entry: (1) outline from หัวข้อ + ระดับชั้น + จุดประสงค์ → inserted headings; (2) per-section "เขียนเนื้อหาส่วนนี้" fill; (3) **paste-import** — teacher pastes existing material (old sheets, Word text, ChatGPT output) → structured lesson draft + suggested questions. Import is the highest-leverage feature for teachers who already have content.

**Acceptance:** a blank lesson reaches a structured multi-section draft with questions in under ~3 round-trips, every insertion previewed first.
**Size:** M · **Repos:** client
**Shipped:** client commit `d09ddb1` — `AiDraftDialog` (3 tabs) + empty-doc CTA + header entry (`AiDraftLauncher`); 12 new tests.

### Phase 3.5.F — AI critic + publish autofill

"ตรวจบทเรียน" panel: typed report (accuracy flags, completeness, readability, age-fit, question coverage) — informational only, never blocks publish. In `PublishSettingsModal`: ✨ autofill for title/description/topics (`lesson_meta`) and AI-suggested `agent_settings` (`persona_note`, `custom_guidelines`, scope + direct-answers recommendation) — editable before save. "AI configures the AI tutor" is the signature move of this tier.

**Acceptance:** critic renders a useful report on the test lesson; autofill fields land in the modal editable (not saved until the teacher saves); tests cover report rendering + autofill mapping.
**Size:** M · **Repos:** client
**Shipped:** client commit `a0d1d1b` — `AiCriticButton` (publish never gated) + modal autofill with overwrite-confirm; 8 new tests. Note: autofill writes canvas.store fields, so the editor's existing autosave interval applies — same semantics as typing in the modal by hand (see plan §7 deviations).

### Phase 3.5.G — AI hub UX rework (owner feedback, 2026-07-11)

The owner's hands-on pass on the shipped copilot: the tools worked but were scattered across header dropdowns and too deep for low-tech teachers. This phase makes the **left sidebar the single AI toolbox** — a 5th "AI" category (leads the rail, primary-tinted) whose panel lists every tool as a card with a plain-language description, grouped by the authoring workflow (1 เริ่มบทเรียน → 2 เขียนและเกลา → 3 คำถาม → 4 ก่อนเผยแพร่), modeled on the Question panel's card style:

- **ร่างโครง / เติมเนื้อหา / วางเนื้อหาเดิม** cards open `AiDraftDialog` directly on the right tab. The outline now inserts **at the teacher's caret** (`caretInsertPoint` in `draftHelpers.ts` — empty doc → top; caret on an empty line → replaces it; otherwise → right below the caret's block), not at the doc top.
- **เติมเนื้อหา** gains an optional "อยากได้เนื้อหาแบบไหน" detail box (wired to the server's existing, previously unused `styleHint` field) and an import-style **suggested-questions step**: `generate_questions` with `scope: "selection"` on the freshly drafted section text → preview cards → accept per question. **No server changes.**
- **ปรับข้อความ** is now both a header button and a hub card. The header button is primary-tinted, lights up solid when text is selected, and stays clickable with no selection — opening a 1-2-3 select-first how-to instead of being a dead disabled button. The hub card shows a live "เลือกอยู่: …" selection snippet with the six actions. Same `WritingPreviewDialog` (before/after → ใช้เลย) behind both.
- **ตรวจบทเรียน** stays next to Publish and gains a hub card — the modal is now a shared `AiCriticDialog` (per-trigger report cache; the error state gained a "ลองใหม่" button).

**Acceptance:** all six tools reachable from the sidebar hub with descriptions; outline lands at the caret; fill sends `styleHint` and offers question preview cards; the writing assistant is discoverable without a prior selection; preview → accept unchanged everywhere; client tests green.
**Size:** M · **Repos:** client
**Shipped:** 2026-07-11 — `AiToolsPanel` hub + `caretInsertPoint` + fill detail/questions + prominent writing assistant; 176 client tests green (8 new), build green.

**Order:** 3.5.A first (everything depends on it), then B → C (highest student-facing ROI), then D → E → F in any order. G is the post-owner-feedback UX pass on top of B–F.

---

## 9. Tier 4 — CI (GitHub Actions) 🤖 — optional, cheap

**Theme: the 269 tests should run themselves when code is pushed.** (Owner decision D6: keep it minimal.)

- One workflow per repo (`.github/workflows/ci.yml`): Node 20, `npm ci`, then — client: `lint` + `test` + `build`; server: `test` + `build`. No secrets needed (server tests are offline with in-memory Mongo; `mongodb-memory-server` downloads its binary on the runner — cache `~/.cache/mongodb-binaries` to keep runs fast).
- **Fix the one lint error first** (`__dirname` no-undef in `client/vite.config.js` — give config files node globals in the ESLint config, or switch to `fileURLToPath(import.meta.url)`), so the lint gate is green from day one.
- Trigger on `push` to `main` + `pull_request`. No deploy steps — Vercel/Render already auto-deploy from GitHub pushes (which begin in Tier 5).

**Acceptance:** first push after Tier 5 shows green checks on both repos; a deliberately broken test on a branch fails the workflow.
**Size:** S · **Repos:** both (workflow files only)

---

## 10. Tier 5 — Launch day: one deploy pass 🚀

**Theme: all the overhead at once** (owner decision D1). Prerequisite: Tiers 0–3.5 merged locally (Tier 4 ideally too, so the push itself gets checked).

Ordered checklist:

1. **Push both repos** (`cd client && git push`, `cd server && git push`) — server is 13+ commits ahead, client 12+. Vercel + Render auto-deploy.
2. **Render env:** set `CORS_ORIGINS` (Vercel prod URL + `http://localhost:5173`); set `GOOGLE_CLIENT_ID` (Google sign-in, 2026-07-11); verify `GEMINI_API_KEY`, `JWT_*`, `MONGO_URI`, Cloudinary vars still present; confirm start command is `node dist/index.js`.
3. **Vercel env:** confirm `VITE_API_URL` points at the Render URL (no trailing surprises — the client appends `/api`… check `axios.ts` convention), plus Cloudinary vars and `VITE_GOOGLE_CLIENT_ID`. **Google OAuth:** in Google Cloud Console, add the Vercel prod URL to the OAuth client's *Authorized JavaScript origins* (currently only `http://localhost:5173`) — the Google button silently fails on unregistered origins.
4. **Admin bootstrap:** `npm run promote -- <owner-email>` against the production DB.
5. **Production smoke, both auth states** (Golden Rule 2 is an executable requirement): open the test lesson `/view/69e39d0b60d467bd515a4945` logged-in **and** incognito — full tutor conversation with streaming, question card feedback, personality switch, `/status` page shows all-green including the new AI/errors cards. Also smoke **Google sign-in** on the prod domain (new + linked account).
6. **Post-launch maintenance sweep:** rebuild the knowledge graph (`/graphify` `--update` — it currently lags the Tier 1–3 file changes), and update `CLAUDE.md`/`AGENT.md` current-state lines to "launched".
7. **Real Facebook contact URL:** replace the placeholder in `client/src/lib/contact.ts` (`OWNER_FACEBOOK_URL`) with the owner's page and re-run the settings help-popup smoke.

**Acceptance:** a stranger's phone, no login, can read a lesson and chat with น้องมันฝรั่ง on the production URL; CORS locked; owner account is admin.
**Size:** S (checklist, not code) · **Repos:** both + dashboards

---

## 11. Tier 6 — Deferred backlog (after launch)

Owner-deferred items, in no particular order — each independent:

### 6.A — Email infra + password reset (owner decision D2)
The only truly *missing* account feature: students who forget passwords are locked out forever (no email system exists at all). When its turn comes: pick a free transactional provider (e.g. Resend/Brevo free tier — principle 6), add `POST /auth/forgot-password` (token model, 15-min expiry, silent-success to avoid account enumeration) + `POST /auth/reset-password` + client pages. Optional riders once email exists: email verification on register, and the **delete-account** flow (restores the Settings row cut in 3.A; cascade: UserData, UserContent, ChatSession, StudentMemory, LearningHistory, images). **Size:** L · **Repos:** both

### 6.B — `/guide` refresh (owner decision D5)
Content pass on the Guide page for the post-rework reality (streaming tutor, personalities, teacher agent settings). Pure copy/content work — owner probably wants a hand in it. **Size:** S · **Repos:** client
> **Superseded 2026-07-11:** expanded into its own track — [`ROADMAP-guide.md`](ROADMAP-guide.md) (hub + learning-showcase + creating-showcase, Tiers G0–G6) with the execution spec in [`plan/guide.md`](plan/guide.md). Execute from there, not from this stub.

### 6.C — Dopamine layer ✨ (carried from old roadmap Tier 4 — still design-gated)
Celebration moments on completed questions, lesson progress feel ("คิดมาแล้ว 3 ข้อ 🔥"), teacher-visible engagement signals. **Needs owner design input before any implementation. No leaderboards, no student-vs-student scores — ever.** **Size:** ? · **Repos:** client (+server for signals)

### 6.D — Per-lesson link previews (found in Tier 0, parked)
Real OG tags per `/view/:id` need prerendering/SSR/edge function on Vercel. Revisit if teachers actually share lesson links widely. **Size:** M

---

## 12. Careful-not-to-break list (carried forward, updated)

| Thing | Where | Why it's fragile |
| --- | --- | --- |
| Anonymous full access | `optionalAuth` on content + chat routes, client guards | **Golden Rule 2.** Adding `protect` to an AI/public-read path is a regression. `/settings` and `/status` are public pages — keep them working logged-out. |
| Structured 401 contract | `auth.middleware.ts` ↔ `client/src/lib/axios.ts` | Client auto-logout parses the exact `{ forceRelogin, clearToken }` shape. |
| TipTap node = source of truth | `client/src/components/editor/*` | Canvas/question state lives in node attrs; React state won't persist. |
| Autosave + optimistic concurrency | `PUT /content/:id`, `clientUpdatedAt` → 409 | Editor lazy-loading (2.A) must not break the autosave cycle. |
| Answers map shape | `UserContent.answers` (Map<blockId, any>) | Question views + threads store state here; shape changes break saved student work. |
| Transient-error retry | `generateWithRetry` (`services/tutor/retry.ts`) | Exists because the owner's ISP drops connections. 1.A hooks its outcomes — never delete or bypass it. |
| Error handler order | `server/src/app.ts` | `app.use(errorHandler)` must stay last — 1.A's ring buffer hooks in here. |
| `[SUGGESTIONS]` output contract | `persona.ts` ↔ `parse.ts` ↔ client chips | Format changes need both sides + tests. |
| SSE streaming path | `tutor.controller.ts` ↔ `callTutorStream` | Route-splitting (2.A) must keep the fetch-based SSE working; it deliberately bypasses axios. |
| Viewer zoom via CSS `zoom` | `TiptapViewer` card container | Font-size control (3.A) is a separate mechanism — don't merge them; `transform: scale` reintroduces the double-scrollbar bug. |
| Creator AI = teacher-only + preview→accept | `creator.routes.ts`, editor AI panels | `protect` + owner/collaborator check on every action; AI output enters the doc only as a normal editor transaction after the teacher accepts (autosave/409 untouched). Never `optionalAuth` here, never auto-apply. |
| `latex` attr = formula source of truth | `FormulaBlock` (`FormulaCanvas.tsx`) | Formula AI (3.5.C) writes through the same `persistLatex` path as manual edits; the legacy `formula` tree is fallback-only for old lessons — don't regenerate it from LaTeX. |

---

## 13. Environment variables

No new env vars through Tier 5 — **including Tier 3.5**, which reuses `AI_FAST_MODEL` / `AI_TUTOR_MODEL` / `AI_OUTPUT_LANGUAGE` and the existing rate-limit knobs. *(One exception, owner-requested 2026-07-11: Google sign-in added `GOOGLE_CLIENT_ID` (server) + `VITE_GOOGLE_CLIENT_ID` (client) — both set locally, both needed on Render/Vercel at Tier 5.)* Tier 6.A will add email-provider keys (e.g. `RESEND_API_KEY`, reset-token TTL) — document them in `server/CLAUDE.md` when they land. Full current table: [`server/CLAUDE.md`](server/CLAUDE.md).

---

## 14. Verification playbook

1. **Automated first:** `cd server && npm test` and `cd client && npm test` green before any manual check. `npm run test:ai` (live Gemini, costs tokens) only if a phase touched prompts/models — none planned here.
2. Run both halves: server `:5000`, client `:5173`. Test lesson: `/view/69e39d0b60d467bd515a4945`.
3. Always test **both auth states** — logged in and incognito (Golden Rule 2 is an executable requirement).
4. Phone check: every UI change gets a ~390 px pass (largest font size too, after 3.A).
5. After changing any client↔server contract (Tier 1 touches `/status/all`), grep the other half for the old shape before calling it done.
6. Bundle changes (Tier 2): compare `npm run build` output sizes before/after in the phase notes.
7. Creator AI (Tier 3.5): manual pass runs **logged in as the lesson owner** on `/canvas/:id`; also verify anonymous/non-owner get 401/403 from `/api/creator/assist`, and spot-check `/view/:id` incognito still works (student side must be unaffected).

---

## 15. Sizing & suggested order

| Phase | What | Size | Repos | Depends on |
| --- | --- | --- | --- | --- |
| **0.A** | Static meta + OG tags + robots.txt | S | client | — | ✅ shipped 2026-07-10 |
| **1.A** | Error ring buffer + AI health (server) | S–M | server | — | ✅ shipped 2026-07-10 |
| **1.B** | Status page v2 (client) | S | client | 1.A | ✅ shipped 2026-07-10 |
| **2.A** | Route-level code splitting | M | client | — | ✅ shipped 2026-07-11 |
| 2.B | Self-host fonts | S | client | — | ✅ shipped 2026-07-11 |
| **3.A** | Settings cleanup + font-size control | M | client | — | ✅ shipped 2026-07-11 |
| **3.B** | Profile completion (tutor-memory UI) | S–M | client | — | ✅ shipped 2026-07-11 |
| **3.5.A** | Creator-assist endpoint (server foundation) | M | server | — | ✅ shipped 2026-07-11 |
| **3.5.B** | Question AI (generate + guide answers + distractors) | M | client | 3.5.A | ✅ shipped 2026-07-11 |
| **3.5.C** | Formula AI (plain text → LaTeX in FormulaBlock) | S | client | 3.5.A | ✅ shipped 2026-07-11 |
| **3.5.D** | Writing assistant (proofread / format / reading level) | M | client | 3.5.A | ✅ shipped 2026-07-11 |
| **3.5.E** | Outline / draft section / paste-import | M | client | 3.5.A | ✅ shipped 2026-07-11 |
| **3.5.F** | AI critic + publish autofill (meta + agent_settings) | M | client | 3.5.A | ✅ shipped 2026-07-11 |
| **3.5.G** | AI hub UX rework (sidebar toolbox, caret insert, fill detail+questions) | M | client | 3.5.B–F | ✅ shipped 2026-07-11 |
| 4 | CI workflows + lint fix | S | both | — (best before 5) |
| **5** | Launch: push + env + promote + prod smoke | S | both | 0–3.5 (4 recommended) |
| 6.A | Email infra + password reset (+ delete account) | L | both | after 5 |
| 6.B | Guide refresh | S | client | after 5 |
| 6.C | Dopamine layer (design-gated) | ? | client | owner design input |
| 6.D | Per-lesson OG (SSR/prerender) | M | client | if needed |

**Recommended path:** ~~0.A → 1.A → 1.B → 2.A → 2.B → 3.A → 3.B → 3.5.A → 3.5.B → 3.5.C → 3.5.D → 3.5.E → 3.5.F~~ (all shipped) → (4 as a breather) → **5 launch** → 6.* as the owner feels like it.
