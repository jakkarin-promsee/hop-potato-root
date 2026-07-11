# testing.md — the testing strategy

> Updated 2026-07-11. What the test suites guarantee, how they're structured, and how to write a test for each layer. The working agreement (from every roadmap): **every phase ships tests for its own acceptance criteria; you're not done until `npm test` is green in every half you touched.**

The guiding principle: **both suites run fully offline** — no network, no real database, no API key. That's deliberate — a contributor (or CI) can prove correctness with zero secrets, and tests never cost Gemini tokens.

---

## 1. Server — Vitest + Supertest + in-memory Mongo

```bash
cd server
npm test            # full offline suite
npm run test:watch
npm run test:ai     # LIVE Gemini smoke — costs tokens, real key, phase boundaries only
```

**How offline works:** `test/setup.ts` spins up `mongodb-memory-server` (a throwaway Mongo) and sets test env vars; the Gemini client is **mocked**, so `callTutorModel` returns canned output. No `.env`, no network. This is possible because of the **app/entry split** (ADR-017): `test/` imports the Express app from `app.ts` without ever calling `listen` or connecting to a real DB.

**Layout:**

```
server/test/
├── setup.ts              # in-memory Mongo + test env
├── fixtures/lesson.json  # committed lesson fixture (re-export via scripts/export-fixture.ts)
├── helpers/seed.ts       # seed users/lessons for API tests
├── unit/                 # pure logic: prompt builders, parse guards, retry, lessonContext serializer
├── api/                  # endpoint integration via Supertest (auth, content, answers, tutor, memory, creator, rateLimit, profile)
└── ai/                   # live Gemini smoke (*.ai.test.ts) — EXCLUDED from npm test
```

**What each layer tests:**
- **unit** — deterministic functions. E.g. `persona.test.ts` snapshots the system instruction (so tone changes are caught); `parse` tests the `[SUGGESTIONS]` split + strip guards; `lessonContext` tests the serializer per node type.
- **api** — real HTTP through the app with a mocked model + in-memory DB. This is where **guard behavior** lives: 401 anonymous / 403 non-owner / happy path / malformed-input. The tutor + creator tests assert Golden Rule 2 (anonymous works) and the teacher-only guard.
- **ai** — the only tests that hit real Gemini; run manually at phase boundaries to sanity-check tone/latency.

---

## 2. Client — Vitest + happy-dom

```bash
cd client
npm test
npm run lint
```

Config in `vitest.config.ts` (re-declares the `@` alias). **Tests live next to the code** they cover, in `__tests__/`:

```
src/stores/__tests__/           # store logic (profile, bookmark, appearance, tutorMemory)
src/lib/__tests__/              # axios helpers (isProtectedPath, buildForcedLoginUrl), creatorApi, clientEnv
src/components/__tests__/       # route guards, TopNav, ThemeToggle, LanguageToggle, TutorMemoryCard
src/pages/__tests__/            # Profile, Login, Dashboard, Explore, Create, Status, History, Setting
src/components/editor/extensions/__tests__/  # tutorApi, questionEvaluation, MarkdownMessage, FeedbackDiscussionPanel
src/components/editor/ai/__tests__/          # creator copilot surfaces (writingAssist, etc.)
src/__tests__/                  # cross-cutting: seo.test.ts, no-cdn-fonts.test.ts
```

**Notable regression tests worth knowing:** `no-cdn-fonts.test.ts` (fails if a CDN font `@import` sneaks back — performance/ADR-014), `seo.test.ts` (meta/OG tags present), the markdown round-trip test for the writing assistant (headless TipTap), and the guard tests (`ProtectedRoute`/`RequireLogin`/`PublicRoute` behavior).

---

## 3. How to test a new change (by layer)

| You added… | Write this test | Where |
| --- | --- | --- |
| A pure function / prompt builder | unit test asserting input → output; snapshot if it's prompt text | `server/test/unit/` |
| A new endpoint | Supertest: each guard state (anon/non-owner/owner/admin) + happy path + bad input | `server/test/api/` |
| A new tutor mode / input field | a `coerce*` unit test + an api test that the mode routes correctly | both |
| A new Zustand store | a store test: initial state, each action, persistence if any | `client/src/stores/__tests__/` |
| A new page/guard | render test: what anon vs logged-in sees | `client/src/**/__tests__/` |
| An editor node / AI surface | a mapping/round-trip test (attrs ↔ serializer, or markdown insert) | `client/src/components/editor/**/__tests__/` |

**Rules of thumb:**
- If a change spans both halves (a contract change like the structured 401 or `[SUGGESTIONS]`), add/adjust tests on **both** sides.
- Prompt/persona edits **must** update the snapshot tests **and** append to [`../server/prompt-notes.md`](../server/prompt-notes.md).
- Don't write tests that need the network or a real key — mock the boundary (that's what keeps the suite offline).
- Test the **invariant**, not the implementation: assert "anonymous can still use the AI", not "this exact internal call happened."

---

## 4. What's intentionally not tested

No end-to-end browser suite in CI (Playwright is used ad-hoc for the guide screenshots + manual smoke, not automated regression). Live AI output isn't asserted for exact wording (it's non-deterministic — tone is owner-gated by hand). Load/perf isn't benchmarked beyond eyeballing the entry-chunk size on build. These are acceptable for a solo project; the offline unit+integration suites are the safety net.
