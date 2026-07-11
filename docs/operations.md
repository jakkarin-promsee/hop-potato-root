# operations.md — deploy & run in production

> Updated 2026-07-11 · the repeatable runbook behind the READMEs' deploy blurbs. The one-time launch *plan* is [`../ROADMAP.md`](../ROADMAP.md) §Tier 5; this is the *repeatable* version (deploys, incidents, routine ops). ⚠️ As of this date **nothing is deployed** — both repos are ahead of origin by design (ADR-015). Production still runs the pre-rework app until the launch pass.

Hot Potato runs on three free tiers: **client → Vercel**, **server → Render**, **data → MongoDB Atlas**, plus **Cloudinary** (images) and **Google Gemini** (AI). Everything below assumes that topology.

---

## 1. The environment matrix — which var lives where

Secrets are set in **three dashboards**, not in git. Get this right and 90% of deploy pain disappears.

| Variable | Render (server) | Vercel (client) | Google Cloud Console |
| --- | --- | --- | --- |
| `MONGO_URI`, `JWT_SECRET`, `JWT_EXPIRES_IN` | ✅ | | |
| `GEMINI_API_KEY` | ✅ | | |
| `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_UPLOAD_PRESET` | ✅ | ✅ (`VITE_` prefixed) | |
| `CORS_ORIGINS` | ✅ (Vercel prod URL + localhost) | | |
| `GOOGLE_CLIENT_ID` / `VITE_GOOGLE_CLIENT_ID` | ✅ | ✅ | ✅ authorized origins |
| `VITE_API_URL` | | ✅ (→ Render URL) | |
| AI tuning (`AI_*_MODEL`, `AI_RATELIMIT_*`, `AI_MEMORY_EVERY_N_TURNS`, `AI_OUTPUT_LANGUAGE`) | ✅ optional | | |

Full descriptions + defaults: [`../server/CLAUDE.md`](../server/CLAUDE.md) §"Environment variables" and [`../server/README.md`](../server/README.md). **`AI_DEBUG_PROMPTS` must never be set in production** (it logs student text).

**Gotchas that bite:** `VITE_API_URL` has no trailing slash (the client appends `/api`); the Google button silently fails on origins not registered in the Cloud Console; `CORS_ORIGINS` unset = fail-open (set it before any wider release).

---

## 2. Deploy (how a release actually happens)

Both platforms **auto-deploy from a git push** — there is no manual deploy step, and CI (ROADMAP Tier 4) gates the push once it exists.

| Half | Build | Start | Notes |
| --- | --- | --- | --- |
| **Server (Render)** | `npm install && npm run build` | `npm start` (= `node dist/index.js`) | ⚠️ confirm the start command is `dist/index.js`, **not** `dist/app.js` (`app.ts` doesn't `listen` — ADR-017) |
| **Client (Vercel)** | `npm run build` | static `dist/` | `vercel.json` rewrites all paths → `index.html` (SPA); new routes need no config |

The first-ever launch is a **checklist, not routine** — push both repos, set Render/Vercel env, register the Google origin, `npm run promote -- <owner-email>` for admin, then smoke both auth states. Execute it from [`../ROADMAP.md`](../ROADMAP.md) §Tier 5 the first time; after that, a release is just "push → auto-deploy → smoke."

---

## 3. Post-deploy smoke (the executable definition of "up")

Golden Rule 2 makes this a *requirement*, not a nicety — **test both auth states**:

1. Open the test lesson `/view/69e39d0b60d467bd515a4945` **logged-in** and **incognito**.
2. Full tutor conversation with streaming; question-card feedback; personality switch.
3. Google sign-in on the prod domain (new + linked account).
4. `/status` shows all-green including the AI-health + recent-errors cards.
5. First request may be a **cold start** (§4) — slow ≠ broken.

---

## 4. The free-tier physics (know these before you diagnose)

- **Render cold starts.** The server sleeps when idle; the first request after a pause takes several seconds *plus* AI latency. The client shows loading states everywhere (`useColdStartHint`). This is the #1 "is it down?" false alarm.
- **Observability resets on restart.** The `/status/all` error buffer + AI-health snapshot are in-memory (ADR-006) — a restart clears them; the page says "since last restart." Don't expect historical metrics.
- **MongoDB Atlas free tier** has connection/storage limits; `connectDB` retries then `process.exit(1)` on exhaustion (so `index.ts`'s boot fails loudly rather than half-running).

---

## 5. Reading the health page (`/status` + `GET /api/status/all`)

Public, sanitized, polled every 30 s. Three original cards (server / database / env presence) + two from Tier 1 (**AI Tutor** health, **Recent errors** — route+code, no messages). If something's wrong:

- **AI degraded** → check `GEMINI_API_KEY` on Render + the Gemini status page; `generateWithRetry` already absorbs transient blips (ADR-018).
- **Errors climbing** → the codes/routes point at the failing endpoint; full names on admin-only `GET /status/`.
- **DB disconnected** → Atlas cluster asleep/over-limit, or `MONGO_URI` rotated.

---

## 6. Routine ops

- **Promote an admin:** `cd server && npm run promote -- <email>` (against the target DB).
- **Check model access:** `npm run check-models` (probes which Gemini models the key can reach — how the `gemini-3.5-flash` default was chosen; log tone-affecting changes in `prompt-notes.md`).
- **Rollback:** both hosts deploy from git — revert the offending commit and push, or redeploy a previous build from the Render/Vercel dashboard. There is no DB migration system, so a rollback is code-only (schema is Mongoose-flexible).
- **Rotate a secret:** update it in the dashboard → redeploy (Render restarts; Vercel needs a rebuild for `VITE_` vars since they're build-time inlined).
- **After significant changes:** rebuild the knowledge graph (`/graphify --update`) and bump the freshness dates in `docs/`.

---

## 7. Incident quick-reference

| Symptom | Likely cause | First check |
| --- | --- | --- |
| All requests fail from the browser, curl works | CORS | `CORS_ORIGINS` includes the exact Vercel origin |
| Everyone logged out unexpectedly | `JWT_SECRET` changed | don't rotate `JWT_SECRET` casually — it invalidates all tokens |
| Google button does nothing | origin not registered | add the prod URL to the OAuth client's authorized origins |
| AI returns 500 "unavailable" | missing/invalid `GEMINI_API_KEY` | Render env + `/status/all` `ai` block |
| First load slow after idle | Render cold start | wait ~10 s; it's expected |
| Client shows stale UI after deploy | old chunk hash | `main.tsx` auto-reloads once on `vite:preloadError`; hard-refresh otherwise |
