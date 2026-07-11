# CLAUDE.md — Hot Potato (root)

Guidance for AI agents (and humans) working anywhere in this project. Read this first, then the relevant half's `CLAUDE.md`.

## What this project is

Hot Potato is an **AI-powered learning platform**. Teachers (even non-technical ones) author rich lessons with embedded **critical-thinking questions**; students read the lessons, answer the questions, and can **freely ask an AI tutor** about the content. The AI (Google Gemini) gives warm, coaching-style feedback. The target audience is students far from good schools and reliable internet.

**The product core is done and owner-approved (2026-07-10).** The old universal roadmap's Tiers 0–3 all shipped and passed the owner's tone/behavior gates: every client flow speaks to the unified `POST /api/chat/tutor` (SSE streaming, suggestions, markdown, phone-first chat), teachers get per-lesson `agent_settings`, students get 6 selectable tutor personalities, all 15 pages are real, and the server is hardened (CORS, admin role, auth rate limits). Context flow is documented in [`asking-flow.md`](asking-flow.md). **⚠️ Nothing is deployed yet** — both repos are ahead of origin (intentional; one deploy pass at the end). Current focus: the **launch-readiness roadmap** — SEO/OG → observability → bundle split → settings/profile completeness → CI → launch:

- [`ROADMAP.md`](ROADMAP.md) — the **launch-readiness roadmap v3** (2026-07-10): vision, the **two Golden Rules**, owner decisions D1–D7, and all remaining work as Tiers 0 → 6 with phases.
- `archive/` — superseded roadmaps: `arc1/` (2026-07-09 tutor-only) and `arc2/` (2026-07-10 universal, its Tiers 0–3 shipped, + `plan/` execution specs). History only; **do not execute from these.**

The two Golden Rules (non-negotiable, they override "best practice" instincts):

1. **Never ration tokens for real students** — no per-student quotas or limits. Rate limiting exists only to stop bots, with generous env-tunable thresholds.
2. **Anonymous users keep full access** — reading content and using the AI never requires login; logged-out users just get no persistence (history/memory). Never add `protect` to AI or public-content read paths.

## The two halves

| | `client/` | `server/` |
| --- | --- | --- |
| Role | Web app (teacher + student UI) | REST API + AI + persistence |
| Stack | React 19 + TS + Vite + Tailwind v4 + shadcn/ui | Express 5 + TS + Mongoose (MongoDB) |
| Deploy | Vercel | Render |
| Deep docs | [`client/CLAUDE.md`](client/CLAUDE.md) | [`server/CLAUDE.md`](server/CLAUDE.md) |

They communicate over a REST API. The client uses an `axios` instance (`client/src/lib/axios.ts`) with `baseURL = VITE_API_URL` and attaches a JWT `Bearer` token to every request. The contract: **routes are defined in `server/src/routes/*`, consumed from `client/src/stores/*` and `client/src/lib`/component API helpers.** When you change an endpoint on one side, update the other.

## ⚠️ Git reality — read before any git command

- **Three repos:** `Hot-Potato/` is the **workspace meta repo** (docs, `graphify-out/`, `archive/`, `.cursor/`). `client/` and `server/` are **separate product repos** (gitignored in the meta repo).
- The server's remote is `github.com/Jakkarin-Promsee/hot-potato-server`.
- Run `git` **in the repo you're changing:** workspace docs → `Hot-Potato/`; app code → `client/` or `server/`. Never run git from `C:/Users/BTCOM` (home) — a stray `git init` there surfaces unrelated files. If `git status` shows foreign projects, you're in the wrong folder.

## What to ignore

These top-level folders are **reference material and scratch, not part of the product**. Do not edit them, build them, or treat them as source of truth:

- `example_copy/` — an older copy of the app (`potato-page`).
- `example_project/` — third-party reference UI projects (`creative-canvas-suite`, `luxe-write`, `react-social-canvas`).
- `test/`, `Ptest/` — scratch files and git-branch experiments.

## The four working folders (the only things that matter)

Everything an agent should touch lives in these four places:

