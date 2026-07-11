# Tier 2 — Product Completeness (pages & accounts) — Execution Plan

**For the implementing agent (Cursor).** This is the complete, self-contained spec for Tier 2 of [`../ROADMAP.md`](../ROADMAP.md) §6. Every code-level claim below was verified against the codebase on 2026-07-10. Follow it phase by phase; each phase is independently shippable.

> **Read first, in this order:** [`../AGENT.md`](../AGENT.md) → [`../CLAUDE.md`](../CLAUDE.md) → [`../client/CLAUDE.md`](../client/CLAUDE.md) → (for 2.A only) [`../server/CLAUDE.md`](../server/CLAUDE.md). Then come back here.

---

## 0. Ground rules (non-negotiable)

1. **Golden Rule 1 — never ration tokens.** Nothing in Tier 2 touches AI quotas. Don't add any.
2. **Golden Rule 2 — anonymous users keep full access.** The new profile endpoints are 🔒 `protect` and that is **allowed** (profile = account persistence, not AI or public content). Never add `protect` to any content-read or `/api/chat/tutor` path while working here.
3. **Git:** `client/` and `server/` are **separate repos**. Run git only inside them, never from `Hot-Potato/`. If `git status` shows foreign files (e.g. `ChronoForge-FPGA-Engine`), you are in the wrong directory — stop.
4. **Tests are the definition of done.** Each phase ships tests for its own acceptance criteria. A phase is not done until `cd server && npm test` and `cd client && npm test` are green in every half you touched, and `cd client && npm run lint` is clean.
5. **Docs:** after each phase, update `ROADMAP.md` (§6 phase status + §9 audit table) and the relevant `CLAUDE.md` sections listed at the end of each phase.
6. **Phone-first:** every UI change gets a ~390 px viewport check. Playwright MCP is mirrored for Cursor in `.cursor/mcp.json` — use it against `npm run dev` (client :5173, server :5000) for the manual checks.
7. **Commits:** conventional messages (`feat:`, `fix:`, `docs:`, `test:`), one repo at a time.

**Before starting:** pull the latest in both repos and record the baseline test counts (Tier 1 work — SSE streaming, personalities — is landing in parallel; `server/test/api/tutorStream.test.ts` and `server/test/unit/personality.test.ts` already exist). Tier 2 barely overlaps Tier 1 file-wise (auth/pages vs. chat), so conflicts should be rare — but always start from a green baseline so you can tell your breakage from someone else's.

**Recommended order:** 2.A → 2.B → 2.C (2.C is the closing sweep; running it last lets it verify the whole tier). The phases are technically independent — any order works if priorities change.

---

## 1. Verified current state (what you'll find in the code)

These are the facts the phases below are built on. If reality has drifted (Tier 1 landing in parallel), trust the code and adapt — but the shape of the work stays the same.

