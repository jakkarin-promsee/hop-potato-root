# ROADMAP-detailed.md — Implementation Spec for Phases -1 → 3

Companion to [`ROADMAP.md`](ROADMAP.md) (read it first for vision, Golden Rules, and Phases 4–6). This document is the **precise, step-by-step implementation spec** for Phases -1, 0, 1, 2, and 3 — written so an implementing agent (Cursor, etc.) can execute each phase without inventing architecture. Where this doc says "✅ verified", the fact was read directly from the current source code. Where it says "⚠️ characterize", the implementing agent must run the current code first, observe the real behavior, and lock that into the test.

---

## 0. Agent contract — read before writing any code

1. **Two separate git repos.** All git commands run inside `server/` or `client/`, never from `Hot-Potato/` root. Commit style follows existing history: `feat: ...`, `fix: ...`, `test: ...`, `docs: ...`. Commit per logical step, per repo.
2. **Golden Rule 1:** no per-student token/message quotas, ever. Rate limits are bot guards with generous defaults, env-configurable.
3. **Golden Rule 2:** anonymous users get full content-reading and AI access. Any change that makes an AI or public-content path require login is a bug, even if it "improves security".
4. **Definition of done for every phase:** `npm test` green in every half you touched, new tests written for the phase's own features, docs updated where this spec says so. Do not mark a phase done with skipped/failing tests.
5. **Stop-and-report conditions:** if the code you find contradicts this spec (file missing, shape different, test impossible as written), stop and report the discrepancy instead of improvising around it.
6. **Env & secrets:** real secrets live in `server/.env` / `client/.env` (git-ignored). Never commit them, never print their values. Tests must not require real secrets except the gated `test:ai` suite.
7. The test lesson in the real DB is `contents` ObjectId **`69e39d0b60d467bd515a4945`** (a complete, teacher-authored lesson).

Current server facts you will rely on (all ✅ verified):

- `server/src/app.ts` runs `dotenv.config()`, `connectDB()` (exits process on failure), builds the Express app, mounts all routes, ends with `app.use(errorHandler)`, then **calls `app.listen` directly and exports nothing**.
- `npm start` = `node dist/app.js`; nodemon exec = `node --use-system-ca -r ts-node/register src/app.ts`; `"type": "commonjs"`; TypeScript 6; Express 5; Mongoose 9.
- AI SDK today: `@google/generative-ai@^0.24.1` (deprecated upstream). All AI code is in `server/src/controllers/chat.controller.ts`; nothing in it is exported except the three request handlers.
- `chat.controller.ts` module-level constants: `MODEL_NAME = "gemini-2.5-flash-lite"`; `WRITE_EVAL_MODEL_NAME` and `OUTPUT_LANGUAGE` are **computed at module load** from env — set env before importing the module in tests.
- Auth 401 body shape (from `auth.middleware.ts`): `{ message, code, forceRelogin: true, clearToken: true }`, `code ∈ TOKEN_MISSING | TOKEN_EXPIRED | TOKEN_INVALID | USER_NOT_FOUND`.

---

# PHASE -1 — Test harness ("give the AI eyes")

Everything in this phase happens in `server/` unless marked **[client]**. Target file tree when done:

```
server/
├── vitest.config.ts
├── vitest.ai.config.ts
├── scripts/export-fixture.ts
├── src/
│   ├── app.ts          # refactored: builds + exports app, NO listen, NO connectDB
│   ├── index.ts        # NEW: dotenv → connectDB → app.listen
│   └── controllers/chat.controller.ts   # prompt builders + retry helpers now exported
└── test/
    ├── setup.ts
    ├── fixtures/lesson.json
    ├── helpers/seed.ts
    ├── unit/prompts.test.ts
    ├── unit/retry.test.ts
    ├── api/auth.test.ts
    ├── api/content.test.ts
    ├── api/answers.test.ts
    ├── api/chat.test.ts
    └── ai/live.ai.test.ts
```

## Step -1.0 — Make the app importable (prerequisite refactor)

Supertest needs to `import app` without starting a listener or connecting to the real DB.

1. Edit `src/app.ts`: **remove** the `connectDB()` call (lines with `connectDB().catch(...)`) and the final `app.listen(...)` block. Keep `dotenv.config()` (harmless, and module-load constants elsewhere depend on env). Add `export default app;` at the end. Keep the route mount order and `app.use(errorHandler)` last, exactly as-is.
2. Create `src/index.ts`:

```ts
import dotenv from "dotenv";
dotenv.config();

import connectDB from "./config/db";
import app from "./app";

connectDB().catch((err) => {
  console.error("DB connection failed:", err);
  process.exit(1);
});

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`🚀 Server running on port ${PORT}`));
```

3. Update `package.json`: `"start": "node dist/index.js"`.
4. Update `nodemon.json`: `"exec": "node --use-system-ca -r ts-node/register src/index.ts"`.
5. **Deploy note for the owner (write this in your PR/commit message):** if the Render service's start command is `npm start`, nothing to do; if it's hard-coded `node dist/app.js`, it must be changed to `node dist/index.js` in the Render dashboard before the next deploy.
6. Verify: `npm run dev` still boots and answers `GET /` with the Hello World JSON.

## Step -1.1 — Install and configure Vitest

```bash
cd server
npm install -D vitest supertest @types/supertest mongodb-memory-server
```

`vitest.config.ts`:

```ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    environment: "node",
    setupFiles: ["./test/setup.ts"],
    exclude: ["**/node_modules/**", "**/dist/**", "**/*.ai.test.ts"],
    fileParallelism: false,     // one in-memory mongod at a time — deterministic
    testTimeout: 20000,
    hookTimeout: 120000,        // first run downloads the mongod binary (~100MB)
  },
});
```

`vitest.ai.config.ts` (live-Gemini suite only):

```ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    environment: "node",
    setupFiles: ["./test/setup.ts"],
    include: ["**/*.ai.test.ts"],
    fileParallelism: false,
    testTimeout: 60000,
    hookTimeout: 120000,
  },
});
```

`package.json` scripts:

```jsonc
"test": "vitest run",
"test:watch": "vitest",
"test:ai": "vitest run --config vitest.ai.config.ts"
```

## Step -1.2 — Test DB harness

`test/setup.ts`. Note: env vars are set at the **top level** (setup files execute before test files import anything, so module-load constants in `chat.controller.ts` see them).

```ts
import { beforeAll, afterAll, afterEach } from "vitest";
import mongoose from "mongoose";
import { MongoMemoryServer } from "mongodb-memory-server";

// Must exist before any src module loads. Never real secrets.
process.env.JWT_SECRET = process.env.JWT_SECRET || "test-jwt-secret";
process.env.JWT_EXPIRES_IN = process.env.JWT_EXPIRES_IN || "1h";
process.env.GEMINI_API_KEY = process.env.GEMINI_API_KEY || "test-gemini-key";

let mongod: MongoMemoryServer;

beforeAll(async () => {
  mongod = await MongoMemoryServer.create();
  await mongoose.connect(mongod.getUri());
});

afterEach(async () => {
  const collections = await mongoose.connection.db!.collections();
  for (const c of collections) await c.deleteMany({});
});

afterAll(async () => {
  await mongoose.disconnect();
  await mongod.stop();
});
```

Gotcha: the live `test:ai` suite needs the *real* `GEMINI_API_KEY` — the `||` fallback above preserves it when it's already set in the environment.

## Step -1.3 — Fixture lesson + seed helpers

`scripts/export-fixture.ts` (run once by the owner with the real `.env` present: `npx ts-node scripts/export-fixture.ts`):

