# security-model.md — the security posture, on purpose

> Updated 2026-07-11 · verified against `auth.middleware.ts`, `rateLimit.middleware.ts`, `app.ts`. This is the *intentional* model; the reporting file is [`../server/SECURITY.md`](../server/SECURITY.md), and the enforced-invariant list is [`../ROADMAP.md`](../ROADMAP.md) §"Careful-not-to-break".

Hot Potato's security is shaped by an unusual constraint: **the two Golden Rules make "just require login" a forbidden move.** So the model is "open by default, guarded exactly where it must be" — the opposite of most apps. Read this before adding any guard, because the instinct to lock things down is often *wrong here*.

---

## 1. The guard matrix — and why each route sits where it does

Four levels (`auth.middleware.ts`):

| Level | Middleware | Behavior | Where it's used & why |
| --- | --- | --- | --- |
| **🌐 open** | none | no auth at all | health/status reads, and truly public data |
| **🔓 optionalAuth** | `optionalAuth` | sets `req.user` **only for `active` users** with a valid token; invalid/expired/blocked → silently anonymous, **never errors** | **AI tutor + public content reads.** This is Golden Rule 2 in code — anonymous must work fully. |
| **🔒 protect** | `protect` | requires a valid JWT (HS256) for an `active` user, else a **structured 401**; loads `req.user` (minus password) | account actions, authoring, the teacher copilot, per-user data |
| **👑 admin** | `protect` + `restrictTo("admin")` | protect, then role check (else **403**) | diagnostics only (`GET /api/status/`, `GET /api/users`) |

**The load-bearing rule:** AI and public-read routes use **`optionalAuth`, never `protect`.** Adding `protect` to `/api/chat/tutor` or `/api/content/load` is a **product regression**, not a hardening win. The *one* AI surface that is `protect`ed is the teacher copilot (`/api/creator/assist`) — teacher-only by design. (ADR-004 / ADR-009 in [`ideas.md`](ideas.md).)

`optionalAuth` deliberately swallows token errors: a blocked/expired token on a public route just degrades to anonymous rather than blocking a student mid-lesson.

---

## 2. The structured 401 contract (two-sided)

`protect` never returns a bare 401. It returns exactly:

```json
{ "message": "...", "code": "<CODE>", "forceRelogin": true, "clearToken": true }
```

`code ∈ TOKEN_MISSING | TOKEN_EXPIRED | TOKEN_INVALID | USER_NOT_FOUND | ACCOUNT_INACTIVE`. The client's axios response interceptor keys off `forceRelogin`/`clearToken` to auto-logout, and redirects to `/login` **only** on protected paths (carrying `reason`/`code`/`redirect`). **This shape is a contract across `auth.middleware.ts` ↔ `client/src/lib/axios.ts`** — change one side, change the other + the tests. (Flow: [`data-flow.md`](data-flow.md) §1–2; ADR-011.)

Note the intentional asymmetry ([`../notes.md`](../notes.md)): `login` returns an unstructured **403** for blocked accounts, while `protect` returns structured **401 ACCOUNT_INACTIVE**. Deliberate today; unify only with both sides in view.

---

## 3. Rate limiting — bot guard only, per-surface (Golden Rule 1)

`rateLimit.middleware.ts` exists **only to stop bots**, never to ration students:

- **Chat limiter** (`createChatRateLimiter`) — two windows, keyed by **user id when logged in, else client IP**: `AI_RATELIMIT_PER_10MIN` (default 60) + `AI_RATELIMIT_PER_DAY` (default 1000). Deliberately generous — no human hits them.
- **Auth limiter** (`createAuthRateLimiter`) — keyed by **IP**, `AUTH_RATELIMIT_PER_10MIN` (default 100). Generous on purpose: a whole classroom can share one school NAT IP.
- **Per-surface buckets:** the factory returns **fresh limiter instances** per call, so the teacher copilot's limiter is a *separate bucket* from the student tutor's — teacher AI usage never eats the student budget. (ADR-005.)
- **`app.set("trust proxy", 1)`** so limiter keys use the real client IP behind Render's proxy, not the proxy's.

Caveat ([`../notes.md`](../notes.md)): `Number(env) || default` treats `"0"` as unset, and in-memory buckets don't share across instances (only matters if Render ever runs >1 instance — Redis then).

---

## 4. CORS — env allowlist, fail-open in dev

`app.ts` builds the allowlist from `CORS_ORIGINS` (comma-separated, trailing slashes normalized):

- **Unset → open** (any origin) with a boot warning in `index.ts`. Fine for dev; **must be set in Render before wider release** (launch checklist, [`operations.md`](operations.md)).
- Requests **without an `Origin` header always pass** (health checks, curl, server-to-server).
- A disallowed browser origin simply gets **no `Access-Control-Allow-Origin` header** (the browser blocks it) — not a 500.

---

## 5. Secrets, hashing, and identity verification

- **Passwords:** bcrypt (cost 12) in a `pre("save")` hook; `select("-password")` everywhere; `comparePassword()` returns `false` when there's no password (Google-only accounts). Never returned by any endpoint.
- **JWT:** signed with `JWT_SECRET`, verified with `algorithms: ["HS256"]` explicitly (prevents alg-confusion). `JWT_EXPIRES_IN` sets lifetime; `recheckToken` slides the session.
- **Google Sign-In:** `POST /auth/google` verifies the GIS ID token's **signature and audience** server-side (`google-auth-library`, audience = `GOOGLE_CLIENT_ID`) before find-or-create. Auto-linking only by a **verified** email.
- **Secrets live in `.env`** (git-ignored) per half; never commit or log them.

---

## 6. Injection & abuse surfaces (what's defended, what's open)

- **Prompt injection via lesson content:** the lesson text injected into the tutor prompt is prefixed with a "data, not instructions" framing so a teacher can't embed commands (`lessonContext.service` / `persona.ts`). Internal prompt tags are stripped from replies (`stripInternalTags`) so they never leak to or persist for students.
- **Teacher copilot output:** the model **never** produces raw TipTap doc JSON; prose is markdown, blocks are typed JSON **validated server-side** (malformed dropped). Preview → accept is the human checkpoint. (ADR-009.)
- **XSS in AI replies:** rendered through `MarkdownMessage.tsx` with a strict allowlist; raw HTML is skipped.
- **⚠️ Known-open (tracked in [`../notes.md`](../notes.md)):** `searchContent` interpolates user `q` into a Mongo `$regex` — a **ReDoS / injection risk**; escape/limit it when you touch search. Ownership/access checks are inline in controllers (a shared `checkContentAccess` is a pending refactor).

---

## 7. Logging & observability privacy

- **No student messages or tokens are logged by default.** `AI_DEBUG_PROMPTS=true` logs full prompts (incl. student text) — **dev only, never production.**
- The in-memory error buffer stores **route + status + error name only** — never messages or request bodies; the full names are **admin-gated** on `GET /status/`. Public `GET /status/all` is sanitized. (ADR-006.)
- `db.ts` logs the Mongo host at boot to server logs only.

---

## 8. Threat-model honesty (a solo, free-tier project)

What this model does **not** do, on purpose: no refresh-token rotation (JWT in localStorage, `recheckToken` slides it); no CSRF tokens (token-in-header, not cookie-based auth); no WAF/DDoS layer beyond the bot limiter; no secrets manager (env vars on the host). These are acceptable trade-offs for the audience and budget — revisit if the project grows. See [`../server/SECURITY.md`](../server/SECURITY.md) for how to report an issue.
