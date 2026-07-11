# AGENT.md — Start here

**The front door for any AI agent (Cursor, Claude, Copilot, …) working on Hot Potato.** Read this first. It tells you where to look, how to understand the code fast, and the rules you must not break.

---

## 1. What Hot Potato is (30 seconds)

An **AI-powered learning platform**. Teachers author rich lessons with embedded **critical-thinking questions**; students read them, answer, and **freely chat with an AI tutor** (Google Gemini, warm Thai coaching tone). Audience: Thai students far from good schools and reliable internet. Two halves — a React web app (`client/`) and an Express + MongoDB API (`server/`) — talking over REST.

**Current mission:** the tutor product is **done and owner-approved** (old roadmap Tiers 0–3 shipped + all tone gates passed, 2026-07-10): unified `POST /api/chat/tutor` with SSE streaming, teacher `agent_settings`, 6 student personalities, phone-first markdown chat, all pages real, server hardened. Now the **launch-readiness roadmap**: SEO/OG tags → error monitoring + Status page v2 → bundle splitting → settings/profile completeness → CI → **one deploy pass** (nothing is pushed/deployed yet — intentional). All work is planned in the roadmap (§4).

---

## 2. Understand the code FAST — use the knowledge graph 🗺️

`graphify-out/` holds a prebuilt **knowledge graph of the entire codebase** — **1231 nodes, 2034 edges, 97 communities** across `client/`, `server/`, and the docs. ⚠️ It predates some Tier 1–3 file changes (e.g. `personality.ts`, `profile.store.ts`) — a rebuild is scheduled in the launch tier; trust it for structure, verify recent files directly. Do not blindly grep a 159-file codebase when the graph can point you straight at the right files.

**Query it in natural language** (the graph already exists — this reads it, it does not rebuild):

```bash
graphify query "how does the AI feedback flow work end to end"
graphify query "what calls the chat controller"
graphify path "QuestionChoiceView" "chat.controller.ts"     # shortest path between two concepts
graphify explain "questionAgentContext"                       # plain-language node explanation
```

Humans can also open `graphify-out/graph.html` in a browser to explore visually, or skim `graphify-out/GRAPH_REPORT.md` for the community map (e.g. "AI Chat Controller (Gemini)", "Editor Extensions & Question Blocks", "Route Guards & Auth Flow").

**When to rebuild the graph:** only after large structural changes, via the `/graphify` skill (`graphify <path> --update` for incremental). Day-to-day, just query it.

**Workflow:** query the graph to locate the relevant files → open them → read the matching `CLAUDE.md` for the rules that govern them → then edit.

---

## 3. The documentation map — read the right `CLAUDE.md`

Context is layered. Always read root first, then the half you're touching, then any deep-dive.

| Doc | Read it when |
| --- | --- |
| [`CLAUDE.md`](CLAUDE.md) (root) | **Always first.** Cross-cutting context, the two halves, the git quirk, what to ignore. |
| [`docs/`](docs/README.md) — **the intuition layer** | Understanding *how a flow works and why* without reading 1,700 graph nodes: [`ARCHITECTURE`](docs/ARCHITECTURE.md) (mental model + load-bearing ideas), [`data-flow`](docs/data-flow.md), [`ui-structure`](docs/ui-structure.md)/[`ui-state`](docs/ui-state.md), [`data-model`](docs/data-model.md), [`GLOSSARY`](docs/GLOSSARY.md), [`ideas`](docs/ideas.md) (ADRs), plus onboarding/testing/security/operations. Start at [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md). |
| [`server/CLAUDE.md`](server/CLAUDE.md) | Touching the API, auth, data model, or AI. Has the full route table + gotchas. |
| [`client/CLAUDE.md`](client/CLAUDE.md) | Touching the web app, routing, stores, or the editor. |
| [`client/src/components/README.md`](client/src/components/README.md) · [`client/docs/editor-internals.md`](client/docs/editor-internals.md) | Touching the TipTap/Fabric lesson editor internals. |
| [`server/docs/ai-internals.md`](server/docs/ai-internals.md) | Touching the tutor engine (`services/tutor/*`) — pairs with [`asking-flow.md`](asking-flow.md). |
| [`server/src/models/*.model.ts`](server/src/models/) | The data contract's **source of truth** (`schema.md` is deprecated — see [`docs/data-model.md`](docs/data-model.md)). |

When you add or change a non-obvious mechanism, **leave a short note in the relevant `CLAUDE.md`** — this project is intentionally over-documented because it's the owner's first full-stack app.

---

## 4. The plan — where all new work comes from

New work is not improvised. It is specced:

| Doc | What it gives you |
| --- | --- |
| [`ROADMAP.md`](ROADMAP.md) | The **launch-readiness roadmap v3** (2026-07-10): vision, the **two Golden Rules**, owner decisions D1–D7, and all remaining work as **Tiers 0 → 6** with phases, acceptance criteria, and sizing. Execute phases from this file. |
| `archive/arc1/` | The 2026-07-09 tutor-only roadmap + detailed spec (its Phases -1 → 3 shipped). History only. |
| `archive/arc2/` | The 2026-07-10 universal roadmap + `plan/` execution specs (its Tiers 0–3 shipped and owner-gated). History only — **do not execute from archive.** |

