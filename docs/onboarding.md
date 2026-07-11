# onboarding.md — from clone to a working AI reply

> Updated 2026-07-11. The 20-minute "get it running and prove it works" path for a new contributor. The READMEs have the exhaustive env tables ([`../server/README.md`](../server/README.md), [`../client/README.md`](../client/README.md)); this is the *guided first run* with the gotchas called out in order.

You need **two terminals** (API + web app) and four accounts (MongoDB, Gemini, Cloudinary; Google OAuth optional). If you only want to *read* the code, skip to §5.

---

## 1. Get the four credentials first (5 min)

Do this before touching code — missing keys are the #1 first-run blocker.

| Need | Where | Note |
| --- | --- | --- |
| MongoDB connection string | a free [Atlas](https://www.mongodb.com/atlas) cluster (or local `mongod`) | the server **exits on boot** if it can't connect |
| Gemini API key | [Google AI Studio](https://aistudio.google.com/) (free) | without it, chat endpoints return 500 |
| Cloudinary cloud name + unsigned preset | a free Cloudinary account | image upload only; the rest works without it |
| Google OAuth Web client id | [Cloud Console](https://console.cloud.google.com/) | **optional** — unset just hides the Google button |

---

## 2. Server (terminal 1) → http://localhost:5000

```bash
cd server
npm install
# create server/.env  (see below)
npm run dev        # nodemon + ts-node, auto-restart
```

Minimum `server/.env`:

```env
PORT=5000
MONGO_URI=mongodb+srv://<user>:<pass>@<cluster>/<db>
JWT_SECRET=any-long-random-string
JWT_EXPIRES_IN=7d
GEMINI_API_KEY=your-gemini-key
CLOUDINARY_CLOUD_NAME=your-cloud-name
CLOUDINARY_UPLOAD_PRESET=your-unsigned-preset
# optional: GOOGLE_CLIENT_ID, CORS_ORIGINS=http://localhost:5173, AI_OUTPUT_LANGUAGE=thai
```

**Verify:** `GET http://localhost:5000/` returns a hello string; `GET http://localhost:5000/api/status/all` returns a health JSON. (Leaving `CORS_ORIGINS` unset is fine locally — it fails open with a boot warning.)

---

## 3. Client (terminal 2) → http://localhost:5173

```bash
cd client
npm install
# create client/.env  (see below)
npm run dev        # vite (falls back to :5174 if busy)
```

`client/.env` (only `VITE_`-prefixed vars reach the browser):

```env
VITE_API_URL=http://localhost:5000
VITE_CLOUDINARY_CLOUD_NAME=your-cloud-name
VITE_CLOUDINARY_UPLOAD_PRESET=your-unsigned-preset
# optional: VITE_GOOGLE_CLIENT_ID=... (same id as server GOOGLE_CLIENT_ID; register http://localhost:5173 as an origin)
```

**Gotcha:** `VITE_API_URL` has **no trailing slash** (the client appends `/api`).

---

## 4. Prove it works — the smoke test (test BOTH auth states)

Golden Rule 2 means "logged out" is a first-class path — always check both:

1. Open `http://localhost:5173`. Register an account (or stay anonymous).
2. Create a lesson from the dashboard, add a Question block, publish it (`public`).
3. Open it in the viewer, answer a question → you should get streaming AI feedback + suggestion chips. **First reply may be slow** (Gemini + any cold state) — that's normal.
4. Open the same lesson in an **incognito window** (logged out) → reading + AI must still work (just no persistence).

If step 3 fails: check `GEMINI_API_KEY` and the server terminal for errors. If the browser can't reach the API: check `VITE_API_URL` and CORS.

---

## 5. Read the code fast (don't brute-force 1,700 files)

1. [`docs/ARCHITECTURE.md`](ARCHITECTURE.md) — the mental model (5 load-bearing ideas). **Start here.**
2. [`docs/GLOSSARY.md`](GLOSSARY.md) — the vocabulary.
3. The map for your side: [`docs/ui-structure.md`](ui-structure.md) + [`docs/ui-state.md`](ui-state.md) (frontend) or [`docs/data-model.md`](data-model.md) (backend).
4. The flow you're touching: [`docs/data-flow.md`](data-flow.md) / [`../asking-flow.md`](../asking-flow.md).
5. Jump to files with the knowledge graph: `graphify query "..."` (see [`../AGENT.md`](../AGENT.md)).
6. The rules you must not break: the half's `CLAUDE.md` + [`docs/ideas.md`](ideas.md).

---

## 6. Before you commit anything

- **Tests green in every half you touched:** `cd server && npm test` and `cd client && npm test` (both offline — no key/network/DB needed). See [`testing.md`](testing.md).
- **Lint (client):** `npm run lint`.
- **⚠️ Git reality:** `client/` and `server/` are **separate repos** — run git *inside* the half, never from the workspace root (a stray home-dir repo will surface unrelated files). See root [`CLAUDE.md`](../CLAUDE.md).
- **Contributing conventions:** [`../server/CONTRIBUTING.md`](../server/CONTRIBUTING.md) / [`../client/CONTRIBUTING.md`](../client/CONTRIBUTING.md).