| Fact | Where |
| --- | --- |
| `Profile.tsx` is 100% mock: local `useState` form (firstName/lastName/nickname/bio/age/role), Save button has no `onClick`, avatar camera button has no handler, nothing loads from `auth.store`. | `client/src/pages/Profile.tsx` |
| `UserData` model exists (`user_id` unique, `avatar?`, `bio?`, `metadata` Mixed) but **no controller or route touches it**. Note the filename: `user_data.model.ts`. | `server/src/models/user_data.model.ts` |
| `/api/users` is scaffolding: `GET /` (protect, dumps all users **including password hashes** — `User.find()` without `.select`) and open `POST /`. Tier 3.A owns cleaning these two; don't expand them, just add the new `/me/profile` routes beside them. | `server/src/routes/user.routes.ts`, `controllers/user.controller.ts` |
| Login always redirects to `/explore` after success; `?reason=` is dumped into the same red error state as validation failures; "Forgot password?" and "Continue with Google" buttons are **dead** (no handlers, no backend). | `client/src/pages/Login.tsx` (redirect at line ~31, dead buttons ~103-110 and ~143-163) |
| `ProtectedRoute` redirects to `/login` without remembering where you were. `PublicRoute` always bounces to `/explore`. | `client/src/components/ProtectedRoute.tsx`, `PublicRoute.tsx` |
| `RequireLogin` has a stray `console.log(token)` — **logs the JWT to the console**. | `client/src/components/RequireLogin.tsx:25` |
| The axios interceptor force-relogin redirect is `/login?reason=<msg>` with no way back to the original page. `isProtectedPath` + prefix list live in the same file. | `client/src/lib/axios.ts` |
| The server's structured 401 body is `{ message, code, forceRelogin: true, clearToken: true }` with `code` ∈ `TOKEN_MISSING \| TOKEN_EXPIRED \| TOKEN_INVALID \| USER_NOT_FOUND`. The client currently uses `message` only. | `server/src/middlewares/auth.middleware.ts` |
| `Index.tsx` is a dead stub (`<div>Index</div>`), still imported in `App.tsx` (line ~17) but never routed. | `client/src/pages/Index.tsx`, `client/src/App.tsx` |
| Logged-in TopNav shows **Settings twice** — it's in both `publicNavItems` and `authOnlyNavItems` and the two arrays are concatenated (also a React duplicate-key bug: two entries with `key="/settings"`). | `client/src/components/TopNav.tsx:24-44` |
| Explore's "Bookmarked" tab is in-memory `useState<Set<string>>` — bookmarks silently vanish on reload. | `client/src/pages/Explore.tsx:18` |
| Dashboard delete is one click, permanent, no confirmation; the delete button is `opacity-0 group-hover:opacity-100` so it's **unreachable on touch devices**. Dashboard reads no error state from `content.store` (failed fetches are silent). | `client/src/pages/Dashboard.tsx:64-69, 193-204` |
| `Guide.tsx` predates the AI tutor: 8 feature sections, **zero mention of the tutor or questions**, and advertises "Feedback & Comments" — a feature that does not exist. Bottom CTA has no buttons. | `client/src/pages/Guide.tsx` |
| Landing never mentions the AI tutor either; showcase cards are hardcoded fakes ("Ms. Chen", "Dr. Kim"). UI brand everywhere is **"Intuita"**, not "Hot Potato". | `client/src/pages/Landing.tsx` |
| `author_name` / `collaborator_names` on `Content` are denormalized snapshots; today a user rename would leave them stale (documented gotcha). `buildAuthorSnapshot(ownerId, collaboratorIds)` exists. | `server/src/services/contentAuthorSnapshot.service.ts` |
| Avatar upload building blocks already exist: `uploadImage(file, {onProgress})` (unsigned Cloudinary upload) and `applyTransform(url, params)`. | `client/src/lib/cloudinary.ts` |
| shadcn/ui primitives present: button, card, input, label, textarea, select, dropdown-menu. **No `alert-dialog` yet** — add it via the shadcn CLI when 2.C needs it. Sonner is installed but the global `<Toaster/>` is commented out in `App.tsx` — use **inline** success/error messages, don't re-enable the toaster as a side quest. | `client/src/components/ui/`, `client/src/App.tsx:1-3, 45-47` |
| Test conventions: server = Vitest + supertest + in-memory Mongo in `server/test/` (`api/*.test.ts`, `unit/*.test.ts`, `helpers/seed.ts` gives `seedUser()`/`seedLesson()`); client = Vitest + happy-dom + @testing-library/react in `__tests__/` folders next to code. `client/vitest.config.ts` picks up any `*.test.ts(x)` under `src/`. | `server/test/`, `client/src/components/editor/extensions/__tests__/` |

---

## 2. Phase 2.A — Real Profile page

**Goal:** `/profile` becomes a real account page: loads real data, saves to the server, avatar upload works. Size M · Repos: both.

### 2.A.0 Product decisions (already made — implement as written)

This is a learning app, not a social network. The page keeps only fields that serve a student:

