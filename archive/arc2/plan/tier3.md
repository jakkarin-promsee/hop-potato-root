# Tier 3 — Hardening & Ops: Full Execution Plan

**For the implementing agent (Cursor).** This is a complete, self-contained spec for Tier 3 of [`../ROADMAP.md`](../ROADMAP.md) — Phase 3.A (Security & API hygiene) and Phase 3.B (Model strategy upkeep). Execute it top to bottom. Every file path, code sketch, decision, and test case you need is in this document — but always read the referenced source file before editing it; this plan describes the code as audited on 2026-07-10 and the file is the truth (Tier 1 and Tier 2 work may have landed since).

> **Read first, in this order:** [`../AGENT.md`](../AGENT.md) → [`../CLAUDE.md`](../CLAUDE.md) → [`../server/CLAUDE.md`](../server/CLAUDE.md) → [`../client/CLAUDE.md`](../client/CLAUDE.md). They are short and they will stop you from breaking things this plan assumes you know.

---

## 0. Ground rules (non-negotiable)

### The two Golden Rules — hardening edition
1. **Never ration tokens for real students.** This tier hardens *bot* and *abuse* surfaces only. Do not lower `AI_RATELIMIT_*` defaults, do not add per-student quotas, do not touch the chat rate limiter except where this plan says (it doesn't).
2. **Anonymous users keep FULL access.** Nothing in this tier may add `protect` to `POST /api/chat/tutor`, `GET /api/content/load`, `GET /api/content/search`, or `GET /api/status/all` (the anonymous Status page uses it). Hardening means locking down *admin/diagnostic/account* surfaces — never the student path.

### Git reality ⚠️
- `client/` and `server/` are **two separate git repositories**. There is **no** repo at the `Hot-Potato/` root. If `git status` shows foreign files (e.g. `ChronoForge-FPGA-Engine`), you are in the wrong directory — stop.
- This tier is ~95% server work. Client changes are verification-only unless the Status page needs a cosmetic fix (§3.A step A6).

### Working agreement
- Every work item ships **its own tests**. Done = `cd server && npm test` green (and `cd client && npm test` if the client was touched).
- `npm test` must stay **fully offline** — nothing you add may require network, a real DB, or `GEMINI_API_KEY`. `npm run test:ai` (live Gemini, costs money) only at the 3.B phase boundary.
- Update `server/CLAUDE.md` for every decision this tier makes — the ROADMAP acceptance for 3.A literally is "decisions documented in `server/CLAUDE.md`".
- This tier **adds two env vars** (`CORS_ORIGINS`, `AUTH_RATELIMIT_PER_10MIN`) — sanctioned here; ROADMAP §11's "no new env vars" applied through Tier 1 only.
- Do not touch `example_copy/`, `example_project/`, `test/` (root), `Ptest/`.

### Phase order & when to run
Execute **3.A → 3.B**. 3.A is independent of Tier 2 work and can land before or after it. 3.B needs a real `GEMINI_API_KEY` and owner availability for a short tone check.

### Careful-not-to-break list (Tier-3 specific)
| Thing | Where | Why |
| --- | --- | --- |
| `optionalAuth` + bot-only limiter on `/api/chat/tutor` | `server/src/routes/chat.routes.ts:9-12` | Golden Rules. This tier never touches chat guards. |
| Structured 401 contract | `auth.middleware.ts` ↔ `client/src/lib/axios.ts:39-42` | Client force-logout keys on **status 401 + `forceRelogin`/`clearToken` flags** (not on `code`). A5 adds a new `code` value — that is safe; changing the flags or the status is not. |
| Anonymous `/status` page | `client/src/pages/Status.tsx` → `status.store.ts:69` → `GET /api/status/all` | The page has no route guard and calls **only** `/status/all`. It must keep working logged-out after A6. |
| Tier 2.A reservation | future `GET/PUT /api/users/me/profile` | Keep the `/api/users` router mounted; A2 prunes routes but Profile work will add `me/profile` routes here later. |
| `app.set("trust proxy", 1)` | `server/src/app.ts:23` | Rate-limit keys need real client IPs behind Render's proxy. A7's new limiter depends on it too. |
| Error handler order | `server/src/app.ts:55` | `app.use(errorHandler)` stays last. |
| `generateWithRetry` | `services/tutor/retry.ts` | 3.B changes model *names*, never the retry wrapper. |
| Offline test suite | `server/test/setup.ts` | New tests must run without network. CORS/limiter tests configure env per-file (see the existing `test/api/rateLimit.test.ts` pattern). |

### Verification environment
- Server: `cd server && npm run dev` → `http://localhost:5000`. Client: `cd client && npm run dev` → `http://localhost:5173`.
- Test lesson: `http://localhost:5173/view/69e39d0b60d467bd515a4945`. Always verify **both auth states** (logged in + incognito).
- CORS cannot be fully verified from same-origin tools — use the supertest cases plus one manual `curl -H "Origin: https://evil.example" -i http://localhost:5000/api/status/all` check of response headers.

---

## 1. Security audit — verified current state (2026-07-10)

Every finding below was traced in source. Line numbers are from the audit date — **re-read each file before editing**.

| # | Severity | Finding | Evidence | Fixed by |
| --- | --- | --- | --- | --- |
| F1 | **Critical** | `POST /api/users` is unauthenticated **mass assignment**: `User.create(req.body)` — anyone on the internet can create a user with `role: "admin"` (and any `status`). It also responds with the full doc including the bcrypt hash. | `user.routes.ts:8`, `user.controller.ts:9-15` | A2 |
| F2 | **High** | `POST /api/auth/register` passes `role` straight from the body into `User.create` — anyone can self-register as `admin`. (The client never sends `role` — verified `auth.store.ts:43` — so stripping it breaks nothing.) | `auth.controller.ts:18,26` | A3 |
| F3 | **High** | `GET /api/users` (any logged-in user) returns `User.find()` **with bcrypt password hashes and all emails**. | `user.controller.ts:4-7` | A2 |
| F4 | Medium | CORS is wide open: `app.use(cors())` reflects any origin. | `app.ts:26` | A1 |
| F5 | Medium | No rate limiting on `POST /auth/login` / `POST /auth/register` — unlimited credential stuffing / account spam. | `auth.routes.ts` | A7 |
| F6 | Medium | `status: blocked \| suspended` is only checked at **login** (`auth.controller.ts:52`). `protect` never checks it — a blocked user's existing JWT keeps working until it expires (up to `JWT_EXPIRES_IN`). | `auth.middleware.ts:53-61` | A5 |
| F7 | Medium | Public `GET /api/status/database` and `GET /api/status/all` leak the MongoDB Atlas **host, port, and db name** to anyone. | `status.controller.ts:49-59,107-114` | A6 |
| F8 | Low | The central error handler echoes raw `err.message` to clients on 500 — internal details (Mongoose/driver messages) can leak. | `error.middleware.ts:9-10` | A8 |
| F9 | Doc drift | `AI_DEBUG_PROMPTS` and `AI_WRITE_EVAL_MODEL` are documented in `server/CLAUDE.md` env table but **no longer exist in code** (died with the legacy `chat.controller.ts` in Tier 0). Grep confirms zero source references. | `server/CLAUDE.md` env table | A9 |
| F10 | Decision | `restrictTo` exists and is imported (`user.routes.ts:3`) but **never used anywhere** — the `admin \| creator \| learner` roles are entirely unenforced. The ROADMAP asks for a decision. | `auth.middleware.ts:106-114` | A4 |

**Decisions baked into this plan (do not re-litigate; document them in `server/CLAUDE.md` as you implement):**
- Roles become minimally real: **`admin` gates diagnostics** (`GET /api/users`, `GET /api/status/`); `creator`/`learner` stay cosmetic for now (any logged-in user can create lessons — that is intentional for a friendly-users launch). This is the F10 decision.
- Registration always produces `learner`. Admins are created by promotion (A4 script), never by self-service.
- The Status page stays anonymous but sees **sanitized** data (connection state yes, Atlas hostname no).
- CORS is env-driven and **fails open with a loud warning when unconfigured** — a misconfigured Render env must degrade to today's behavior, never brick the app for students.

---

## Phase 3.A — Security & API hygiene

**Goal:** the API is safe to leave running unattended and exposed. All ten findings closed or consciously accepted, with tests pinning each fix. Size **S–M** (bigger than ROADMAP's "S" because the audit found F1–F3). Repos: server (client verification only).

### A1 — Lock down CORS (F4)

`server/src/app.ts`: replace `app.use(cors())` with an env-driven allowlist. **Implement exactly this behavior:**

```ts
function parseAllowedOrigins(): string[] {
  return (process.env.CORS_ORIGINS ?? "")
    .split(",")
    .map((s) => s.trim().replace(/\/+$/, ""))
    .filter(Boolean);
}

app.use(
  cors({
    origin: (origin, callback) => {
      const allowed = parseAllowedOrigins();
      if (allowed.length === 0) return callback(null, true); // unconfigured → open (see boot warning)
      if (!origin) return callback(null, true);              // curl / server-to-server / health checks
      callback(null, allowed.includes(origin.replace(/\/+$/, "")));
    },
  }),
);
```

Rules:
- **Read the env var inside the callback** (per request), not at module load — Render env changes apply on restart without code assumptions, and tests can set `process.env.CORS_ORIGINS` per file without re-importing the app.
- **Never `callback(new Error(...))`** — pass `callback(null, false)` via the boolean. A disallowed origin gets a normal response *without* `Access-Control-Allow-Origin` (the browser blocks it); erroring instead would spray 500s through the error handler.
- Requests with **no** `Origin` header always pass — CORS protects browsers, not curl; blocking origin-less requests would break health checks and `npm run test:ai`-style tooling.
- No `credentials: true` — auth is a Bearer header, not cookies.
- In `server/src/index.ts` (after `dotenv.config()`), warn loudly when unset: `if (!process.env.CORS_ORIGINS) console.warn("⚠️ CORS_ORIGINS not set — CORS is wide open. Set it in production (comma-separated origins).");`

Env var format (document in `server/CLAUDE.md` + `server/README.md`): `CORS_ORIGINS=https://<vercel-app>.vercel.app,http://localhost:5173,http://localhost:5174` — scheme + host, no trailing slash, no path, comma-separated. Local dev doesn't need it set at all.

### A2 — Fix the `/api/users` surface (F1, F3)

`server/src/routes/user.routes.ts`:
- **Delete `POST /`** and the `createUser` controller entirely — it duplicates `/api/auth/register` minus every safety check. Grep both halves for callers first (audit found none: the client never calls `/users`, and no server test file covers it).
- `GET /` becomes `router.get("/", protect, restrictTo("admin"), getUsers);` — `restrictTo` finally earns its import.

`server/src/controllers/user.controller.ts`: `getUsers` becomes `User.find().select("-password")` — even admins never need hashes over the wire. Keep the plain-array response shape.

### A3 — Harden registration (F2)

`server/src/controllers/auth.controller.ts`, `register`:
- Stop reading `role` from the body: `const { name, email, password } = req.body;` — the schema default (`learner`) applies. A client-sent `role` is silently ignored (matches the "old clients keep working" convention).
- Add the minimal validation register currently lacks (match the manual-validation style used in `changePassword`): `name`, `email`, `password` must be non-empty strings → else 400 `{ message: "Name, email and password are required" }`; `password.length < 8` → 400 `{ message: "Password must be at least 8 characters" }` (same rule `changePassword` already enforces at `auth.controller.ts:127`).
- **Read `server/test/api/auth.test.ts` first** — if existing cases register short passwords, update those fixtures rather than weakening the rule. The client (`Login.tsx`) may also need its copy checked so the 400 message surfaces; verify manually, only touch it if the error renders as nothing.

### A4 — The role story + admin bootstrap (F10)

- Roles decision (see §1): `admin` = diagnostics access; `creator`/`learner` cosmetic. Document this verbatim in `server/CLAUDE.md` (Auth section) so the next agent doesn't "helpfully" gate lesson creation.
- With self-service admin gone (A2+A3), the owner needs a promotion path. New script `server/scripts/promote-admin.ts` (follow the conventions of the existing `scripts/export-fixture.ts` / `scripts/measure-context.ts`):

```
Usage: npx ts-node scripts/promote-admin.ts <email> [role]
```

  - `dotenv.config()` → connect via `MONGO_URI` → `User.updateOne({ email: email.toLowerCase() }, { $set: { role } })` where `role` defaults to `"admin"` and must be one of `admin|creator|learner` (else exit 1 with usage) → print matched/modified counts → disconnect → exit 0. No password handling, no user creation — promotion only.
- Add npm script in `server/package.json`: `"promote": "ts-node scripts/promote-admin.ts"`.

### A5 — Enforce account status on every request (F6)

`server/src/middlewares/auth.middleware.ts`:
- Extend the union: `type TokenErrorCode = ... | "ACCOUNT_INACTIVE";`
- In `protect`, after the user loads (line ~53): `if (user.status !== "active") { sendReloginResponse(res, 401, "Your account has been suspended or blocked", "ACCOUNT_INACTIVE"); return; }`
- In `optionalAuth`, attach only active users: `if (user && user.status === "active") req.user = user;` — a blocked user browsing anonymously-accessible pages degrades to anonymous (Golden Rule 2 preserved: they can still *read* and use the AI; they lose the account's persistence).
- **Why 401 + the structured body, not 403:** the client interceptor (`client/src/lib/axios.ts:39-42`) only auto-logs-out on status 401 with the `forceRelogin`/`clearToken` flags. A new `code` value is invisible to it (it never reads `code`) — safe. Do not change the flags or status.

### A6 — Sanitize the status endpoints (F7)

`server/src/controllers/status.controller.ts`:
- `getDatabaseStatus`: delete the `host` / `port` / `db_name` spread (lines ~54-58). Keep `status`, `database`, `connection`, `ready_state`.
- `getAllStatus`: delete `host` / `db_name` from the database check (lines ~110-113). Everything else stays.
- `getServerStatus` (already `protect`-guarded, `status.routes.ts:13`): tighten to `protect, restrictTo("admin")` — it exposes hostname/memory internals and no client code calls it (verified: the Status page uses only `/status/all`).
- Leave `GET /env` as-is: it returns presence booleans only, and the env-var names are already public in the GitHub repo docs. Note this accepted risk in `server/CLAUDE.md`.
- **Client check (only change if broken):** `Status.tsx` renders Host/Port/Database rows conditionally (`{data.host && ...}`, lines ~198-200) — with the fields gone the rows simply disappear. Verify the page renders cleanly logged-out; no client edit expected.

### A7 — Bot-guard the auth endpoints (F5)

`server/src/middlewares/rateLimit.middleware.ts` — add alongside `createChatRateLimiter` (same style, same lib):

```ts
// Bot guard for credential endpoints. Generous on purpose: a whole classroom
// can share one school NAT IP, so the ceiling must comfortably exceed
// "40 students all logging in during first period".
export function createAuthRateLimiter() {
  const perWindow = Number(process.env.AUTH_RATELIMIT_PER_10MIN) || 100;
  return rateLimit({
    windowMs: 10 * 60 * 1000,
    limit: perWindow,
    keyGenerator: (req) => ipKeyGenerator(req.ip ?? ""),
    standardHeaders: true,
    legacyHeaders: false,
    message: { message: "Too many attempts, please try again later" },
  });
}
```

`server/src/routes/auth.routes.ts`: apply to `POST /register` and `POST /login` only (`recheck`/`change-password` are already behind `protect`). IP-keyed only — there is no user yet at login time. This is bot mitigation, not brute-force-proofing (bcrypt cost 12 is the real defense); say so in the code comment and docs.

### A8 — Stop echoing internals on 500 (F8)

`server/src/middlewares/error.middleware.ts`:

```ts
export const errorHandler = (err, req, res, next) => {
  console.error(err.stack ?? err);
  const message =
    process.env.NODE_ENV === "production"
      ? "Internal Server Error"
      : err.message || "Internal Server Error";
  res.status(500).json({ message });
};
```

Dev keeps real messages (the owner debugs locally); production goes generic. Full stack always logs server-side.

### A9 — Log/secrets sweep + resurrect the prompt-debug gate (F9)

- **Sweep (verify, then state the result in docs):** grep `console.` across `server/src` — as audited, nothing logs student messages, tokens, or keys by default (`db.ts` logs the Mongo host at boot to server logs only — acceptable; tutor/memory paths log errors and counts only). Confirm this is still true after Tiers 1–2 landed and fix anything that regressed.
- **Reimplement `AI_DEBUG_PROMPTS`** (documented but dead — and genuinely useful for the owner's prompt iteration loop): in `tutor.controller.ts`, at the point where the system instruction and final user turn are fully built and the model is resolved (shared by the stream and non-stream paths — read the current controller to place it exactly once), add:

```ts
if (process.env.AI_DEBUG_PROMPTS === "true") {
  console.log(`[AI_DEBUG] model=${model} mode=${mode} historyTurns=${history.length}`);
  console.log("[AI_DEBUG] systemInstruction:\n" + systemInstruction);
  console.log("[AI_DEBUG] finalUserTurn:\n" + finalUserTurn);
}
```

  (Adapt the variable names to the controller's actual ones.) Comment + docs must say: **dev-only flag — it logs student messages; never enable in production.**
- **Doc cleanup:** in `server/CLAUDE.md` (and `server/README.md` if it has an env table): remove `AI_WRITE_EVAL_MODEL` (dead in code — grep confirms only `AI_FAST_MODEL`/`AI_TUTOR_MODEL`/`AI_HEAVY_MODEL` are read); keep `AI_HEAVY_MODEL` documented as a legacy alias (it *is* still read, `tutor.controller.ts:28`); re-document `AI_DEBUG_PROMPTS` as implemented above.

### Server tests (`server/test/`)

New files follow the existing supertest style (`test/api/*.test.ts`, in-memory Mongo via `setup.ts`, seed helpers from `test/helpers/seed.ts`).

- **New `test/api/users.test.ts`:**
  1. `POST /api/users` → 404 (route gone).
  2. `GET /api/users` anonymous → 401; as `learner` → 403; as `admin` → 200.
  3. Admin response contains no `password` field on any element.
- **New `test/api/cors.test.ts`** (set `process.env.CORS_ORIGINS = "https://app.example"` at the top of the file, before importing `app` is not required since the callback reads per-request — but set it in `beforeAll` and restore in `afterAll` to be tidy):
  1. Request with `Origin: https://app.example` → response has `access-control-allow-origin: https://app.example`.
  2. Request with `Origin: https://evil.example` → response has **no** `access-control-allow-origin` header (and is otherwise a normal response, not a 500).
  3. Request with no `Origin` header → 200, works as before.
  4. `OPTIONS` preflight from the allowed origin → succeeds with the CORS headers.
  5. With `CORS_ORIGINS` unset (delete the env var in the test) → any origin is reflected (fail-open documented behavior).
- **Extend `test/api/auth.test.ts`** (read it first; keep existing cases green):
  1. Register with `role: "admin"` in the body → 201, but the created user's role is `learner` (assert via the response body and/or a direct `User.findOne`).
  2. Register with a 5-char password → 400.
  3. Register missing `name` → 400.
  4. Blocked-user enforcement: register + login → set `status: "blocked"` directly via the model → call any `protect`-ed route (e.g. `GET /api/auth/recheck`) with the still-valid token → 401 with `code: "ACCOUNT_INACTIVE"`, `forceRelogin: true`, `clearToken: true`.
  5. Suspended user calling `POST /api/chat/tutor` with their token → **200** (optionalAuth degrades to anonymous; Golden Rule 2 regression guard). Mock Gemini the way `test/api/tutor.test.ts` does.
- **Auth rate limiter:** add cases to `test/api/rateLimit.test.ts` mirroring its existing env-override pattern (set `AUTH_RATELIMIT_PER_10MIN=3` the way that file sets the chat limits): 4th login attempt from one IP → 429 with `{ message: "Too many attempts, please try again later" }`; chat limiter behavior untouched.
- **New `test/api/status.test.ts`:**
  1. `GET /api/status/all` anonymous → 200; `checks.database` has no `host`/`db_name`.
  2. `GET /api/status/database` anonymous → contains no `host`/`port`/`db_name` keys.
  3. `GET /api/status/` as learner → 403; as admin → 200.
- **New `test/unit/errorHandler.test.ts`:** call `errorHandler(new Error("secret detail"), req, mockRes, next)` directly with `NODE_ENV` stubbed to `"production"` → body message is `"Internal Server Error"`; with `"test"`/undefined → `"secret detail"`. (Stub/restore `process.env.NODE_ENV` carefully — vitest sets it to `test`.)
- **`AI_DEBUG_PROMPTS`:** one case in `test/api/tutor.test.ts` style — with the env flag `"true"`, spy on `console.log` and assert a `[AI_DEBUG]` line fires; with it unset, assert it doesn't. Keep the spy scoped so other tests' logs don't bleed.

### Acceptance checklist (3.A)

- [ ] `cd server && npm test` green, including every new file above; suite still fully offline.
- [ ] `curl -H "Origin: https://evil.example" -i http://localhost:5000/api/status/all` → no `access-control-allow-origin` header when `CORS_ORIGINS` is set locally; with it unset the boot log shows the ⚠️ warning.
- [ ] Live app check with `CORS_ORIGINS=http://localhost:5173` in `server/.env`: full lesson + tutor flow works logged-in **and** incognito at the test lesson (Golden Rule 2 proof under locked CORS).
- [ ] `POST /api/users` → 404. Registering with `role: "admin"` yields a `learner`. `GET /api/users` needs an admin.
- [ ] `npm run promote -- <owner-email>` flips the owner's account to admin (verify against the local DB), and that account can open `GET /api/users` + `GET /api/status/`.
- [ ] Block a test user in the DB → their existing session force-logs-out on the next protected call, with the login page showing the reason (`?reason=` flow) — and in incognito they can still read the lesson and use the tutor.
- [ ] `/status` page renders cleanly logged-out: no Host/Port/Database rows, no `undefined` text.
- [ ] All decisions (roles story, fail-open CORS, `/env` accepted risk, auth limiter rationale) written into `server/CLAUDE.md`; the "CORS is wide open" gotcha replaced; API-surface tables updated (`/users`, `/status` guards); env table gains `CORS_ORIGINS` + `AUTH_RATELIMIT_PER_10MIN`, drops `AI_WRITE_EVAL_MODEL`.

### Owner actions after merge (put this list in the PR/commit description too)

1. **Render dashboard:** set `CORS_ORIGINS=https://<the-real-vercel-domain>` (add preview domains if you use them, comma-separated, no trailing slash). Until set, the server stays open and warns at boot — set it the same day.
2. **Promote your account:** run `cd server && npm run promote -- <your-email>` with the production `MONGO_URI` in `.env` (or run it in the Render shell).
3. **Smoke from the real Vercel app** after deploy: open a lesson, chat with the tutor (logged-in + incognito), open `/status`. If the app can't reach the API, the `CORS_ORIGINS` value doesn't match the deployed origin exactly — fix the env var, don't revert the code.

**Commit (server):** `feat: security hardening — CORS allowlist, users/status lockdown, auth rate limit, account-status enforcement` (split into logical commits if you prefer; keep tests with their feature).

---

## Phase 3.B — Model strategy upkeep

**Goal:** re-test the Gemini lineup, pick current-best **cheap** defaults, and leave a repeatable check for next time. Size **XS**. Repo: server. **Recurring trigger:** re-run whenever Google ships a new Flash generation — no fixed date.

### Current state you're building on

- Defaults: `AI_FAST_MODEL` → `gemini-2.5-flash-lite` (`tutor.controller.ts:22`, duplicated in `services/tutor/memory.ts:16`); `AI_TUTOR_MODEL` → `gemini-2.5-flash` with legacy `AI_HEAVY_MODEL` fallback (`tutor.controller.ts:25-31`).
- Why frozen: on 2026-07-10 the stable `gemini-3-flash` slug **404'd** while `gemini-3.5-flash` and `gemini-3-flash-preview` worked (`server/prompt-notes.md`, first row).
- Routing: `question_feedback`/`quick_check` + memory updates → fast model; free chat / reflection / write-eval → tutor model. Golden Rule 1's economics depend on staying in the flash/flash-lite class.

### B1 — A repeatable model-check script

New `server/scripts/check-models.ts` (conventions of the existing scripts; `dotenv.config()`, requires `GEMINI_API_KEY`, **costs a few tokens** — never wire it into `npm test`):

1. List available models via the `@google/genai` SDK (`ai.models.list()` — it returns an async pager; read the SDK's types in `node_modules` if the shape differs) and print every model name matching `/flash|lite/i`.
2. For a candidate list — `["gemini-3-flash", "gemini-3-flash-lite", "gemini-3.5-flash", "gemini-3.5-flash-lite", "gemini-3-flash-preview", "gemini-2.5-flash", "gemini-2.5-flash-lite"]` (declare as a const at the top; future runs edit it) — attempt a minimal `generateContent` (`"ตอบคำเดียว: สวัสดี"`) per slug and print a table: slug · ok/404/error · latency ms.
3. Exit 0 regardless of individual failures (it's a report, not a gate).

Add npm script: `"check-models": "ts-node scripts/check-models.ts"`.

### B2 — Decision rules (apply, don't deliberate)

- **Stable slugs beat previews.** Only adopt a `-preview` slug as a default if no stable slug of that generation works — and prefer not to; previews get pulled.
- **Newest generation that passes** the script *and* the live smoke wins.
- **Class discipline:** fast/memory default stays in the *-lite* class; tutor default stays in the plain *flash* class. **Never** a pro/ultra-class default — that's Golden Rule 1's cost posture.
- If nothing newer passes, keeping the 2.5 defaults **is a valid outcome** — the deliverable is a *documented, dated decision*, not necessarily a change.

### B3 — Centralize the defaults, then update them

The fast-model default string currently lives in **two places** (`tutor.controller.ts:21-23` and `memory.ts:16`) — a drift bug waiting to happen. Fix while you're here:

- New `server/src/services/tutor/models.ts` exporting `getFastModel()` and `getTutorModel()` (move the bodies verbatim, including the `AI_HEAVY_MODEL` fallback and `.trim()`).
- `tutor.controller.ts` imports them and **re-exports** (`export { getFastModel, getTutorModel }`) so existing imports and `test/unit/tutorModel.test.ts` keep working; keep the deprecated `FAST_MODEL`/`TUTOR_MODEL` consts re-exported too (read the controller first — Tier 1 may have moved things).
- `memory.ts` deletes its local copy and imports `getFastModel`.
- Update the default strings per the B1/B2 outcome. One unit test asserts controller and memory resolve from the same source (e.g. both reflect an `AI_FAST_MODEL` env override).

### B4 — Verify, document, gate

- `cd server && npm test` (offline) green, then `npm run test:ai` with the real key — the live smoke exercises all four tutor modes against the new defaults.
- `server/prompt-notes.md`: dated row with the script's result table (which slugs passed), the chosen defaults, and why.
- Docs: env-table default values in `server/CLAUDE.md` + `server/README.md`; update the stale "`gemini-3-flash` 404'd" guidance if it no longer holds.
- **Owner mini tone-gate:** a model swap changes tone even with identical prompts. One 10-turn Thai conversation on the test lesson (any card + free chat), owner signs off, logged in `prompt-notes.md`. If tone regressed and prompt tweaks can't fix it cheaply, **revert the default** and note it — tone beats novelty.

### Acceptance checklist (3.B)

- [ ] `npm run check-models` prints the availability table (run output pasted into prompt-notes).
- [ ] Defaults updated (or explicitly re-confirmed) per the decision rules; fast default defined in exactly one module.
- [ ] `npm test` green offline; `npm run test:ai` green live.
- [ ] `server/CLAUDE.md`, `README.md`, `prompt-notes.md` all reflect the chosen defaults with the date.
- [ ] Owner tone sign-off logged (only if the default actually changed).

**Commit (server):** `chore: re-test Gemini lineup, centralize model defaults` (+ `feat:` if defaults changed).

---

## Tier-3 definition of done

1. Both phase checklists fully ticked; the three **owner actions** after 3.A communicated (Render `CORS_ORIGINS`, promote-admin, production smoke).
2. `cd server && npm test` green and still fully offline; `cd server && npm run build` compiles; client suite green **if** the client was touched.
3. One live pass on the test lesson, **both auth states**, with `CORS_ORIGINS` set locally: lessons load, tutor chats (stream + fallback), Status page renders, a blocked account force-logs-out but can still learn anonymously.
4. `server/CLAUDE.md` tells the truth: new guards in the API tables, the roles decision, the CORS/env-var story, no dead env vars documented.
5. Nothing committed from the `Hot-Potato/` root; server (and client, if touched) committed separately.