```ts
import dotenv from "dotenv";
dotenv.config();
import mongoose from "mongoose";
import { mkdirSync, writeFileSync } from "fs";
import { Content } from "../src/models/content.model";

const LESSON_ID = "69e39d0b60d467bd515a4945";

async function main() {
  await mongoose.connect(process.env.MONGO_URI as string);
  const doc = await Content.findById(LESSON_ID).lean();
  if (!doc) throw new Error(`Lesson ${LESSON_ID} not found`);
  const fixture = {
    ...doc,
    _id: String(doc._id),
    owner_id: "FIXTURE_OWNER",        // remapped at seed time
    collaborators: [],
    collaborator_names: [],
    author_name: "Fixture Teacher",
  };
  mkdirSync("test/fixtures", { recursive: true });
  writeFileSync("test/fixtures/lesson.json", JSON.stringify(fixture, null, 2));
  console.log(`Wrote test/fixtures/lesson.json (${fixture.tiptap_json.length} chars of tiptap_json)`);
  await mongoose.disconnect();
}
main();
```

Commit `test/fixtures/lesson.json` to the server repo (it contains only lesson content, no secrets — but eyeball it once before committing). If the fixture doesn't exist yet when tests run, tests that need it must `throw new Error("Run: npx ts-node scripts/export-fixture.ts")` — a clear message, not a silent skip.

> **Phase -1 note (2026-07):** A **synthetic** Thai photosynthesis fixture is committed so `npm test` works out of the box without DB access. The owner should still run `npx ts-node scripts/export-fixture.ts` once (with real `.env`) to replace it with the real test lesson (`69e39d0b60d467bd515a4945`) before Phase 1+ integration work that depends on exact lesson content.

`test/helpers/seed.ts`:

```ts
import { readFileSync } from "fs";
import path from "path";
import jwt from "jsonwebtoken";
import { Types } from "mongoose";
import { User } from "../../src/models/user.model";
import { Content } from "../../src/models/content.model";

let seq = 0;

export async function seedUser(overrides: Record<string, unknown> = {}) {
  const user = await User.create({
    name: "Test Student",
    email: `student${++seq}@test.local`,
    password: "password-123",          // hashed by the model's pre-save hook
    ...overrides,
  });
  const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET as string, {
    expiresIn: "1h",
  });
  return { user, token };
}

export function loadLessonFixture(): Record<string, unknown> {
  const p = path.join(__dirname, "..", "fixtures", "lesson.json");
  try {
    return JSON.parse(readFileSync(p, "utf-8"));
  } catch {
    throw new Error("Missing lesson fixture. Run: npx ts-node scripts/export-fixture.ts");
  }
}

export async function seedLesson(
  ownerId: Types.ObjectId,
  overrides: Record<string, unknown> = {},
) {
  const { _id, owner_id, ...rest } = loadLessonFixture();
  return Content.create({ ...rest, owner_id: ownerId, ...overrides });
}
```

## Step -1.4 — Gemini mock pattern (copy into each API test file that touches `/api/chat`)

`vi.mock` factories are hoisted, so the mock fn must be created with `vi.hoisted`. This exact pattern works; do not deviate:

```ts
import { vi } from "vitest";

const { mockGenerateContent } = vi.hoisted(() => ({
  mockGenerateContent: vi.fn(async () => ({
    response: { text: () => "MOCK_AI_REPLY" },
  })),
}));

vi.mock("@google/generative-ai", () => ({
  GoogleGenerativeAI: class {
    getGenerativeModel() {
      return { generateContent: mockGenerateContent };
    }
  },
}));
```