| Field | Storage | Keep? |
| --- | --- | --- |
| Display name | `User.name` | ✅ editable (see rename propagation below) |
| Email | `User.email` | ✅ read-only display |
| Avatar | `UserData.avatar` (Cloudinary secure_url) | ✅ upload via existing Cloudinary flow |
| Nickname | `UserData.metadata.nickname` | ✅ — future hook: what น้องมันฝรั่ง can call the student |
| Bio | `UserData.bio` | ✅ short free text |
| firstName/lastName split | — | ❌ cut (one `name` field is enough) |
| Age | — | ❌ cut (serves nothing) |
| Role select (student/teacher) | — | ❌ cut (the mock's values don't even match the server enum `admin\|creator\|learner`; role stays server-managed) |
| Theme row + Change-password button | already on the page | ✅ keep as-is |

### 2.A.1 Server — `GET/PUT /api/users/me/profile`

**Routes** (`server/src/routes/user.routes.ts`): add above the existing routes —

```ts
router.get("/me/profile", protect, getMyProfile);
router.put("/me/profile", protect, updateMyProfile);
```

**Controller** — add `getMyProfile` / `updateMyProfile` to `server/src/controllers/user.controller.ts` (typed `(req: AuthRequest, res: Response) => Promise<void>`, manual inline validation like `tutor.controller.ts` — no validation library).

**GET response** (when no `UserData` row exists yet, return empty-string defaults — do **not** create a row on read):

```json
{
  "user":    { "id": "...", "name": "...", "email": "...", "role": "learner" },
  "profile": { "avatar": "", "bio": "", "nickname": "" }
}
```

`nickname` is read from / written to `UserData.metadata.nickname` — flatten it in the API so the client never touches `metadata` directly (the Mixed bag stays free for future use).

**PUT request** — `{ name?, avatar?, bio?, nickname? }`, all optional, **at least one required** (else 400). Validation (400 with a specific `message` on each failure):

| Field | Rule |
| --- | --- |
| `name` | string, trimmed, 1–100 chars (empty after trim → 400) |
| `avatar` | string, `""` (clear) or an `https://res.cloudinary.com/` URL, ≤ 500 chars |
| `bio` | string, trimmed, ≤ 500 chars (empty allowed) |
| `nickname` | string, trimmed, ≤ 50 chars (empty allowed) |

Persistence: `name` → save on the `User` doc. avatar/bio/nickname → upsert `UserData` via `findOneAndUpdate({ user_id }, { $set: ... }, { upsert: true, new: true })` — only `$set` the fields present in the body. **Important Mixed-type gotcha:** set nickname with dot-path `{ "metadata.nickname": value }` inside `$set` so you don't clobber other future metadata keys. PUT responds with the same shape as GET (fresh values).

**Rename propagation** (closes the documented `author_name` staleness gotcha): add to `contentAuthorSnapshot.service.ts`:

```ts
export async function refreshAuthorSnapshotsForUser(userId: Types.ObjectId): Promise<void>
```

- `Content.updateMany({ owner_id: userId }, { author_name: newName })` — cheap bulk fix for owned lessons.
- For collaborations: `Content.find({ collaborators: userId }).select("owner_id collaborators")`, rebuild each with the existing `buildAuthorSnapshot`, save. (Collections are small; a loop is fine.)

Call it from `updateMyProfile` **only when `name` actually changed**.

### 2.A.2 Client — store + rebuilt page

**New store** `client/src/stores/profile.store.ts` (plain Zustand, **not** persisted — server is the source of truth):

- State: `profile: { avatar: string; bio: string; nickname: string } | null`, `isLoading`, `isSaving`, `error: string | null`.
- `fetchProfile()`: `GET /users/me/profile` via the shared `api` from `@/lib/axios`; store `profile`; also sync `user` into `auth.store` if the response's user differs (cheap freshness win).
- `saveProfile(changes: Partial<{ name; avatar; bio; nickname }>)`: send **only the changed fields**; on success update `profile` from the response and, if `name` changed, `useAuthStore.setState({ user: { ...user, name } })` so the rest of the app sees it immediately.

**Rebuild `Profile.tsx`:**