Tier order (v3): **0** SEO & link previews → **1** observability (error ring buffer + Status page v2) → **2** performance (bundle splitting, fonts) → **3** settings & profile completeness (font-size control, Help popup, tutor-memory UI) → **4** CI (optional) → **5** launch (push + deploy + prod smoke, one shot) → **6** deferred backlog (email/password reset, guide refresh, dopamine layer).

---

## 5. 🥇 The Golden Rules (non-negotiable — they override "best practice")

If a change conflicts with a Golden Rule, the Golden Rule wins.

1. **Never ration tokens for real students.** No per-student quotas, no "daily limit" UI, no message caps, no paywall logic. Students (and the owner 😂) can chat with the AI as much as they want. Rate limiting exists **only** to stop bots/non-human abuse, with limits set so high no human ever hits them.

2. **Anonymous users keep FULL access.** No-login users can read all public content **and** use the AI fully (ask, answer, get feedback). The *only* difference when logged out: nothing is persisted (no history, no memory). **Never add `protect` to an AI or public-content read path.** This already works today (`optionalAuth` + open chat routes) — preserve it in every change.

Design principles behind the tutor: Thai-first; **critical thinking > correctness** (admire the idea first, never say "wrong", give dopamine); students are low-tech AI newbies, so the AI *navigates* them ("how to think with AI, not follow AI"); teacher-authored questions/answers/guidelines drive the agent.

---

## 6. ⚠️ Git reality — read before any git command

- **Three repos, one workspace:** `Hot-Potato/` (this folder) is the **workspace meta repo** — docs, `graphify-out/`, `archive/`, `.cursor/` rules. `client/` and `server/` are **separate product repos** (Vercel + Render).
- Run `git` **in the repo you're changing:** workspace docs → `Hot-Potato/`; app code → `client/` or `server/`. Never run git from `C:/Users/BTCOM` (home) — a stray `git init` there surfaces unrelated projects. If `git status` shows foreign files, **you're in the wrong folder** — stop.
- `client/` and `server/` are **gitignored** in the meta repo (nested `.git` folders). Product commits still happen inside each half.

---

## 7. 🧰 Agent tooling — plugins & MCP servers

Two extras are wired into the agent dev environment (added 2026-07-10). Both are **user-scoped** — they live in the developer's Claude Code config (`~/.claude.json`, `~/.claude/`), **not** in either sub-repo, so they sidestep the root git quirk in §6 and travel with the developer, not the repo.

| Tool | What it is | Install (already done) |
| --- | --- | --- |
| **frontend-design** — Claude Code plugin | UI/design skills + slash commands from the official marketplace (`claude-plugins-official`). Pairs with `client/` React/Tailwind work. | `claude plugin install frontend-design@claude-plugins-official -s user` |
| **playwright** — MCP server | Real browser automation: 24 `browser_*` tools (navigate, snapshot, click, type, screenshot, console, network). Drives Chromium against a running dev server. | `claude mcp add playwright -s user -- npx @playwright/mcp@latest` |

**⚠️ Restart to load them.** Plugin skills/commands and the `mcp__playwright__*` tools bind at **session start**. After install — or after setting up a new machine — quit and relaunch Claude Code (or Cursor) before they appear. Verify anytime with `claude plugin list` and `claude mcp list` (Playwright should read `✔ Connected`).

**Drive the client with Playwright:**

1. `cd client && npm run dev` (Vite on :5173, falls back to :5174 if the port is busy).
2. In a fresh session, ask the agent to navigate to that URL, then `browser_snapshot` to read the rendered accessibility tree, `browser_take_screenshot` for a visual, or click/type to exercise flows. Headed Chromium by default; re-add the server with a trailing `--headless` for background runs.

Smoke-tested 2026-07-10: headless Chromium loaded the client landing page and read back the title "Hot Potato" and hero copy — the full MCP stack works end to end.

**Cursor** reads its own MCP config, not Claude Code's, so Playwright is mirrored for it in `.cursor/mcp.json`. Plugins are a Claude Code concept and have no Cursor equivalent.

---

## 8. Verify your work

- **Always:** `cd server && npm test` and `cd client && npm test` are the first check for every change — they must be green (offline, no API key needed). `npm run test:ai` (real Gemini) at phase boundaries only.
- **Definition of done for a roadmap phase:** its own new tests written and passing, `npm test` green in every half you touched, and the relevant `CLAUDE.md` updated. Don't declare a phase done with skipped/failing tests.
- **Manual smoke test:** run both halves (`server` on :5000, `client` on :5173), open the test lesson at `/view/69e39d0b60d467bd515a4945`, and test **both** logged-in and logged-out (incognito) — because of Golden Rule 2, both must work.
- Render's free tier **cold-starts**; the first request after idle is slow on top of AI latency. Slow ≠ broken.

---

## 9. Ignore these folders

`example_copy/`, `example_project/`, `test/`, `Ptest/` are reference material and scratch — **not the product**. Don't edit, build, or trust them. The only folders that matter: `client/`, `server/`, `graphify-out/`, and the root `ROADMAP*.md` / doc files.

---

*TL;DR: Read root `CLAUDE.md` → query `graphify` to find files → read the half's `CLAUDE.md` → follow `ROADMAP.md` → obey the two Golden Rules → keep `npm test` green.*