`mockGenerateContent.mock.calls[0][0]` is the full prompt string — assert on its contents. Call `mockGenerateContent.mockClear()` in `beforeEach`. (Phase 0 changes this mock's shape — see Step 0.1.)

## Step -1.5 — Export the pure logic from `chat.controller.ts`

Minimal production change, zero behavior change: add `export` to `buildLearningPrompt`, `buildFeedbackPrompt`, `buildWriteEvaluationPrompt`, `isTransientAIError`, and `generateWithRetry` in `server/src/controllers/chat.controller.ts`.

## Step -1.6 — The test files

Every test below states its expected result. ✅ = verified against current source; assert exactly this. ⚠️ = characterize: run first, observe, then hard-code the observed value with a comment `// characterized YYYY-MM-DD`.

### `test/unit/prompts.test.ts`

| # | Case | Assert |
|---|---|---|
| 1 | `buildLearningPrompt("Q","CTX","UCTX")` | contains `=== Learning Content ===`, `CTX`, `=== Learner Chat History ===`, `UCTX`, `=== Learner Question ===`, `Q` ✅ |
| 2 | `buildLearningPrompt("Q","","")` | contains `(No learning content provided)` and `(No prior learner chat context provided)` ✅ |
| 3 | `buildFeedbackPrompt(...)` any args | contains `Never label the learner as 'wrong'` and `You MUST follow the provided evaluation level` ✅ — **the persona tripwire** |
| 4 | feedback, `evaluationLevel:"almost"`, `accuracyPercent:70` | contains `level: almost` and `accuracyPercent: 70` ✅ |
| 5 | feedback, mode `quick_check` vs `full_reflection` | quick contains `1-3 sentences`; full contains `5-8 sentences` ✅ |
| 6 | `buildWriteEvaluationPrompt(...)` | contains `The teacher guide answer is only a reference direction` and `Start with strengths` ✅ |
| 7 | Full-prompt snapshots (`toMatchSnapshot()`) of all three builders with fixed args | any future prompt edit shows up as a reviewed snapshot diff |
| 8 | Default language | all three contain the Thai instruction (match substring `ภาษาไทย`) ✅ (OUTPUT_LANGUAGE defaults to thai) |

### `test/unit/retry.test.ts`

Use `vi.useFakeTimers()` + `await vi.advanceTimersByTimeAsync(...)` — real backoff waits up to 10s.

| # | Case | Assert |
|---|---|---|
| 1 | `isTransientAIError` on messages: `"fetch failed"`, `"ECONNRESET"`, `"socket hang up"`, `"503 Service Unavailable"`, `"429 rate limit"`, `"model is overloaded"` | all `true` ✅ |
| 2 | on `"400 Bad Request"`, `"API key not valid"` | `false` ✅ |
| 3 | fake model fails twice with `Error("fetch failed")` then succeeds | resolves; called exactly 3 times ✅ |
| 4 | fake model always fails with `Error("fetch failed")` | rejects after exactly 5 calls ✅ |
| 5 | fake model fails once with `Error("invalid argument")` | rejects after exactly 1 call (non-transient → no retry) ✅ |

### `test/api/auth.test.ts` (supertest against imported `app`)

| # | Case | Assert |
|---|---|---|
| 1 | `POST /api/auth/register` valid body | ⚠️ status; body has `user` and `token`; `user.password` absent |
| 2 | register duplicate email | ⚠️ status + message |
| 3 | `POST /api/auth/login` correct creds | `{ user, token }` ⚠️ status |
| 4 | login wrong password | ⚠️ 4xx + message |
| 5 | `GET /api/auth/recheck` no header | 401, body exactly `{ message: "No token, authorization denied", code: "TOKEN_MISSING", forceRelogin: true, clearToken: true }` ✅ |
| 6 | recheck with expired token (`jwt.sign({id}, secret, {expiresIn: "-1s"})`) | 401, `code: "TOKEN_EXPIRED"`, `forceRelogin: true` ✅ |
| 7 | recheck with garbage token | 401, `code: "TOKEN_INVALID"` ✅ |
| 8 | recheck with valid token for a deleted user | 401, `code: "USER_NOT_FOUND"` ✅ |
| 9 | recheck with valid token | 200-ish ⚠️, returns fresh user |

Cases 5–8 are the client auto-logout contract — the most important tests in this file.

### `test/api/content.test.ts`

Seed: owner (`seedUser`), stranger (`seedUser`), lesson via `seedLesson(owner._id, { access_type })`.

| # | access_type | Caller | Assert |
|---|---|---|---|
| 1 | `public` | anonymous | 200, body includes `tiptap_json` ✅ — **Golden Rule 2 as a test** |
| 2 | `link-only` | anonymous | 200 ✅ |
| 3 | `private` | anonymous | **401** `{ message: "Login required to view this content" }` ✅ (yes 401, not 403) |
| 4 | `private` | stranger (valid token) | **403** `{ message: "No permission to view this content" }` ✅ |
| 5 | `private` | owner | 200 ✅ |
| 6 | `private` | collaborator (seed user, push into `collaborators`) | 200 ✅ |
| 7 | any | nonexistent id | 404 `{ message: "Content not found" }` ✅ |
| 8 | create: `POST /api/content/create` with token | 201 `{ content_id }` ✅; without token → 401 TOKEN_MISSING ✅ |
| 9 | update: `PUT /api/content/:id` with stale `clientUpdatedAt` (older than server `updatedAt`) | 409 `{ message: "Conflict: a newer version exists on the server.", serverUpdatedAt }` ✅ |
| 10 | update by non-owner/non-collaborator | 403 ✅ |

### `test/api/answers.test.ts`

| # | Case | Assert |
|---|---|---|
| 1 | `GET /api/content-answer/:contentId` first visit (logged in) | 200, body `{}` ✅ |
| 2 | `PUT /api/content-answer/:contentId` `{ blockId: "b1", answer: {...} }` then GET | map contains `b1` with the same value ✅ |
| 3 | PUT without `blockId` | 400 `{ message: "blockId is required" }` ✅ |
| 4 | `PUT /:contentId/bulk` `{ answers: {...} }` then GET | whole map replaced ⚠️ |
| 5 | any of the above without token | 401 `code: "TOKEN_MISSING"` ✅ |

### `test/api/chat.test.ts` (Gemini mocked per Step -1.4)

| # | Case | Assert |
|---|---|---|
| 1 | `POST /api/chat/ask` `{}` | 400 `{ message: "prompt is required" }` ✅ |
| 2 | ask `{ prompt: "อธิบายหน่อย", context: "LESSON", userContext: "HIST" }` | 200 `{ answer: "MOCK_AI_REPLY" }`; captured prompt contains all three sections ✅ |
| 3 | ask works **without any auth header** | 200 ✅ — Golden Rule 2 |
| 12 | feedback works **without any auth header** | 200 ✅ — Golden Rule 2 |
| 4 | `POST /api/chat/feedback` `{}` | 400 `{ message: "question is required" }` ✅ |
| 5 | feedback `evaluationLevel: "banana"` | captured prompt contains `level: incorrect` (invalid coerces to incorrect) ✅ |
| 6 | feedback `accuracyPercent: 150` / `-5` | prompt contains `accuracyPercent: 100` / `0` (clamped) ✅ |
| 7 | feedback `feedbackMode: "  FULL_REFLECTION  "` | prompt contains `Mode: full_reflection` (trim+lowercase) ✅ |
| 8 | `POST /api/chat/write-evaluate` missing `studentAnswer` | 400 `{ message: "question and studentAnswer are required" }` ✅ |
| 9 | write-evaluate happy path | 200 `{ feedback: "MOCK_AI_REPLY" }` ✅ |
| 13 | write-evaluate works **without any auth header** | 200 ✅ — Golden Rule 2 |
| 10 | `delete process.env.GEMINI_API_KEY` (restore in `finally`) → ask | 500 `{ message: "Missing GEMINI_API_KEY in server env" }` ✅ |
| 11 | mock rejects with `Error("boom")` (non-transient) → ask | 500 `{ message: "Gemini request failed" }` ✅ |

### `test/ai/live.ai.test.ts` (gated; costs real tokens)

Top of file:

```ts
const LIVE = process.env.RUN_AI_TESTS === "true" && !!process.env.GEMINI_API_KEY
  && process.env.GEMINI_API_KEY !== "test-gemini-key";
const describeLive = LIVE ? describe : describe.skip;
```

| # | Case | Assert |
|---|---|---|
| 1 | `POST /api/chat/ask` with `prompt: "ช่วยสรุปเนื้อหานี้ให้หน่อย"` and `context` = first 4000 chars of the fixture lesson's serialized text | 200; `answer.length > 20`; contains Thai chars (`/[฀-๿]/`) |
| 2 | `POST /api/chat/write-evaluate` with a question + short student answer from the fixture lesson | 200; non-empty feedback; Thai chars |

## Step -1.7 — **[client]** light test setup

```bash
cd client
npm install -D vitest happy-dom
```

`vitest.config.ts` (new file at client root — the `@` alias must be re-declared for Vitest):

```ts
import { defineConfig } from "vitest/config";
import path from "path";

export default defineConfig({
  resolve: { alias: { "@": path.resolve(__dirname, "src") } },
  test: { environment: "happy-dom", exclude: ["**/node_modules/**", "**/dist/**"] },
});
```

Add `"test": "vitest run"` to client `package.json` scripts.

**Refactor (small, safe):** extract the inline evaluation math from `QuestionChoiceView.tsx` (`handleSubmit`, the block computing `matchedCount` / `accuracyPercent` / `evaluationLevel` / `missedCorrect` / `wrongSelected` / `correctAnswer` / `userAnswer` — ✅ verified at lines ~437–471) into a new pure module `src/components/editor/extensions/questionEvaluation.ts`:

```ts
export interface Choice { text: string; correct: boolean }
export function evaluateChoiceAnswer(choices: Choice[], selectedIndices: number[]): {
  accuracyPercent: number;
  evaluationLevel: "correct" | "almost" | "incorrect";
  missedCorrect: string;   // " | "-joined
  wrongSelected: string;
  correctAnswer: string;
  userAnswer: string;
}
```

Semantics must be identical: `matchedCount` counts choices where `choice.correct === selectedIndices.includes(idx)`; percent = `round(matched/total*100)` (0 if no choices); level = 100 → correct, ≥60 → almost, else incorrect. Rewire `QuestionChoiceView.tsx` (and `QuestionBlankChoiceView.tsx` if its inline math matches — ⚠️ check; if it differs, leave it and note why).

Client test files:

- `src/components/editor/extensions/__tests__/questionEvaluation.test.ts` — all-correct → 100/correct; half-matched cases around the 60 boundary (59 → incorrect, 60 → almost); empty choices → 0/incorrect; missed/wrong strings join with ` | ` and skip empty texts.
- `__tests__/questionAgentContext.test.ts` — test the pure `buildQuestionAgentUserContext`: current block's history first; other blocks appended as `Block <id>:` sections; entries missing question or answer skipped ✅; result over 8000 chars keeps the **tail** ✅.
- `__tests__/questionFeedbackApi.test.ts` — `vi.mock("@/lib/axios")`: success trims whitespace; empty/missing feedback → returns fallback constant; axios rejection → returns fallback constant ✅ (note: Phase 0.4 rewrites these — update then).

## Step -1.8 — Docs + working agreement

- `server/CLAUDE.md`: replace "No tests yet" with the three commands + `test/` layout + "app.ts exports the app; index.ts boots it".
- `client/CLAUDE.md`: same for the client ("No tests yet" → vitest, where tests live).
- Root `CLAUDE.md`: change "There is no test suite in either half yet" to point at the commands.

**Phase -1 Definition of Done:** `cd server && npm test` green offline (unset `MONGO_URI`/real keys to prove it); `npm run test:ai` green with real key in `server/.env`; `cd client && npm test` green; deliberately editing `buildFeedbackPrompt` to remove the "Never label" line makes test fail (try it, revert); docs updated.

> **Phase -1 completion note (2026-07-10):** Shipped. Post-audit polish added: explicit Thai assertion on `buildLearningPrompt`; Golden Rule 2 anonymous-access tests on `/feedback` and `/write-evaluate`; exact 59% boundary case in client evaluation tests; `test/setup.ts` loads `server/.env` before fallbacks; `vitest.ai.config.ts` sets `RUN_AI_TESTS=true` so `npm run test:ai` runs live smoke when a real `GEMINI_API_KEY` is in `.env` (skips safely with the placeholder key). `QuestionBlankChoiceView` evaluation math intentionally left inline (blank-matching semantics differ from choice-matching). Owner should still run `npx ts-node scripts/export-fixture.ts` once to swap the synthetic fixture for the real lesson before Phase 1+ work that depends on exact content.

---

# PHASE 0 — Hygiene & safety net

## Step 0.1 — SDK migration `@google/generative-ai` → `@google/genai`

```bash
cd server
npm uninstall @google/generative-ai
npm install @google/genai
```

Changes confined to `chat.controller.ts` (all three handlers use the identical pattern):

| Old (`@google/generative-ai`) | New (`@google/genai`) |
|---|---|
| `import { GoogleGenerativeAI, GenerativeModel } from "@google/generative-ai"` | `import { GoogleGenAI } from "@google/genai"` |
| `const genAI = new GoogleGenerativeAI(apiKey)` | `const ai = new GoogleGenAI({ apiKey })` |
| `const model = genAI.getGenerativeModel({ model: MODEL_NAME })` | (no separate step) |
| `const result = await model.generateContent(prompt)` | `const result = await ai.models.generateContent({ model: MODEL_NAME, contents: prompt })` |
| `result.response.text().trim()` | `(result.text ?? "").trim()` |

`generateWithRetry` changes signature — it no longer receives a model object; it receives a thunk. Retry/backoff/transient logic stays byte-identical:

```ts
export async function generateWithRetry<T>(
  call: () => Promise<T>,
  tries = 5,
): Promise<T> {
  let last: unknown;
  for (let attempt = 1; attempt <= tries; attempt++) {
    try {
      return await call();
    } catch (error) {
      last = error;
      if (attempt === tries || !isTransientAIError(error)) throw error;
      const msg = error instanceof Error ? error.message : String(error);
      console.warn(`Gemini transient error (attempt ${attempt}/${tries}), retrying: ${msg}`);
      await new Promise((r) => setTimeout(r, 1000 * attempt));
    }
  }
  throw last;
}
```

Usage: `const result = await generateWithRetry(() => ai.models.generateContent({ model: MODEL_NAME, contents: finalPrompt }));`

**Update the test mock** (all files using Step -1.4's pattern) to:

```ts
const { mockGenerateContent } = vi.hoisted(() => ({
  mockGenerateContent: vi.fn(async () => ({ text: "MOCK_AI_REPLY" })),
}));

vi.mock("@google/genai", () => ({
  GoogleGenAI: class {
    models = { generateContent: mockGenerateContent };
  },
}));
```

Captured argument is now `mockGenerateContent.mock.calls[0][0]` = `{ model, contents }` — prompt assertions read `.contents`. Retry unit tests now test the thunk signature (simpler — no fake model object).

**Gate:** the entire Phase -1 suite (including prompt snapshots — builders untouched) must pass after migration. Run `npm run test:ai` once to prove the real API path works on the new SDK.

## Step 0.2 — Bot-only rate limiting + `optionalAuth` on chat routes

```bash
npm install express-rate-limit
```

New file `src/middlewares/rateLimit.middleware.ts`. **Factory pattern** (reads env at call time, so tests can construct with tiny limits):

```ts
import rateLimit, { ipKeyGenerator } from "express-rate-limit";
import { AuthRequest } from "../types";

// Bot guard ONLY (Golden Rule 1): limits are generous enough that no human
// student ever hits them. Tune via env, never add per-student quotas.
export function createChatRateLimiter() {
  const per10Min = Number(process.env.AI_RATELIMIT_PER_10MIN) || 60;
  const perDay = Number(process.env.AI_RATELIMIT_PER_DAY) || 1000;

  const keyGenerator = (req: AuthRequest) =>
    req.user?._id?.toString() ?? ipKeyGenerator(req.ip ?? "");

  const shortWindow = rateLimit({
    windowMs: 10 * 60 * 1000,
    limit: per10Min,
    keyGenerator,
    standardHeaders: true,
    legacyHeaders: false,
    message: { message: "Too many requests" },
  });
  const dayWindow = rateLimit({
    windowMs: 24 * 60 * 60 * 1000,
    limit: perDay,
    keyGenerator,
    standardHeaders: true,
    legacyHeaders: false,
    message: { message: "Too many requests" },
  });
  return [shortWindow, dayWindow];
}
```

(If the installed `express-rate-limit` version doesn't export `ipKeyGenerator`, use `(req) => req.user?._id?.toString() ?? req.ip` and note the IPv6 caveat in a comment.)

Wire-up:

- `src/app.ts`: add `app.set("trust proxy", 1);` right after `const app = express();` (Render sits behind a proxy; without this every client shares the proxy's IP and one bot could rate-limit the whole school).
- `src/routes/chat.routes.ts`:

```ts
import { optionalAuth } from "../middlewares/auth.middleware";
import { createChatRateLimiter } from "../middlewares/rateLimit.middleware";

const chatGuards = [optionalAuth, ...createChatRateLimiter()];
router.post("/ask", ...chatGuards, askChat);
router.post("/feedback", ...chatGuards, askFeedback);
router.post("/write-evaluate", ...chatGuards, askWriteEvaluation);
```

**No `protect` anywhere on chat routes** — `optionalAuth` only (Golden Rule 2).

Tests — new file `test/api/rateLimit.test.ts`:

```ts
// Env must be set BEFORE app import; use dynamic import + resetModules.
import { vi, describe, it, expect, beforeEach } from "vitest";
// ... Gemini mock (Step 0.1 shape) ...

beforeEach(() => { vi.resetModules(); });

it("returns 429 after the 10-min limit and keeps serving other IPs/users", async () => {
  process.env.AI_RATELIMIT_PER_10MIN = "3";
  const { default: app } = await import("../../src/app");
  // 3 asks → 200; 4th → 429 { message: "Too many requests" }
  // then same request with a seeded user's Bearer token → 200 (different key)
});
```

Also: anonymous ask still 200 (existing chat tests re-run under the new middleware — they already assert this).

## Step 0.3 — Gate prompt logging

In `chat.controller.ts`, both `console.log(prompt)` calls (✅ in `askFeedback` and `askWriteEvaluation`) become:

```ts
if (process.env.AI_DEBUG_PROMPTS === "true") console.log(prompt);
```

Test (optional but cheap): spy on `console.log`, call `/api/chat/feedback` mocked, assert no call containing `=== Question ===` when the env is unset.

## Step 0.4 — **[client]** honest AI errors

Current behavior (✅): every AI call in `questionFeedbackApi.ts` and `QuestionAgentView.tsx` catches all errors and returns a hardcoded string that looks like real feedback. Replace with an explicit failure signal + retry UI.

1. In `questionFeedbackApi.ts`: add `export class AiUnavailableError extends Error {}`. `requestQuestionFeedback` / `requestWriteEvaluation` / `requestFeedbackFollowup`: on axios error **or** empty/whitespace feedback, `throw new AiUnavailableError()`. Delete `FALLBACK_FEEDBACK` / `FALLBACK_WRITE_EVALUATION`.
2. Every caller (✅ verified callers: `QuestionChoiceView.tsx`, `QuestionBlankChoiceView.tsx`, `QuestionWriteView.tsx`, `QuestionBlankWriteView.tsx`, `FeedbackDiscussionPanel` via its parent views, plus `askAi` in `QuestionAgentView.tsx`) gets an `aiError` state: on catch, show a small inline box with Thai copy **"ตอนนี้ AI ตอบไม่ได้ ลองกดส่งอีกครั้งนะ 🙏"** and a retry button that re-invokes the same handler with the same inputs. The student's answer/draft must NOT be lost on failure (don't clear inputs before success).
3. `QuestionAgentView.askAi`: same pattern — remove `buildFallbackReply`, keep the question in the input box on failure.
4. Update client tests: `questionFeedbackApi.test.ts` now asserts `rejects.toThrow(AiUnavailableError)` for axios-reject and empty-feedback cases.

**Phase 0 Definition of Done:** full suite green in both halves; `test:ai` green (real SDK path); manual check — kill the server, submit an answer in the browser, see the retry UI, restart server, retry succeeds.

> **Phase 0 completion note (2026-07):** Implemented as specced. Two small deviations documented here:
> - **Rate-limit test** uses a minimal standalone Express app (not `vi.resetModules` + dynamic `import app`) because resetting modules re-registers Mongoose models and throws `OverwriteModelError`. The test still proves 429 + per-user key separation via `X-Forwarded-For`.
> - **`test:ai` on Windows** runs via `node --use-system-ca` (matching `nodemon.json`) plus `execArgv: ["--use-system-ca"]` in `vitest.ai.config.ts` — required on this machine's network for Gemini TLS.

---

# PHASE 1 — Agent core: server-side context + real conversations

## Step 1.1 — Lesson context service

New file `src/services/lessonContext.service.ts`.

TipTap node names & attributes (✅ all verified from the client `*Node.ts` files — these exact strings appear in the parsed `tiptap_json`):

| Node `type` | Attributes (`attrs`) |
|---|---|
| `QuestionChoice` | `id`, `question`, `choices: {text, correct}[]`, `answerType: "single"\|"multi"`, `feedbackMode` |
| `QuestionWrite` | `id`, `question`, `answer` (the teacher's guide answer), `feedbackMode` |
| `QuestionBlankChoice` | `id`, `template`, `choices: string[]`, `correctByBlank: number[]`, `feedbackMode` |
| `QuestionBlankWrite` | `id`, `template`, `blankAnswers: string[]`, `feedbackMode` |
| `QuestionAgent` | `id`, `title`, `chatHistory`, `collapsed` — **exclude from lesson text** (mirrors client behavior ✅) |

Public API:

```ts
export interface LessonQuestion {
  blockId: string;
  kind: "choice" | "write" | "blank_choice" | "blank_write";
  question: string;                 // question text or blank template
  choices?: { text: string; correct: boolean }[] | string[];
  guideAnswer?: string;             // QuestionWrite.answer / joined blankAnswers / correct choices
  feedbackMode: string;
}

export interface LessonContext {
  contentId: string;
  updatedAt: Date;
  title: string;
  accessType: "public" | "link-only" | "private";
  ownerId: string;
  collaboratorIds: string[];
  text: string;                     // serialized lesson (see rules)
  questions: LessonQuestion[];
}

export async function getLessonContext(contentId: string): Promise<LessonContext | null>;
export function clearLessonContextCache(): void;   // for tests
```

Serialization rules — port from the client's `serializeBlock` (✅ `client/src/components/editor/extensions/questionAgentContext.ts`), walking `JSON.parse(content.tiptap_json).content`:

- text node → its `text`; `hardBreak` → `\n`; inline children concatenated.
- `heading` → `H{level}: {text}`; `paragraph` → text; `codeBlock` → `Code:\n{text}`; `blockquote` → each inner line prefixed `> `; `bulletList` items → `- {text}`; `orderedList` items → `{n}. {text}`; `image` → `[Image] {alt}` or `[Image]`.
- `QuestionAgent` → skipped entirely.
- The four question nodes → emit a placeholder line in `text` — `[คำถามที่ {n}] {question-or-template}` — **and** push a full entry into `questions[]`. `guideAnswer`: for `write` = `attrs.answer`; for `blank_write` = `blankAnswers.join(" | ")`; for `blank_choice` = `correctByBlank.map(i => choices[i]).join(" | ")`; for `choice` = correct choices' texts joined `" | "`.
- Unknown node types → recurse into children (same as client).
- Malformed `tiptap_json` (JSON.parse throws) → treat as empty doc, `text: ""`, `questions: []` — never throw to the caller.
- **No char cap on the server** (the 12k cap was a client-payload constraint; the server sends it to a 1M-token-context model).

Cache: module-level `Map<string, { updatedAt: number; ctx: LessonContext }>`; hit only if `content.updatedAt` matches; on insert, if `map.size > 100`, delete the oldest inserted key. Load with `Content.findById(contentId).lean()`.

Unit tests `test/unit/lessonContext.test.ts`: hand-built minimal `tiptap_json` docs covering every rule above (one test per rule, plus: agent node excluded; question node emits both placeholder and `questions[]` entry with correct `guideAnswer` for all four kinds; malformed JSON → empty; cache returns same object until `updatedAt` changes — insert, mutate the Content doc, expect re-serialize). Integration test: `seedLesson` with the real fixture → `text.length > 0`, `questions.length` equals the number of question nodes in the fixture (⚠️ count them once and hard-code with a comment).

## Step 1.2 — `ChatSession` model

New file `src/models/chat_session.model.ts`:

```ts
import { Schema, model, Document, Types } from "mongoose";

export const MAX_SESSION_MESSAGES = 50;

export interface IChatMessage {
  role: "student" | "tutor";
  text: string;
  mode: string;
  createdAt: Date;
}

export interface IChatSession extends Document {
  user_id: Types.ObjectId;
  content_id: Types.ObjectId;
  block_id: string;                 // "" = lesson-wide chat
  messages: IChatMessage[];
  createdAt: Date;
  updatedAt: Date;
}

const chatSessionSchema = new Schema<IChatSession>(
  {
    user_id: { type: Schema.Types.ObjectId, ref: "User", required: true },
    content_id: { type: Schema.Types.ObjectId, ref: "Content", required: true },
    block_id: { type: String, default: "" },
    messages: [
      {
        role: { type: String, enum: ["student", "tutor"], required: true },
        text: { type: String, required: true },
        mode: { type: String, default: "free_chat" },
        createdAt: { type: Date, default: Date.now },
      },
    ],
  },
  { timestamps: true },
);

chatSessionSchema.index({ user_id: 1, content_id: 1, block_id: 1 }, { unique: true });
chatSessionSchema.pre("save", function () {
  if (this.messages.length > MAX_SESSION_MESSAGES) {
    this.messages = this.messages.slice(-MAX_SESSION_MESSAGES);
  }
});

export const ChatSession = model<IChatSession>("ChatSession", chatSessionSchema);
```

## Step 1.3 — Gemini wrapper (the mock seam from Phase -1 moves here)

New file `src/services/tutor/genaiClient.ts`:

```ts
import { GoogleGenAI } from "@google/genai";
import { generateWithRetry } from "../../controllers/chat.controller"; // or move retry helpers into this file and re-export

export interface TutorTurn { role: "user" | "model"; text: string }

export async function callTutorModel(opts: {
  model: string;
  systemInstruction: string;
  history: TutorTurn[];
  message: string;
}): Promise<string> {
  const apiKey = process.env.GEMINI_API_KEY;
  if (!apiKey) throw new Error("MISSING_GEMINI_API_KEY");
  const ai = new GoogleGenAI({ apiKey });
  const contents = [
    ...opts.history.map((t) => ({ role: t.role, parts: [{ text: t.text }] })),
    { role: "user", parts: [{ text: opts.message }] },
  ];
  const result = await generateWithRetry(() =>
    ai.models.generateContent({
      model: opts.model,
      contents,
      config: { systemInstruction: opts.systemInstruction },
    }),
  );
  return (result.text ?? "").trim();
}
```

Housekeeping: move `isTransientAIError` + `generateWithRetry` from `chat.controller.ts` into `src/services/tutor/retry.ts` and re-export from the controller (so Phase -1 unit tests keep passing unchanged). Tutor tests mock `@google/genai` exactly as in Step 0.1.

## Step 1.4 — `POST /api/chat/tutor`

Route: `router.post("/tutor", optionalAuth, ...createChatRateLimiter(), tutorChat);` in `chat.routes.ts`. New controller `src/controllers/tutor.controller.ts`.

Request body & validation (top-to-bottom; first failure wins):

| Field | Type | Rule | On violation |
|---|---|---|---|
| `contentId` | string | required, `mongoose.isValidObjectId` | 400 `{ message: "contentId is required and must be a valid id" }` |
| `blockId` | string | optional, default `""`, max 200 chars | coerce non-string → `""` |
| `mode` | string | one of `free_chat`, `question_feedback`, `write_evaluation`, `followup` | 400 `{ message: "mode must be one of: free_chat, question_feedback, write_evaluation, followup" }` |
| `message` | string | required non-empty after trim, max 8000 chars | 400 `{ message: "message is required" }` |
| `clientThread` | array | optional; sanitize: keep entries where `role ∈ {student, tutor}` and `text` is non-empty string; cap each text at 2000 chars; keep last 30 | silently sanitize (never 400) |
| `questionContext` | object | required when mode is `question_feedback` or `write_evaluation` | 400 `{ message: "questionContext is required for feedback modes" }` |
| `questionContext.question` | string | required non-empty in feedback modes | same 400 as above |
| `questionContext.guideAnswer` | string | optional, default `""` | — |
| `questionContext.evaluation.level` | string | one of `correct`, `almost`, `incorrect`, `ai_judge`; default `ai_judge` | coerce invalid → `ai_judge` |
| `questionContext.evaluation.accuracyPercent` | number | clamp 0–100, round; default 0 | coerce |
| `questionContext.feedbackMode` | string | `quick_check` (default) or `full_reflection` | coerce |

Controller flow (numbered — implement in this order):

1. Validate per the table.
2. `const ctx = await getLessonContext(contentId)`. `null` → 404 `{ message: "Content not found" }`.
3. Access control, mirroring `content/load` semantics (✅): if `accessType === "private"`: anonymous → 401 `{ message: "Login required to view this content" }`; logged-in but not owner/collaborator → 403 `{ message: "No permission to view this content" }`. `public`/`link-only` → everyone passes.
4. Resolve history: logged-in → `ChatSession.findOne({ user_id, content_id, block_id })`, use its `messages` (ignore `clientThread` entirely); anonymous → sanitized `clientThread`. Map to Gemini roles: `student → "user"`, `tutor → "model"`.
5. Build `systemInstruction` via `buildSystemInstruction(...)` (Phase-1 minimal version below; Phase 2 replaces its persona section).
6. Build the final user turn: for `free_chat`/`followup` it's `message` verbatim. For feedback modes, wrap with the task template:

```
[งานของเธอตอนนี้: ให้ฟีดแบ็กคำตอบของเพื่อน]
คำถามจากคุณครู: {questionContext.question}
แนวคำตอบของคุณครู (ใช้เป็นแนวทาง ไม่ใช่คำตอบเดียวที่ถูก): {guideAnswer || "(ไม่มี)"}
ผลการตรวจ: {level === "ai_judge" ? "ให้เธอประเมินเองอย่างใจดี เน้นคุณภาพการคิด" : `level=${level}, accuracy=${accuracyPercent}% — ต้องยึดตามนี้`}
โหมดความยาว: {feedbackMode}
คำตอบของเพื่อน: {message}
```

7. `const raw = await callTutorModel({ model, systemInstruction, history, message: finalUserTurn })` — model = `process.env.AI_TUTOR_MODEL || "gemini-2.5-flash-lite"` in Phase 1 (Phase 2 introduces the fast/tutor split).
8. `const { reply, suggestions } = parseTutorReply(raw)` — in Phase 1, `parseTutorReply` just returns `{ reply: raw, suggestions: [] }` (real parser lands in Phase 2 — create the function now so the contract is stable).
9. Persist (logged-in only): upsert the session, push `{ role: "student", text: <message as sent by student, NOT the wrapped template>, mode }` and `{ role: "tutor", text: reply, mode }`, save (pre-save hook trims to 50).
10. Respond 200 `{ reply, suggestions, sessionId: session?._id ?? null }`. On `MISSING_GEMINI_API_KEY` → 500 `{ message: "Missing GEMINI_API_KEY in server env" }`; on other AI failure → 502 `{ message: "AI request failed" }`.

Phase-1 minimal `buildSystemInstruction` (new file `src/services/tutor/persona.ts` — Phase 2 rewrites the persona block, keep the assembly function signature):

```ts
export function buildSystemInstruction(opts: {
  lesson: LessonContext;
  memoryDigest?: string;      // Phase 3
  teacherNote?: string;       // Phase 4
}): string
```

Phase-1 content: the tone rules from today's prompts (friend tone, never "wrong", admire first, one next step, Thai output per `AI_OUTPUT_LANGUAGE`), then `=== บทเรียน: {title} ===\n{text}`, then `=== คำถามในบทเรียน ===` listing `questions[]` with their guide answers, then a line telling the model guide answers are reference-only and must never be revealed verbatim unless the student already answered.

### Tests `test/api/tutor.test.ts` (Gemini mocked)

| # | Case | Assert |
|---|---|---|
| 1 | missing/invalid `contentId` | 400 exact message |
| 2 | bad `mode` | 400 exact message |
| 3 | empty `message` | 400 |
| 4 | feedback mode without `questionContext` | 400 |
| 5 | valid free_chat, **anonymous**, public lesson | 200 `{ reply: "MOCK_AI_REPLY", suggestions: [], sessionId: null }`; `ChatSession.countDocuments() === 0` |
| 6 | anonymous with `clientThread` of 2 turns | captured `contents` has 3 entries (2 history + final user turn), roles mapped student→user, tutor→model |
| 7 | anonymous, private lesson | 401 `Login required to view this content` |
| 8 | stranger token, private lesson | 403 |
| 9 | owner token, private lesson | 200 |
| 10 | logged-in first message | session created with 2 messages (student+tutor); `sessionId` returned |
| 11 | logged-in second message, same block | same session, 4 messages; captured `contents` includes turn-1 text (server-side history works) |
| 12 | logged-in sends a fake `clientThread` | captured `contents` does NOT contain the fake text (server history wins) |
| 13 | 52 messages pushed | session holds exactly 50 (`MAX_SESSION_MESSAGES`) |
| 14 | `question_feedback` with `level: "correct"`, `accuracy: 100` | final user turn contains `level=correct` and `accuracy=100%` |
| 15 | `write_evaluation` with `level: "ai_judge"` | final user turn contains the ai_judge instruction, not a `level=` line |
| 16 | systemInstruction captured | contains lesson title, a known phrase from fixture lesson text, and a known question's guide answer |
| 17 | mock rejects non-transient | 502 `{ message: "AI request failed" }` |
| 18 | nonexistent contentId | 404 |

**Old endpoints untouched in Phase 1** — the client keeps calling them; all Phase -1 chat tests must still pass.

**Phase 1 Definition of Done:** all above green; live check with real key + real DB: `POST /api/chat/tutor` for the test lesson answers a Thai question about a section that appears *late* in the lesson (proves full-lesson context), and remembers your name told in turn 1 when asked in turn 3.

> **Phase 1 completion note (2026-07-10):** Shipped. `cd server && npm test` — 93 tests (18 tutor API + 17 lessonContext unit/integration + all Phase -1/0 tests still green). `npm run build` green. `npm run test:ai` — 2 legacy smoke tests green. Test case 13 first-kept message is `msg-2` (52 pushed → slice(-50) drops turn 1 student+tutor). Manual live smoke against real DB/test lesson left to owner before Phase 2 persona work.

---

# PHASE 2 — Persona & prompt system

## Step 2.1 — The persona (rewrite `persona.ts`)

`TUTOR_NAME = "น้องมันฝรั่ง"` (a constant — the owner may rename). The system instruction template below is the deliverable; implement `buildSystemInstruction` to emit exactly this structure, with `{...}` placeholders filled and optional sections omitted when empty:

```
You are "น้องมันฝรั่ง", a warm, playful Thai study-buddy inside the Hot Potato learning app.
You are chatting with a school student who may have never used an AI before.

== YOUR CHARACTER ==
- Talk like a close friend who happens to love this subject: casual Thai (ภาษาพูด สุภาพแบบเพื่อน), short sentences, light humor, occasional emoji (max 1-2 per reply).
- You genuinely believe every student's idea has something good in it. Your job is to find it and say it FIRST, specifically — name the exact part of their thinking that works. Generic praise ("เก่งมาก") is forbidden; specific praise ("ตรงที่เธอเชื่อม A กับ B นี่คิดได้ดีมาก") is required.
- NEVER say the student is wrong. Never use "ผิด", "ไม่ถูก", "แย่". Frame gaps as "เกือบแล้ว", "ขาดอีกนิดเดียว", "ลองมองอีกมุมนึง".
- Keep formulas, symbols, and technical terms in their original language.

== HOW YOU TEACH (คิดกับ AI ไม่ใช่ตามที่ AI บอก) ==
- Coach thinking; don't dump answers. When the student asks for a direct answer to a teacher's question, use the hint ladder: (1) a nudge question → (2) a bigger hint → (3) walk through it together. Only give the full answer after they've genuinely tried, and even then explain the "why", starting from what they already got right.
- If the student seems lost or asks something vague, help them form a better question — suggest what they could ask.
- If something is not in the lesson, say so honestly, then give your best general guidance and connect it back to the lesson.
- Match the student's energy: short question → short answer. Never lecture when a sentence will do.

== LENGTH RULES ==
- Mode quick_check: 2-4 sentences.
- Mode full_reflection: 2-4 flowing paragraphs (no bullet walls, no scorecards, no numbered rubrics).
- Free chat: as short as a real friend would text, unless depth is asked for.

== OUTPUT LANGUAGE ==
{Thai-primary instruction, or English if AI_OUTPUT_LANGUAGE=english}

== OUTPUT FORMAT (STRICT) ==
After your reply, on a new line, output exactly:
[SUGGESTIONS]
- คำถามต่อยอดสั้นๆ ข้อ 1
- ข้อ 2
- ข้อ 3
Three short (under 15 words) follow-up questions the STUDENT could tap to ask you next — curious, fun, connected to what you just said. Written in the student's voice (e.g. "แล้วถ้า...จะเกิดอะไรขึ้น?").

{== สิ่งที่รู้เกี่ยวกับเพื่อนคนนี้ ==            ← Phase 3, only when memoryDigest present
{memoryDigest}
Use this to be warm and personal (e.g. connect examples to their interests). NEVER use it to limit or pigeonhole them.}

{== โน้ตจากคุณครู ==                        ← Phase 4, only when teacherNote present
{teacherNote}}

== บทเรียน: {lesson.title} ==
{lesson.text}

== คำถามในบทเรียน (ของคุณครู) ==
{for each q: [{blockId}] ({kind}) {question} | แนวคำตอบ: {guideAnswer || "-"}}
Guide answers are reference directions, not the only acceptable answers. Never reveal a guide answer verbatim before the student has attempted the question.
```

## Step 2.2 — `parseTutorReply` (real implementation)

In `src/services/tutor/parse.ts`:

- Find the **last** occurrence of a line that is exactly `[SUGGESTIONS]` (trimmed). Everything before it = `reply` (trimmed); lines after it starting with `- ` (max 3, each trimmed, truncated to 120 chars, empties dropped) = `suggestions`.
- No marker → `{ reply: raw.trim(), suggestions: [] }`.
- Marker but no valid `- ` lines → empty suggestions, reply still stripped of the marker block.

Unit tests `test/unit/parse.test.ts`: happy path (3 suggestions); no marker; marker mid-text as literal content earlier + real marker at end (last-occurrence rule); 5 `- ` lines → first 3 kept; `- ` line over 120 chars → truncated; whitespace around marker tolerated (`trim` before compare).

## Step 2.3 — Model strategy

In `tutor.controller.ts` (and later memory updater):

```ts
const FAST_MODEL = (process.env.AI_FAST_MODEL || "gemini-2.5-flash-lite").trim();
const TUTOR_MODEL = (process.env.AI_TUTOR_MODEL
  || process.env.AI_HEAVY_MODEL          // legacy alias
  || "gemini-2.5-flash").trim();
```

Task → model: `free_chat`, `followup`, `write_evaluation`, `question_feedback` with `full_reflection` → `TUTOR_MODEL`; `question_feedback` with `quick_check` → `FAST_MODEL`; Phase-3 memory updates → `FAST_MODEL`. Test: captured `{ model }` arg matches per mode (set env in test via `vi.resetModules` + dynamic import).

## Step 2.4 — Prompt-quality loop (owner-facing, not agent-facing)

Create `server/prompt-notes.md` with a 3-column log template (date/transcript-snippet/what-felt-off → change made). Agents editing prompt text must append an entry. Live test addition to `live.ai.test.ts`: one `/api/chat/tutor` free-chat call on the fixture lesson → assert non-empty Thai reply; log (don't assert) whether suggestions parsed, since the model may occasionally miss the format.

**Phase 2 Definition of Done:** suite green; live tutor call returns Thai reply + (usually) 3 suggestions; owner manually runs a 10-turn conversation and signs off on tone (this sign-off is a hard gate — do not proceed to Phase 3 without it); `prompt-notes.md` exists with the first entry.

> **Phase 2 completion note (2026-07-10):** Shipped. Persona rewritten (`TUTOR_NAME`, admire-first, hint ladder, `[SUGGESTIONS]` format). `parseTutorReply` moved to `parse.ts`. Model split: `quick_check` → `AI_FAST_MODEL`, everything else → `AI_TUTOR_MODEL`. Tests: 7 parse unit, 6 persona unit, 6 model unit, 4 new tutor API cases. `prompt-notes.md` created. **Default override:** `AI_TUTOR_MODEL` default is `gemini-2.5-flash` instead of spec'd `gemini-3-flash` — stable slug not in API (404 on live probe); `gemini-3.5-flash` / `gemini-3-flash-preview` work via env. Owner 10-turn tone sign-off still required before Phase 3.

---

# PHASE 3 — Student memory (logged-in only)

## Step 3.1 — Model

`src/models/student_memory.model.ts`:

```ts
import { Schema, model, Document, Types } from "mongoose";

export const MEMORY_LIST_CAP = 10;      // max entries per list
export const MEMORY_ITEM_MAX = 120;     // max chars per entry
export const MEMORY_TOPICS_CAP = 15;

export interface IStudentMemory extends Document {
  user_id: Types.ObjectId;
  interests: string[];
  strengths: string[];
  growth_areas: string[];
  preferences: string[];
  recent_topics: { content_id: Types.ObjectId; summary: string; updatedAt: Date }[];
  updatedAt: Date;
  createdAt: Date;
}

const studentMemorySchema = new Schema<IStudentMemory>(
  {
    user_id: { type: Schema.Types.ObjectId, ref: "User", required: true, unique: true },
    interests: { type: [String], default: [] },
    strengths: { type: [String], default: [] },
    growth_areas: { type: [String], default: [] },
    preferences: { type: [String], default: [] },
    recent_topics: [
      {
        content_id: { type: Schema.Types.ObjectId, ref: "Content" },
        summary: { type: String, default: "" },
        updatedAt: { type: Date, default: Date.now },
      },
    ],
  },
  { timestamps: true },
);

export const StudentMemory = model<IStudentMemory>("StudentMemory", studentMemorySchema);
```

## Step 3.2 — Memory updater

New file `src/services/tutor/memory.ts`, exporting **both** functions so tests can call them directly:

```ts
export async function runMemoryUpdate(opts: {
  userId: Types.ObjectId;
  contentId: Types.ObjectId;
  lessonTitle: string;
  recentTurns: { role: string; text: string }[];   // last 6
}): Promise<void>;

export function buildMemoryDigest(mem: IStudentMemory | null): string;  // "" when null
```

`runMemoryUpdate` flow: load current memory (or defaults) → call `FAST_MODEL` via `@google/genai` with JSON output:

```ts
const result = await ai.models.generateContent({
  model: FAST_MODEL,
  contents: prompt,
  config: {
    responseMimeType: "application/json",
    responseSchema: {
      type: Type.OBJECT,
      properties: {
        changed: { type: Type.BOOLEAN },
        interests: { type: Type.ARRAY, items: { type: Type.STRING } },
        strengths: { type: Type.ARRAY, items: { type: Type.STRING } },
        growth_areas: { type: Type.ARRAY, items: { type: Type.STRING } },
        preferences: { type: Type.ARRAY, items: { type: Type.STRING } },
        topic_summary: { type: Type.STRING },
      },
      required: ["changed"],
    },
  },
});
```

(`Type` is imported from `@google/genai`.) The prompt (English instructions, Thai values fine):

```
You maintain a tiny memory sketch about one student, used only to make a tutor warmer.
Current memory (JSON): {current memory as JSON}
New conversation turns (lesson: "{lessonTitle}"):
{role: text, one per line}

Update the memory. Rules:
- Only durable facts: interests ("ชอบฟุตบอล"), strengths ("เชื่อมโยงเหตุผลเก่ง"), growth areas ("ยังสับสนเรื่องเศษส่วน"), preferences ("ชอบคำอธิบายสั้นๆ").
- Each entry ≤ 120 chars, Thai preferred. Max {MEMORY_LIST_CAP} per list — drop the least useful when full.
- topic_summary: ≤ 200 chars, what the student worked on in this lesson just now.
- If nothing durable was revealed, return {"changed": false}.
- Never store anything sensitive, embarrassing, or judgmental. This is a friend's sketch, not surveillance.
```

Post-processing: if `changed === false` → still upsert `recent_topics` entry? No — do nothing at all. If `changed` → validate/enforce caps in code (truncate strings to 120, arrays to 10) — never trust model output for limits; upsert the doc; update `recent_topics`: replace the entry with the same `content_id` or prepend, cap at 15. Any error → `console.warn`, swallow (memory must never break tutoring).

Cadence + wiring in `tutor.controller.ts` step 9 (after persisting the session, logged-in only):

```ts
const everyN = Number(process.env.AI_MEMORY_EVERY_N_TURNS) || 3;
const studentTurns = session.messages.filter((m) => m.role === "student").length;
if (mode === "write_evaluation" || studentTurns % everyN === 0) {
  void runMemoryUpdate({ userId, contentId, lessonTitle: ctx.title,
    recentTurns: session.messages.slice(-6) }).catch((e) =>
    console.warn("memory update failed:", e));
}
```

The response to the student is **never** delayed or failed by memory work.

## Step 3.3 — Digest + injection

`buildMemoryDigest(mem)`: `""` if null/empty; otherwise a ≤600-char Thai block:

```
ความสนใจ: {interests joined ", "}
จุดแข็งที่เคยเห็น: {strengths}
กำลังฝึก: {growth_areas}
สไตล์ที่ชอบ: {preferences}
เพิ่งเรียน: {latest 2 recent_topics summaries}
```

Omit empty lines; hard-truncate to 600 chars. `tutor.controller.ts` step 5 passes `memoryDigest: buildMemoryDigest(await StudentMemory.findOne({ user_id }))` for logged-in users (one extra indexed query; acceptable). Anonymous → `undefined`, and the persona template omits the section entirely.

## Step 3.4 — Memory endpoints

In `chat.routes.ts`, both with `protect` (these are personal-data endpoints — the ONLY protected chat routes; AI usage itself stays open):

- `GET /api/chat/memory` → 200 with the user's memory doc or `{}` — transparency/debugging.
- `DELETE /api/chat/memory` → `deleteOne({ user_id })`, 200 `{ message: "Memory cleared" }` — the wipe switch.

## Step 3.5 — Tests

`test/unit/memoryDigest.test.ts`: null → `""`; full doc → all lines present, joined correctly; empty arrays omit their lines; >600 chars → truncated.

`test/api/memory.test.ts` (Gemini mocked; make the mock return valid memory JSON — for the new-SDK mock, `{ text: JSON.stringify({ changed: true, interests: ["ฟุตบอล"], ... }) }`):

| # | Case | Assert |
|---|---|---|
| 1 | `runMemoryUpdate` called directly, `changed: true` | `StudentMemory` doc upserted with capped, truncated fields |
| 2 | directly, `changed: false` | no doc created |
| 3 | model returns 15 interests of 300 chars | stored: ≤10 entries, each ≤120 chars (code-enforced caps) |
| 4 | model throws | resolves without throwing; no doc |
| 5 | tutor turn #3 by logged-in user (integration) | `vi.waitFor(() => StudentMemory.findOne(...))` finds the doc |
| 6 | tutor turns #1, #2 | no memory update triggered (cadence) — assert the memory-model mock not called |
| 7 | anonymous tutor turns (any number) | no doc ever |
| 8 | existing memory + tutor call | captured `systemInstruction` contains `สิ่งที่รู้เกี่ยวกับเพื่อนคนนี้` and a memory value |
| 9 | no memory + tutor call | captured `systemInstruction` does NOT contain that header |
| 10 | `GET /api/chat/memory` without token | 401 `TOKEN_MISSING` ✅ shape |
| 11 | `DELETE /api/chat/memory` then GET | memory gone |

**Phase 3 Definition of Done:** suite green; live two-lesson manual test — mention an interest in lesson A (logged in), start a chat in lesson B, the tutor references it naturally; `DELETE /api/chat/memory` then a new chat shows no trace; anonymous flows byte-identical to Phase 2.

> **Phase 3 completion note (2026-07-10):** Shipped. `StudentMemory` model, `memory.ts` (JSON-schema `AI_FAST_MODEL` updater + Thai digest builder), wired into `tutor.controller.ts` step 5 (digest) and step 9 (async update, cadence `AI_MEMORY_EVERY_N_TURNS` default 3). `GET`/`DELETE /api/chat/memory` with `protect`. Tests: 5 digest unit, 11 memory API/integration. `npm test` — 132 green. Owner live two-lesson sign-off still required.

---

# Appendix A — New packages

| Package | Repo | Phase | Purpose |
|---|---|---|---|
| `vitest`, `supertest`, `@types/supertest`, `mongodb-memory-server` | server (dev) | -1 | test harness |
| `vitest`, `happy-dom` | client (dev) | -1 | light unit tests |
| `@google/genai` (replaces `@google/generative-ai`) | server | 0 | current Gemini SDK |
| `express-rate-limit` | server | 0 | bot guard |

# Appendix B — New/changed API surface after Phase 3

| Method | Path | Guard | Phase |
|---|---|---|---|
| POST | `/api/chat/tutor` | optionalAuth + bot rate limit | 1 |
| GET | `/api/chat/memory` | protect | 3 |
| DELETE | `/api/chat/memory` | protect | 3 |
| POST | `/api/chat/ask` / `/feedback` / `/write-evaluate` | optionalAuth + bot rate limit (Phase 0) | retired in Phase 5, not before |

# Appendix C — Env vars introduced

| Var | Default | Phase |
|---|---|---|
| `RUN_AI_TESTS` | `false` | -1 |
| `AI_RATELIMIT_PER_10MIN` / `AI_RATELIMIT_PER_DAY` | `60` / `1000` | 0 |
| `AI_DEBUG_PROMPTS` | `false` | 0 |
| `AI_FAST_MODEL` | `gemini-2.5-flash-lite` | 2 |
| `AI_TUTOR_MODEL` | `gemini-2.5-flash` (override via env: `gemini-3.5-flash`; legacy alias: `AI_HEAVY_MODEL`) | 2 |
| `AI_MEMORY_EVERY_N_TURNS` | `3` | 3 |

# Appendix D — Update `CLAUDE.md` files when

- Phase -1: test commands + `app.ts`/`index.ts` split (both halves' docs + root).
- Phase 0: SDK name, rate-limit env vars, chat routes now `optionalAuth` (server doc "Gotchas" — the open-quota warning can be softened).
- Phase 1: new `/api/chat/tutor` in the API table; `ChatSession` in the data-model table; `lessonContext` service in the architecture tree.
- Phase 2: persona file location + `prompt-notes.md` loop.
- Phase 3: `StudentMemory` model + memory endpoints.