- On mount: `fetchProfile()`. Loading state: centered spinner (match Explore's `Loader2` pattern). Error state: inline destructive text + a "Retry" button that calls `fetchProfile()` again.
- Avatar block: if `profile.avatar`, render `<img>` with `applyTransform(avatar, "w_150,h_150,c_fill")` in the existing rounded circle; else keep the initial-letter fallback (first letter of nickname → name → "U", as the mock does). The camera button triggers a hidden `<input type="file" accept="image/*">`; validate client-side (image mime + ≤ 5 MB → inline error otherwise), upload with `uploadImage(file, { onProgress })` from `@/lib/cloudinary`, show the progress percent over the avatar while uploading, then `saveProfile({ avatar: secure_url })`. **Do not** register the avatar in the `/api/images` library — that library is for lesson assets.
- Form: Display name (from `auth.store` user.name initially, editable), Nickname (placeholder hint in Thai, e.g. `ให้เพื่อน ๆ เรียกว่าอะไร`), Bio textarea (keep 3 rows, add a subtle `x/500` counter), Email shown read-only (muted text under the avatar or a disabled input). Keep the Theme row and the Change-password button exactly where they are.
- Save button: disabled while `isSaving` or when nothing changed; spinner while saving; on success show an inline "Saved ✓" flash (little green text next to the button for ~2s — the pattern `ChangePassword.tsx` uses; **no toast**).
- 390 px check: the page is already `max-w-lg` single-column — verify nothing new overflows.

### 2.A.3 Tests

**Server — new `server/test/api/profile.test.ts`** (supertest against the exported `app`, `seedUser`/`seedLesson` from `test/helpers/seed.ts`):

1. `GET /api/users/me/profile` without token → 401 with the structured body (`forceRelogin: true`).
2. GET with token, no `UserData` row → defaults `{ avatar: "", bio: "", nickname: "" }` + correct `user` block; **no `UserData` row created** by the read.
3. PUT `{ bio, nickname }` → 200, row upserted; subsequent GET returns the values.
4. PUT partial (`{ bio }` only) → nickname untouched.
5. Validation 400s: empty body; `name: "  "`; `bio` 501 chars; `avatar: "http://evil.com/x.png"`.
6. Rename propagation: seed user + `seedLesson(user._id)` + a second lesson where the user is a collaborator (seed a second owner via `seedUser()`); PUT `{ name: "New Name" }`; assert owned lesson's `author_name` **and** the other lesson's `collaborator_names` both updated.

**Client — new `client/src/stores/__tests__/profile.store.test.ts`** (mock `@/lib/axios` with `vi.mock`):

1. `fetchProfile` populates state from the GET shape.
2. `saveProfile({ bio })` PUTs only `{ bio }`.
3. `saveProfile({ name })` syncs `auth.store` user.name on success.
4. API failure sets `error` and clears the busy flag.

**Client — new `client/src/pages/__tests__/Profile.test.tsx`** (light: render with mocked stores; assert fields populate from profile state and Save calls `saveProfile` with only changed fields).

### 2.A.4 Acceptance checklist

- [ ] Edit name/nickname/bio → Save → hard reload → values persist.
- [ ] Avatar upload works end-to-end (needs real Cloudinary env vars in `client/.env`); avatar renders after reload.
- [ ] Renaming yourself updates `author_name` on your lessons (check a Dashboard card / Explore card).
- [ ] Logged-out `/profile` still shows the friendly `RequireLogin` prompt (route untouched).
- [ ] `GET /api/chat/tutor` untouched; no new `protect` anywhere outside `/api/users/me/*`.
- [ ] Both suites green + lint clean.

### 2.A.5 Docs & commits

- `server/CLAUDE.md`: add the two routes to the `/api/users` table; update the "denormalized author fields" gotcha (renames now propagate via `refreshAuthorSnapshotsForUser`).
- `client/CLAUDE.md`: add `profile.store` to the stores table; note the Profile page is real now; add the new tests to the test-layout list.
- `ROADMAP.md`: mark 2.A shipped in §6 + flip `/profile` to ✅ in §9.
- Commits: one in `server/` (`feat: profile endpoints GET/PUT /api/users/me/profile + rename propagation`), one in `client/` (`feat: real profile page wired to /users/me/profile`).

---

## 3. Phase 2.B — Login-flow UX cleanup

**Goal:** forced logout explains itself and returns you where you were; validation errors are specific; no dead buttons. Size S–M · Repos: client (plus one optional 10-line server touch).

### 3.B.1 Scope (precise)

Touch together, as the roadmap prescribes: `Login.tsx`, `ProtectedRoute.tsx`, `PublicRoute.tsx`, `RequireLogin.tsx`, `lib/axios.ts`.

**1. Redirect-back-to-where-you-were.**

- `ProtectedRoute`: use `useLocation()`; redirect as `<Navigate to="/login" replace state={{ from: location.pathname + location.search }} />`.
- `RequireLogin`: the "Log in" `<Link>` gains the same `state={{ from: ... }}`.
- `lib/axios.ts`: extract a **pure exported helper** `buildForcedLoginUrl(pathname: string, search: string, message: string, code?: string): string` producing `/login?reason=<msg>&code=<code>&redirect=<pathname+search>`; the interceptor uses it (also pass `error.response.data.code` through). Export `isProtectedPath` too. Pure functions = testable without touching `window.location`.
- `Login.tsx`: on success, `navigate(target, { replace: true })` where `target` = `location.state?.from` → else `searchParams.get("redirect")` → else `/explore`. **Open-redirect guard:** only accept targets that start with `/` and not `//`.
- `PublicRoute`: when already logged in and a `redirect` query param is present (and passes the same guard), bounce there instead of `/explore`.

**2. Human `?reason=` messaging.** Render the forced-logout reason as an **info banner** (calm styling — `border-border bg-accent`-style box with an info icon), visually distinct from the red validation-error text. Map the `code` param to friendly Thai copy, falling back to the raw `reason` message:

| `code` | Banner copy |
| --- | --- |
| `TOKEN_EXPIRED` | `เซสชันหมดอายุแล้ว เข้าสู่ระบบอีกครั้งเพื่อไปต่อได้เลย` |
| `TOKEN_INVALID` / `TOKEN_MISSING` / `USER_NOT_FOUND` | `กรุณาเข้าสู่ระบบอีกครั้ง` |
| (none) | show `reason` text as-is |

Also stop writing the reason into `useAuthStore.setState({ error })` (today's hack) — banner state is local to the page.

**3. Specific validation, clean error lifecycle.**

- Client-side pre-submit checks with per-field inline messages: sign-up → name required, valid email shape, password ≥ 8 (matches the server's change-password floor); sign-in → both fields required.
- Server errors (`Invalid email or password`, `Email already in use`, 403 suspended) keep rendering in the red error slot.
- Clear all errors when toggling Sign in ↔ Create account, and clear a field's error when the user edits that field. (Today a stale error survives the toggle.)

**4. Dead-button removal.** Delete "Forgot password?" and the whole "or / Continue with Google" block (button + divider + inline SVG). No backend exists for either; honest UI wins. (Google OAuth can return as its own roadmap item when it's real.)

**5. Guard hygiene.** Remove `console.log(token)` from `RequireLogin.tsx` (it logs the JWT).

**6. Optional-but-recommended server touch** (10 lines, keeps the contract intact): `register` in `auth.controller.ts` currently accepts a 1-character password while change-password enforces ≥ 8. Add the same ≥ 8 check (400) + a case in `server/test/api/auth.test.ts`. Skip if you want 2.B to stay client-pure — but note the asymmetry in `server/CLAUDE.md` if you do.

### 3.B.2 Tests

- **New `client/src/lib/__tests__/axios.test.ts`:** `isProtectedPath` (protected prefix, nested path, public path) and `buildForcedLoginUrl` (encodes reason, includes code + redirect, composes pathname+search).
- **New `client/src/pages/__tests__/Login.test.tsx`** (MemoryRouter + mocked `auth.store`): reason+code query renders the Thai banner (not the red error); mode toggle clears errors; invalid email/short password blocks submit with the specific message; successful login with `state.from = "/history"` navigates to `/history`; malicious `redirect=//evil.com` falls back to `/explore`.
- **New `client/src/components/__tests__/guards.test.tsx`:** ProtectedRoute redirects to `/login` carrying `state.from`; RequireLogin renders children when token exists and the prompt (with `state`-carrying link) when not.

### 3.B.3 Acceptance checklist

- [ ] Corrupt the token in localStorage (`auth-storage`), visit `/dashboard` → land on `/login` with a calm Thai banner → log in → **you are back on `/dashboard`**.
- [ ] Same flow via a RequireLogin page (`/history`) using the prompt's Log in button.
- [ ] Register with a bad email / short password → specific field messages, nothing submitted.
- [ ] No dead buttons remain on `/login`; no JWT in the console anywhere.
- [ ] 390 px pass on `/login`.
- [ ] Owner signs off on the flow feel (this phase's roadmap gate).

### 3.B.4 Docs & commits

- `client/CLAUDE.md`: update the axios interceptor description (`reason`+`code`+`redirect`) and the roadmap-note under Routing.
- `ROADMAP.md`: mark 2.B in §6, flip `/login` in §9.
- Commit in `client/` (`feat: login flow UX — redirect-back, reason banner, specific validation`); if the optional server touch happened, commit in `server/` (`fix: enforce password minimum on register`).

---

## 4. Phase 2.C — Page polish sweep

**Goal:** close every remaining ❌/🔧 gap from ROADMAP §9 that isn't owned by another phase. Size S · Repos: client. This is the tier's closing sweep — run last.

### 4.C.1 Work items (checklist)

**Dead code**
- [ ] Delete `client/src/pages/Index.tsx`; remove its import from `App.tsx` (~line 17). While there, delete the commented-out Toaster/Tooltip imports + JSX in `App.tsx` (lines ~1-3, 45-47, 144) — dead weight.
- [ ] Verify no `console.log` of tokens/secrets remains in `src/` (2.B removed the RequireLogin one; sweep for others: `grep -r "console.log" client/src` and prune anything noisy).

**TopNav**
- [ ] Fix the duplicate Settings entry for logged-in users: restructure so Settings appears exactly once (e.g. `const navItems = token ? [guide, explore, history, create, profile, settings] : [guide, explore, settings]` — build from de-duplicated pieces rather than concatenating overlapping arrays). This also kills the React duplicate-`key` bug.

**Guide.tsx — content refresh (it predates the tutor)**
- [ ] Rewrite the `features` array to describe the app that actually exists. Keep the page's structure/styling; replace content. The eight sections become (keep English tone consistent with the page):
  1. **Visual Lesson Editor** — keep, current copy is fine.
  2. **Immersive Viewer** — keep.
  3. **Critical-Thinking Questions** — new: teachers embed choice / fill-in-the-blank / open-ended writing blocks; students get warm AI coaching on every answer, never a red ✗.
  4. **AI Tutor — น้องมันฝรั่ง** — new: ask anything about the lesson, anytime, in Thai; suggestion chips guide you when you don't know what to ask; **works without an account**.
  5. **Explore & Discover** — keep, trim the "bookmark favourites" promise to match reality (bookmarks are device-local).
  6. **Learning History** — keep.
  7. **Share & Collaborate** — keep.
  8. **~~Feedback & Comments~~** → replace (feature doesn't exist) with **Your AI remembers you** — logged-in students get continuity: the tutor remembers interests and growth areas across lessons.
- [ ] Bottom CTA: add real buttons — `<Link to="/explore">` (primary) and `<Link to="/create">` (outline, "I'm a teacher").

**Landing.tsx — copy check**
- [ ] Add the AI tutor to the story: replace the weakest of the three feature cards (or add a fourth) with an AI-tutor card ("A tutor in your pocket — ask anything, get warm coaching in Thai, no login needed"). Keep it honest with GR2: emphasize *no login needed*.
- [ ] Leave the fake showcase cards ("Ms. Chen"…) as-is for now but flag them in the owner-decision list (§5) — replacing them with real lesson cards needs owner-picked content.

**Explore.tsx — make bookmarks survive reload**
- [ ] New `client/src/stores/bookmark.store.ts`: Zustand + `persist` (name `bookmark-storage`), state `ids: string[]` + `toggle(id)` + `has(id)` (store an array, expose Set-like helpers — localStorage can't hold a `Set`). Works logged-out too, which fits GR2 (device-local, no account needed).
- [ ] Replace Explore's `useState<Set<string>>` with the store. The "Bookmarked" tab now shows the same lessons after a reload.

**Dashboard.tsx — stop silent data loss**
- [ ] Add the shadcn `alert-dialog` primitive (`npx shadcn@latest add alert-dialog` — CLI, per client conventions).
- [ ] Wrap lesson delete in a confirm dialog (title = lesson name, destructive confirm button, Thai-or-English copy matching nearby strings). Delete only fires on confirm.
- [ ] Make the delete button reachable on touch: `opacity-100 md:opacity-0 md:group-hover:opacity-100`.
- [ ] Surface fetch errors: read `error` from `content.store` (add it there if missing) and render the same inline destructive pattern Explore uses.

**Empty/loading/error pass**
- [ ] Explore / History: already good (verified) — just re-verify at 390 px after the bookmark change.
- [ ] Dashboard: covered by the error item above; empty + loading states already exist.

### 4.C.2 Tests

- **New `client/src/components/__tests__/TopNav.test.tsx`:** logged-in render contains exactly one "Settings" item; logged-out contains no auth-only items.
- **New `client/src/stores/__tests__/bookmark.store.test.ts`:** toggle adds/removes; state round-trips through the persisted storage shape.
- **New `client/src/pages/__tests__/Dashboard.test.tsx`:** clicking delete opens the dialog and does **not** call `deleteContent`; confirming calls it with the right id.

### 4.C.3 Acceptance checklist

- [ ] ROADMAP §9 audit table shows no ❌ left (Index deleted, Guide refreshed, Profile ✅ from 2.A, Login ✅ from 2.B).
- [ ] `npm run lint` clean; client suite green.
- [ ] 390 px pass on Guide, Landing, Explore, Dashboard.
- [ ] Bookmark a lesson → reload → still bookmarked.
- [ ] Deleting a lesson requires a confirm; works on a touch viewport.

### 4.C.4 Docs & commits

- `client/CLAUDE.md`: stores table (+`bookmark.store`), remove Index from any page references, note the delete-confirm.
- `ROADMAP.md`: mark 2.C in §6; update §9 (Guide → ✅, Index row removed, Landing note).
- Commit in `client/` (`feat: page polish sweep — guide refresh, persistent bookmarks, delete confirm, nav dedupe`).

---

## 5. Owner-decision items (surface these, don't decide them)

Collect these in the final report to the owner — none block the phases above:

1. **Brand name:** the UI says **"Intuita"** (TopNav, Landing, Guide) while the project/repo/API say **"Hot Potato"**. Pick one for user-facing copy. Tier 2 keeps "Intuita" as-is everywhere it already appears.
2. **Landing showcase cards** are fictional (fake lessons, fake teachers). Replace with real public lessons? Needs owner-picked content.
3. **Landing language:** the landing page is fully English while the product is Thai-first. Worth a Thai hero variant? (Owner call — copywriting, not engineering.)
4. **Forgot-password:** removed as a dead button in 2.B. If the owner wants it for real, it's a new small phase (email service — new infra, new env vars).
5. **Security findings for Tier 3.A** (found in passing while speccing 2.A — do **not** fix in Tier 2, but don't lose them): (a) `GET /api/users` returns full user docs **including bcrypt password hashes** (`User.find()` with no `.select`, and the schema has no `select: false` on `password`) to any logged-in user; (b) `register` in `auth.controller.ts` passes `role` from the request body straight into `User.create` — anyone can self-register as `admin`. Both slot into the existing Tier 3.A "clean `/api/users` scaffolding / role story" items; they raise its urgency.

---

## 6. Tier-2 definition of done

1. All three phase acceptance checklists fully checked.
2. `cd server && npm test` and `cd client && npm test` green; `cd client && npm run lint` clean.
3. Manual smoke: both halves running, test lesson `/view/69e39d0b60d467bd515a4945` opens **logged-in and incognito** (Golden Rule 2 executable check — Tier 2 must not have touched anonymous access).
4. `ROADMAP.md` §6 shows 2.A/2.B/2.C shipped with dates; §9 has no ❌ rows.
5. Both `CLAUDE.md`s updated per phase notes.
6. Owner-decision list (§5 above) delivered to the owner.
