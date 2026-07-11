# 🥔 Hot Potato

**An AI-powered learning platform that lets any teacher — even non-technical ones — author rich, critical-thinking lessons, and lets students learn by freely questioning the material with an AI tutor.**

Hot Potato is built for learners who are far from good schools and good internet. A teacher writes a lesson with embedded critical-thinking questions; a student reads it, answers the questions, and can ask the built-in AI anything about the content. The AI grades open-ended answers gently (like a friend, not a strict examiner) and helps the student think, not just memorize.

> **Status:** Phase 1 complete and working end-to-end. This phase focuses on polishing the experience — the AI feedback/questioning quality and the overall UX (sign-in flow, etc.).

---

## What it does

| For teachers (creators) | For students (learners) |
| --- | --- |
| Write lessons in a rich editor (text, images, tables, math formulas, drawings) | Read lessons and answer embedded questions |
| Embed critical-thinking questions: multiple-choice, written, fill-in-the-blank | Get warm, friend-tone AI feedback on answers |
| Add an AI tutor block so students can freely ask about the lesson | Ask the AI tutor anything, grounded in the lesson |
| Control access: public / link-only / private | Pick up where they left off (learning history) |
| Manage a personal image library (Cloudinary) | Works without an account for public lessons |

The AI is **Google Gemini** (`gemini-2.5-flash-lite`). It answers in **Thai by default** and is instructed to coach, never to shame.

---

## Architecture at a glance

Hot Potato is a classic two-tier web app:

- **`client/`** — the web app the teacher and student use. React 19 + TypeScript + Vite, styled with Tailwind v4 + shadcn/ui. The lesson editor is built on TipTap with custom question/canvas/formula blocks. Deployed to **Vercel**.
- **`server/`** — the REST API. Express 5 + TypeScript on MongoDB (Mongoose), with JWT auth and the Gemini integration. Deployed to **Render**.

The client talks to the server over a REST API (`/api/...`) using `axios`, sending a JWT `Bearer` token for authenticated calls.

```
Browser ──HTTPS──▶ client (Vercel)  ──REST /api──▶ server (Render) ──▶ MongoDB Atlas
                                                          ├──▶ Google Gemini (AI)
                                                          └──▶ Cloudinary (images)
```

Each half has its own deep documentation:

- 📘 [`client/README.md`](client/README.md) — run & deploy the web app · [`client/CLAUDE.md`](client/CLAUDE.md) — full client architecture
- 📗 [`server/README.md`](server/README.md) — run & deploy the API · [`server/CLAUDE.md`](server/CLAUDE.md) — full server architecture
- 🧠 [`docs/`](docs/README.md) — **system intuition**: how flows work and *why* (architecture, data-flow, UI structure/state, data model, glossary, design decisions, onboarding, testing, security, operations) — start at [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md)
- 🤖 [`CLAUDE.md`](CLAUDE.md) — how AI agents (and you) should work across both halves

---

## Quick start (local)

You need **two terminals** — one for the API, one for the web app.

```bash
# 1) API  (http://localhost:5000)
cd server
npm install
# create server/.env  (see server/README.md for the full list)
npm run dev

# 2) Web app  (http://localhost:5173)
cd client
npm install
# create client/.env  (see client/README.md)
npm run dev
```

You also need a **MongoDB** connection string, a **Gemini API key**, and a **Cloudinary** account — see [`server/README.md`](server/README.md) for the exact env vars.

---

## Tech stack

| Layer | Tech |
| --- | --- |
| Frontend | React 19, TypeScript, Vite 7, Tailwind CSS v4, shadcn/ui, Zustand, TanStack Query, React Router 7 |
| Editor | TipTap 3 (rich text), Fabric.js 7 (canvas), KaTeX (math) |
| Backend | Node.js, Express 5, TypeScript, Mongoose 9 (MongoDB) |
| Auth | JWT (`jsonwebtoken`) + `bcryptjs` |
| AI | Google Gemini (`@google/generative-ai`) |
| Media | Cloudinary |
| Hosting | Client → Vercel · Server → Render · DB → MongoDB Atlas |

---

## Repository layout

> **Important:** three git repos, one workspace folder. `Hot-Potato/` (this repo) holds **workspace docs**, the knowledge graph, and agent rules. `client/` and `server/` are **separate repos** for the deployable halves. Run `git` in the folder you're actually changing — see [`CLAUDE.md`](CLAUDE.md) and [`AGENT.md`](AGENT.md) §6.

```
Hot-Potato/            # workspace meta repo (docs, graphify, ROADMAP archive)
├── client/            # React web app (own git repo) → Vercel
├── server/            # Express API (own git repo) → Render
├── docs/              # system intuition (architecture, data-flow, …)
├── graphify-out/      # prebuilt codebase knowledge graph (query from here)
├── archive/           # superseded roadmaps & plans (history only)
├── .cursor/           # Cursor rules + Playwright MCP config
├── AGENT.md           # agent front door
├── CLAUDE.md          # cross-cutting guide for AI agents
├── README.md          # ← you are here
```

---

## Roadmap (phase 2)

- **AI feedback & questioning** — sharpen Gemini prompts, ground answers more tightly in lesson content, improve follow-up conversation quality and evaluation accuracy.
- **Overall UX polish** — clean up the sign-in/login flow and smooth out the authoring and learning experience.

---

## License

Not yet specified. (Personal / educational project.)