| Folder / file | What it is |
| --- | --- |
| `client/` | The web app (its own git repo). |
| `server/` | The REST API + AI (its own git repo). |
| `graphify-out/` | A prebuilt **knowledge graph** of the whole codebase (1231 nodes, 97 communities; may lag the Tier 1–3 file changes — rebuild scheduled at launch). Query it to understand the code fast — see [`AGENT.md`](AGENT.md). |
| [`ROADMAP.md`](ROADMAP.md) | The launch-readiness roadmap (v3) — all remaining work as Tiers 0 → 6. (Old roadmaps live in `archive/arc1` and `archive/arc2`.) |
| [`ROADMAP-guide.md`](ROADMAP-guide.md) | The guide/tutorial track (2026-07-11, expands old Tier 6.B): rebuild `/guide` into a hub + two showcase walkthroughs (`/guide/learning` for students, `/guide/creating` for teachers), Tiers G0 → G6, Playwright-generated screenshots. Execution spec + scene scripts: [`plan/guide.md`](plan/guide.md). |

Start any session by reading [`AGENT.md`](AGENT.md) — it is the front door that points at the graph, these docs, and the roadmap.

## Common commands

Always `cd` into the half you're working on first.

```bash
# Server (API) — http://localhost:5000
cd server && npm install && npm run dev     # nodemon → src/index.ts
cd server && npm run build                  # tsc → dist/
cd server && npm start                      # node dist/index.js
cd server && npm test                       # vitest (offline, in-memory Mongo)
cd server && npm run test:ai                # live Gemini smoke (RUN_AI_TESTS=true + real key)

# Client (web) — http://localhost:5173
cd client && npm install && npm run dev     # vite
cd client && npm run build                  # vite build → dist/
cd client && npm run lint                   # eslint
cd client && npm test                       # vitest + happy-dom
```

**Tests:** run `cd server && npm test` and `cd client && npm test` before manual verification. Server tests use in-memory MongoDB and a mocked Gemini — no network, no real DB, no API key. Live Gemini smoke tests: `cd server && npm run test:ai` with `RUN_AI_TESTS=true` and a real `GEMINI_API_KEY`. Test layout is documented in [`server/CLAUDE.md`](server/CLAUDE.md) and [`client/CLAUDE.md`](client/CLAUDE.md).

**Working agreement for implementing agents:** every phase ships tests for its own acceptance criteria; a phase is not done until `npm test` is green in every half you touched.

## Conventions across both halves

- **TypeScript everywhere.** Match the existing style of the file you're editing (it varies a little between the two halves).
- **AI output language is Thai by default** (configurable server-side via `AI_OUTPUT_LANGUAGE`). UI copy is mixed English/Thai. The owner is a Thai speaker; user-facing strings may be Thai.
- **Secrets live in `.env`** (git-ignored) in each half. Never commit secrets or print their values. See each half's docs for the variable list.
- **Don't duplicate the data contract.** The data model is owned by `server/src/models/*`. The simplified sketch in `server/src/models/schema.md` is older than the code — trust the models.

## Where the important things live

| Topic | Look here |
| --- | --- |
| AI prompts & Gemini calls | `server/src/services/tutor/` (persona, parse, memory, retry, personality, models) + `server/src/controllers/tutor.controller.ts` (the only AI endpoint — legacy `chat.controller.ts` deleted in Tier 0) |
| Auth (JWT, roles, guards) | `server/src/middlewares/auth.middleware.ts` |
| Data model | `server/src/models/*.model.ts` |
| API routes (full surface) | `server/src/routes/*.routes.ts` + [`server/CLAUDE.md`](server/CLAUDE.md) |
| The lesson editor (TipTap + Fabric) | `client/src/components/editor/` + `client/src/components/README.md` |
| Question/AI blocks | `client/src/components/editor/extensions/Question*` |
| Client → AI calls | `client/src/components/editor/extensions/tutorApi.ts` (`callTutor` / `callTutorStream` — the single bridge) |
| Client routing & page map | `client/src/App.tsx` + [`client/CLAUDE.md`](client/CLAUDE.md) |
| Global client state | `client/src/stores/*.store.ts` (Zustand) |
| Codebase knowledge graph | `graphify-out/` (query it — see [`AGENT.md`](AGENT.md)) |
| The plan for all new work | [`ROADMAP.md`](ROADMAP.md) (launch-readiness v3, Tiers 0 → 6) |
| Agent tooling (plugins, Playwright MCP) | [`AGENT.md`](AGENT.md) §7 — `frontend-design` plugin + browser automation; user-scoped, **restart to load** |

## Working style for this repo

- This is the owner's **first full-stack project** and is intentionally getting better documentation. When you add or change a non-obvious mechanism, leave a short note in the relevant `CLAUDE.md`.
- Prefer **prose and tables over ASCII diagrams** in docs (owner preference).
- The server is on Render's free tier and **cold-starts**; AI responses can take a few seconds. Account for latency in client UX (loading states), not by assuming the server is down.
