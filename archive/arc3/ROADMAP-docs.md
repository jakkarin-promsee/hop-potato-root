# ROADMAP-docs.md — Hot Potato Documentation Roadmap

The single plan for the project's **explanation / "system intuition" documentation** — the docs that let a *new human developer* (not just an AI agent) understand *how the code actually works and why*, and extend it confidently. Written 2026-07-11.

> **How this differs from the two things we already have:**
>
> - **graphify (`graphify-out/`) = the skeleton.** An auto-generated map of *what connects to what* — 1190 nodes, communities, god nodes, shortest paths. Great for "where is X", useless for "why is it built this way".
> - **The `CLAUDE.md` / `README.md` layer = the reference + the agent rulebook.** Facts, route tables, env vars, gotchas, and the rules an agent must not break. Already strong.
> - **This roadmap = the intuition.** End-to-end flows, the *why* behind non-obvious decisions, contributor recipes, a shared vocabulary. The one exemplar that already exists is [`asking-flow.md`](asking-flow.md) — read it once and you can *predict the exact prompt Gemini sees*. **That is the bar.** This roadmap extends that treatment to every part of the system.
>
> เป้าหมาย: คนที่ไม่เคยเห็นโปรเจกต์นี้มาก่อน อ่านชุดนี้แล้ว "เก็ต" ว่าระบบคิดยังไง แก้ตรงไหนต้องระวังอะไร โดยไม่ต้องไปนั่งไล่โค้ดเองทั้งหมด

---

## 0. Progress — Tiers D0–D5 written (2026-07-11)

The intuition documentation was **built out end-to-end in one autonomous pass** (all source-verified against local HEAD). What landed:

| File | Covers | Tier |
| --- | --- | --- |
| [`docs/README.md`](docs/README.md) | Human docs index + reading order | D0.1 |
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | System mental model + 5 load-bearing ideas + request lifecycle | D0.2 |
| [`docs/data-flow.md`](docs/data-flow.md) | Lesson save, answers, auth/401, tutor call, teacher copilot (Mermaid sequences) | D1.1–D1.5 (consolidated) |
| [`client/docs/editor-internals.md`](client/docs/editor-internals.md) | Node system, question blocks, Fabric bridge, formula, AI hub | D1.6 |
| [`server/docs/ai-internals.md`](server/docs/ai-internals.md) | Module-by-module tour of `services/tutor/*` + controller | D1.7 |
| [`docs/data-model.md`](docs/data-model.md) | ERD + entity lifecycles (supersedes stale `schema.md`, now deprecated) | D2.1 |
| [`docs/GLOSSARY.md`](docs/GLOSSARY.md) | Domain vocabulary | D2.2 |
| [`docs/ui-structure.md`](docs/ui-structure.md) + [`docs/ui-state.md`](docs/ui-state.md) | UI skeleton (pages/guards/editor tree) + store map/persistence/token flow | D2.3 (+ new UI-structure) |
| [`docs/onboarding.md`](docs/onboarding.md) | Clone → working AI reply → what to read | D3.1 |
| [`docs/testing.md`](docs/testing.md) | Offline test strategy + a recipe per layer | D3.7 |
| [`client/CONTRIBUTING.md`](client/CONTRIBUTING.md) + [`server/CONTRIBUTING.md`](server/CONTRIBUTING.md) | Conventions + recipes: add endpoint / node / store / creator action / prompt / env | D3.2–3.6, D3.8, D5.1, D5.5 |
| [`docs/ideas.md`](docs/ideas.md) | 19 ADRs — the "why" + what-breaks | D4.1 |
| [`docs/security-model.md`](docs/security-model.md) | Intentional "open by default" posture, guard matrix, threat model | D4.2 |
| [`docs/operations.md`](docs/operations.md) | Env matrix, deploy, smoke, incident table | D4.3 |
| [`docs/performance.md`](docs/performance.md) | The speed budget & why | D4.4 |
| `{client,server}/CHANGELOG.md`, `server/SECURITY.md`, `.github/` PR + issue templates | Release history, vuln reporting, PR/issue scaffolds | D5.2, D5.3, D5.6 |

**Deliberate deviations:** the granular per-flow D1 files were consolidated into one `data-flow.md`; the D3 how-to recipes were folded into the per-repo `CONTRIBUTING.md` files (as the plan permitted). **Diagrams (D5.7)** are embedded as Mermaid throughout, not a separate pass.

**Still open (owner decisions / optional):**
- **D5.4 LICENSE** — not created: this is the owner's call. `server/README.md` declares **ISC**; the root README says "unspecified." Recommend resolving to ISC and adding a `LICENSE` file per repo. *(Left for the owner — I don't pick a license unilaterally.)*
- **D5.8 OpenAPI/Postman** — optional; skipped (the README route tables cover humans).
- **Freshness:** every doc is dated 2026-07-11 against local HEAD; re-verify at the launch-tier graph rebuild.

The tiers below remain the plan of record and the rationale for each doc.

---

## 1. Philosophy — the four documentation layers

Good project docs answer four *different* questions for four *different* moments. Most projects blur them; Hot Potato should keep them distinct so each doc has one job and stays short.

| Layer | Answers | Reader's moment | Hot Potato status |
| --- | --- | --- | --- |
| **A. Orientation** | "Where do I even start?" | First 5 minutes | ✅ Strong (`README.md`, `AGENT.md`) — missing only a *human* docs index |
| **B. Reference** | "What is the exact signature / route / field / env var?" | While coding, need a fact | ✅ Strong (`server/README.md` route tables, `CLAUDE.md` data model + env) |
| **C. Explanation / intuition** | "How does this flow work end-to-end, and *why* is it built this way?" | Before touching an unfamiliar area | ⚠️ **The gap.** Only `asking-flow.md` exists |
| **D. How-to guides** | "How do I add a new X, correctly, the way this repo does it?" | Extending the system | ⚠️ **The gap.** Scattered hints in `CLAUDE.md`, no recipes |

**This roadmap is 80% about Layers C and D** — the two that make a project *hand-off-able*. Layers A and B get only small top-ups (a docs index, an ADR link, keeping the stale `schema.md` honest).

### Non-negotiable doc conventions (carried from the owner's style)

1. **Prose and tables over ASCII diagrams.** The owner explicitly prefers written explanation. The *one* sanctioned exception is **Mermaid** diagrams (`sequenceDiagram`, `erDiagram`, `stateDiagram`) — they render as real diagrams on GitHub, not ASCII art, and earn their place for request lifecycles, the data model, and state machines. Use sparingly, prose-first.
2. **Explain intent and invariants, not line-by-line code.** Code walkthroughs rot on the next refactor. Document *why a thing exists*, *what must stay true*, and *what breaks if you violate it* — those survive refactors. When you must point at code, point at a *file + concept*, not a line number.
3. **Bilingual where the reader is the owner or a Thai contributor; English for the neutral dev-facing spine.** The sibling roadmaps (`ROADMAP.md`) are English; `asking-flow.md` (tone-sensitive AI flow) is Thai. Match that: system/architecture/how-to docs in English with Thai glosses for domain terms; anything about *tone/persona/user-facing copy* stays Thai-first.
4. **Every doc names its own freshness.** Top line: date + "verified against commit `<hash>`" (like `GRAPH_REPORT.md` does). A doc that can't say when it was last true is a trap.
5. **One fact, one home.** Never re-state the route table or env list that already lives in `server/CLAUDE.md` — *link* to it. Intuition docs reference the source of truth, they don't fork it.

---

## 2. Current-state audit (what exists, by layer)

Honest inventory so we build on strength and only fill real gaps.

| Existing doc | Layer | Verdict |
| --- | --- | --- |
| `README.md`, `client/README.md`, `server/README.md` | A + B | Excellent product + run/deploy reference. Keep. |
| `AGENT.md` | A | The *agent* front door. A *human* newcomer still lacks an equivalent "read these 6 docs in this order" map. |
| `CLAUDE.md` (root / client / server) | B (+ rules) | Dense, accurate reference + agent guardrails. The route table, data model, env vars, and "careful-not-to-break" lists live here — the intuition docs should link, never duplicate. |
| **`asking-flow.md`** | **C** | **The template.** The only true end-to-end intuition doc. Every Tier D1 doc should aim for this quality. |
| `client/src/components/README.md` | C (partial) | Good editor data-flow sketch (TipTap-as-source-of-truth, Fabric `saveState` bridge). Short — a candidate to expand, not restart. |
| `server/prompt-notes.md` | C (log) | Excellent decision *log* for prompts. Model for a broader `CHANGELOG` / ADR habit. |
| `server/src/models/schema.md` | B | **Stale** (predates several models/fields). Slated for replacement by a generated/verified data-model doc (Tier D2). |
| `notes.md` | — (backlog) | Audit findings / tech-debt backlog. Not user-facing docs; leave as-is. |
| `ROADMAP.md`, `ROADMAP-guide.md`, `plan/*`, `archive/*` | — (planning) | Work planning, not system docs. This file is their sibling. |

**Takeaway:** reference and agent-rules are ~90% done; **explanation and how-to are ~10% done** (one flow doc, one partial editor doc). That is exactly the target of Tiers D1–D4.

---

## 3. Where the docs live (placement rules)

The repo split forces a decision for every doc. Codify it once:

- **`client/` and `server/` are separate git repos; the `Hot-Potato/` root is an untracked workspace.** (See root `CLAUDE.md` §"Git reality".)
- **Cross-cutting intuition docs** (a flow that spans both halves, the glossary, ADRs, the system architecture) → live at the **workspace root**, ideally in a new **`docs/`** folder, exactly like `asking-flow.md` already sits at root. They travel with neither sub-repo — that is acceptable and already the norm for `ROADMAP*.md`.
- **Half-specific intuition docs** (client editor internals, server AI service internals) → live **inside that half** (`client/docs/`, `server/docs/`) so they ship with the repo a contributor actually clones. `client/src/components/README.md` already follows this.
- **Standard OSS hygiene files** (`CONTRIBUTING.md`, `CHANGELOG.md`, `SECURITY.md`, `LICENSE`) → **one per sub-repo** (they must be in the repo GitHub renders). Do **not** put them only at the workspace root.
- **The human docs index** → workspace root `docs/README.md` (or a root `DOCS.md`), linked from `README.md` and `AGENT.md`.

Proposed target tree (new files in **bold**):

```
Hot-Potato/
├── docs/                         # NEW — cross-cutting intuition (workspace-level)
│   ├── README.md                 # ← the human docs index / reading order (D0)
│   ├── ARCHITECTURE.md           # system-level "how it all thinks" (D0)
│   ├── GLOSSARY.md               # domain vocabulary (D2)
│   ├── flows/                    # end-to-end flow docs (D1) — asking-flow.md joins here
│   │   ├── lesson-lifecycle.md
│   │   ├── answer-and-feedback.md
│   │   ├── auth-and-session.md
│   │   ├── rendering-pipeline.md
│   │   └── teacher-copilot.md
│   ├── data-model.md             # ERD + entity lifecycles (D2, replaces schema.md)
│   ├── decisions/                # ADRs (D4)
│   │   └── NNNN-<slug>.md
│   ├── security-model.md         # (D4)
│   └── operations.md             # deploy / ops runbook (D4)
├── asking-flow.md                # (existing) → move under docs/flows/ or link from index
├── client/
│   ├── CONTRIBUTING.md           # NEW (D5)
│   ├── CHANGELOG.md              # NEW (D5)
│   ├── docs/
│   │   ├── editor-internals.md   # expand components/README.md (D1)
│   │   └── state-architecture.md # Zustand + axios + query (D2)
│   └── src/components/README.md  # (existing) editor data flow
└── server/
    ├── CONTRIBUTING.md           # NEW (D5)
    ├── CHANGELOG.md              # NEW (D5)
    ├── SECURITY.md               # NEW (D5)
    └── docs/
        └── ai-internals.md       # persona/parse/retry/memory deep-dive (D1)
```

> Keep `AGENT.md` as-is (the AI front door); `docs/README.md` is its human counterpart. They cross-link.

---

## 4. Tier D0 — Foundation & map 🧭

**Theme: a newcomer must know *what to read and in what order* before reading anything deep.** Two documents; both small; both unblock everything else.

### D0.1 — `docs/README.md` (the human docs index)

- **Purpose:** the "read these, in this order" front door for a *human* who just cloned the workspace. AGENT.md serves agents; this serves people.
- **Audience:** new contributor, day 0.
- **Outline:**
  - 60-second product recap + the two Golden Rules (link, don't restate).
  - A reading-order table: README → this ARCHITECTURE → the flow relevant to your task → the half's CLAUDE.md → the how-to guide.
  - A map table of *every* doc (name · layer · one-line "read when").
  - The freshness convention + "how to update a doc when you change a mechanism".
- **Lives:** `docs/README.md` · **Size:** S · **Depends on:** nothing (write first, fill links as docs land).

### D0.2 — `docs/ARCHITECTURE.md` (how the whole system thinks)

- **Purpose:** the single narrative that ties the two halves together — the doc that's currently *missing between* the module maps in each `CLAUDE.md`. Not a route list; a mental model.
- **Audience:** anyone before their first non-trivial change.
- **Outline:**
  - The two-tier picture: browser → client (Vercel SPA) → REST → server (Render) → Mongo/Gemini/Cloudinary. **One Mermaid `graph` diagram** (the sanctioned exception).
  - The request lifecycle in words: axios attaches JWT → route → `optionalAuth`/`protect` → controller validates inline → model → response; error handler last. Link `server/CLAUDE.md` for the module map.
  - The **five load-bearing ideas** a reader must internalize, each 2–4 sentences with a "why" and a "what breaks if violated":
    1. **TipTap document = single source of truth** (all block/canvas/question/formula data is node attrs; React state won't persist).
    2. **Server owns lesson context** (client never ships lesson text to the AI).
    3. **Anonymous = full access, login = persistence only** (Golden Rule 2, enforced by `optionalAuth`).
    4. **Never ration tokens** (Golden Rule 1; rate limits are bot-only, per-surface).
    5. **Preview → accept for all AI writes into the doc** (creator copilot never auto-applies).
  - The trust/latency realities: Render cold starts, transient-ISP retry, in-memory observability wiped on restart.
- **Lives:** `docs/ARCHITECTURE.md` · **Size:** M · **Depends on:** nothing (draws only on existing knowledge).

---

## 5. Tier D1 — Core intuition flows 🌊 (the heart)

**Theme: one `asking-flow.md`-quality doc per major end-to-end flow.** Each answers "what happens, in order, across both halves, and where are the sharp edges." These are the highest-value docs in the whole plan — they are what a maintainer will actually reach for.

**Shared template for every flow doc:** (1) the trigger + surfaces, (2) the ordered steps across client→server→client with the file that owns each step, (3) a Mermaid `sequenceDiagram` if it clarifies, (4) persistence (logged-in vs anonymous), (5) the invariants / "careful-not-to-break", (6) how to observe/measure it locally.

### D1.0 — `asking-flow.md` (already exists) — adopt as the canonical example
Re-home under `docs/flows/` (or link from the index) and add the standard freshness header. It is the reference implementation for D1.1–D1.5.

### D1.1 — Lesson lifecycle: create → edit → publish → view
- **Why it's needed:** the autosave + optimistic-concurrency dance and the TipTap-source-of-truth rule are the two things a new editor-touching dev breaks first.
- **Content pointers:** `POST /content/create` (blank) → editor autosave `PUT /content/:id` with `clientUpdatedAt` → **409 on conflict** → `access_type` (public/link-only/private) + `agent_settings` at publish → `GET /content/load` enforces access → serializer for the read-only viewer. Denormalized `author_name`/`collaborator_names` and how renames propagate.
- **Lives:** `docs/flows/lesson-lifecycle.md` · **Size:** M · **Depends on:** D0.2.

### D1.2 — Answer & feedback lifecycle (the student's core loop)
- **Why:** the deterministic-evaluation-then-AI-coaching split (client computes level + diagnostics; server gates evaluation on the `[งานของเธอตอนนี้…]` tag) is subtle and central to the product.
- **Content pointers:** answer a question node → `questionEvaluation.ts` computes level/accuracy/diagnostics → `callTutorStream` mode `question_feedback`/`write_evaluation` → `FeedbackDiscussionPanel` thread (mode `followup`, *no* eval payload — why "สวัสดี" isn't graded) → answers persisted per `blockId` in `UserContent.answers` (login) or client store (anon). A **state diagram** of a question card (unanswered → evaluated → threaded) earns its Mermaid here.
- **Lives:** `docs/flows/answer-and-feedback.md` · **Size:** M–L · **Depends on:** D1.0 (references `asking-flow.md`).

### D1.3 — Auth & session flow
- **Why:** the structured-401 ↔ axios auto-logout contract is a two-sided invariant that silently breaks login if either side drifts.
- **Content pointers:** register/login/Google (`@react-oauth/google` → verify ID token → find-or-create/auto-link) → JWT in `auth.store` (persisted) → axios request interceptor attaches Bearer → server `protect`/`optionalAuth` → **structured 401** (`forceRelogin`/`clearToken`, the `code` enum) → response interceptor logs out + redirects on protected paths only → the three route guards (`ProtectedRoute`/`RequireLogin`/`PublicRoute`). Why `optionalAuth` (not `protect`) on AI/read paths (Golden Rule 2).
- **Lives:** `docs/flows/auth-and-session.md` · **Size:** M · **Depends on:** D0.2.

### D1.4 — Rendering pipeline: saved doc → viewer → safe markdown AI replies
- **Why:** how a stringified TipTap doc becomes the student view, and how AI text is sanitized, is invisible until someone breaks XSS-safety or the zoom.
- **Content pointers:** `tiptap_json` (stringified) → `TiptapViewer`/`FabricCanvasReadOnly` read-only render → the CSS `zoom` decision (not `transform: scale`; the double-scrollbar bug) vs the *separate* app-wide font-size knob → AI replies through `MarkdownMessage.tsx` (react-markdown + strict allowlist; raw HTML skipped) paired with the server `stripReportLabels`/`stripInternalTags` guards.
- **Lives:** `docs/flows/rendering-pipeline.md` · **Size:** M · **Depends on:** D1.6 (editor internals) for shared vocabulary.

### D1.5 — Teacher copilot flow (`/api/creator/assist`)
- **Why:** the "AI writes into the doc, but only via preview→accept, and never as raw TipTap JSON" rule is the safety spine of the whole editor-AI feature.
- **Content pointers:** `lib/creatorApi.ts` single bridge → `POST /creator/assist` (11 actions, teacher-only `protect` + owner check) → server prompts (`services/creator/`) separate from the tutor persona → JSON mode + tolerant parse + **server-side validation** (malformed items dropped) → client preview dialog → accept inserts as a normal editor transaction (autosave/409 untouched). Model routing per action.
- **Lives:** `docs/flows/teacher-copilot.md` · **Size:** M · **Depends on:** D1.1.

### D1.6 — Editor internals (expand `client/src/components/README.md`)
- **Why:** the current doc is a good sketch but stops at the Fabric bridge; the *question-node system* and *formula block* — the actual critical-thinking machinery — are undocumented as a mental model.
- **Content pointers:** the node registry (`createEditorExtensions`) → each `*Node.ts` (schema/attrs) + `*View.tsx` (render) pattern → the five question types and their shared answer shape → `FabricCanvasNode` + `saveState` sync bridge + `CanvasContext` as UI-pointer-only → `FormulaBlock` (`latex` attr = source of truth; the legacy tree is fallback-only) → `dynamicUpdate` static-mode toolbar trick → H1=title / H2=auto-numbered-section convention.
- **Lives:** expand `client/src/components/README.md` (or new `client/docs/editor-internals.md` that it links) · **Size:** M–L · **Depends on:** nothing.

### D1.7 — AI service internals (server-side companion to `asking-flow.md`)
- **Why:** `asking-flow.md` covers *what prompt gets built*; this covers *the code that builds it* — the module a server dev edits.
- **Content pointers:** `services/tutor/` tour — `persona.ts` (system-instruction assembly order, the exported tag constants + `INTERNAL_TAG_PREFIXES` rule), `parse.ts` (suggestions gate + strip guards), `genaiClient.ts` (stream vs JSON), `retry.ts` (why it exists — flaky ISP), `personality.ts` (layered under teacher settings), `memory.ts` (async every N turns), `models.ts` (routing single source of truth). Cross-link `prompt-notes.md` as the change log.
- **Lives:** `server/docs/ai-internals.md` · **Size:** M · **Depends on:** D1.0.

---

## 6. Tier D2 — Data & domain 🗃️

**Theme: a shared vocabulary and an honest data model.** These stop the "what does *block* / *mode* / *diagnostics* even mean" tax that every newcomer pays.

### D2.1 — `docs/data-model.md` (ERD + entity lifecycles) — replaces stale `schema.md`
- **Purpose:** the authoritative, *current* data model as a mental model, not a field dump (the field dump stays in `server/CLAUDE.md`).
- **Outline:** a Mermaid `erDiagram` of the 9 models and their relationships (User owns Content; (User,Content)→UserContent+LearningHistory; ChatSession/StudentMemory per user; Category→Image) → a short *lifecycle* paragraph per entity (born when? mutated by? capped/indexed how? deleted by?) → the denormalization note (author snapshots) → the "`.model.ts` is truth, this doc explains it" disclaimer.
- **Action:** **delete or clearly deprecate `server/src/models/schema.md`** and point it here (it is a known stale trap flagged in three other docs).
- **Lives:** `docs/data-model.md` · **Size:** M · **Depends on:** nothing.

### D2.2 — `docs/GLOSSARY.md` (domain vocabulary)
- **Purpose:** define every term the codebase assumes you know. One place, alphabetical, one-line each with a link to where it lives.
- **Must include (at minimum):** *block / node*, *question node* (5 types), *agent_settings*, *personality (preset)*, *StudentMemory / memory digest*, *feedback mode* (`quick_check` / `full_reflection`), *tutor mode* (`free_chat`/`question_feedback`/`write_evaluation`/`followup`), *diagnostics*, *clientThread*, *ChatSession*, *suggestions / chips*, *ai_judge*, *guide answer / distractor*, *denormalized author snapshot*, *optimistic concurrency / `clientUpdatedAt` / 409*, *optionalAuth vs protect*, *structured 401*, *cold start*, *loanword rule*, *the two Golden Rules*.
- **Lives:** `docs/GLOSSARY.md` · **Size:** S–M · **Depends on:** nothing (grows as flow docs land).

### D2.3 — `client/docs/state-architecture.md` (frontend state & data flow)
- **Purpose:** how the 15 Zustand stores relate, when to use a store vs TanStack Query, and the persistence boundaries.
- **Outline:** store-by-responsibility table (link `client/CLAUDE.md`, don't re-list) → what's `persist`ed to localStorage (`auth`, `bookmark`, `tutor-personality`, `app-font-size`) vs server vs anonymous-ephemeral → the axios single-instance rule (token read outside React) → store-driven vs query-driven pages ("match the page you're editing").
- **Lives:** `client/docs/state-architecture.md` · **Size:** S–M · **Depends on:** D1.3.

---

## 7. Tier D3 — Contributor how-to guides 🛠️

**Theme: the "how do I add a new X the way this repo already does it" recipes.** Task-oriented, copy-the-pattern, each ending in "…and here's the test you must add." These convert a reader into a contributor.

### D3.1 — Onboarding runbook: "from clone to a working AI reply in 20 minutes"
- Beyond the README's env list: the *full loop* — both halves running, seed/demo data (the guide seed scripts, the canonical test lesson `/view/69e39d0b60d467bd515a4945`), how to get a Gemini key, how to test **both auth states**, and the first-run gotchas (cold start ≠ broken, Cloudinary preset, CORS-open-in-dev). **Lives:** `docs/onboarding.md` (or a section of `docs/README.md`) · **Size:** S–M.

### D3.2 — "Add a REST endpoint" (route → controller → model → guard → test)
The canonical server change: pick guard (`open`/`optionalAuth`/`protect`/admin), inline validation style (no schema lib — match `tutor.controller.ts`), owner/collaborator access pattern, error via `error.middleware`, and the supertest to add. **Lives:** `server/docs/` or `server/CONTRIBUTING.md` · **Size:** S.

### D3.3 — "Add a question type / editor node"
The `*Node.ts` + `*View.tsx` pattern, registering in `createEditorExtensions`, the shared answer shape in `UserContent.answers`, how it reaches the tutor (`tutorApi.ts` mode), and the serializer entry in `lessonContext.service.ts` so the AI can *see* it. **Lives:** `client/docs/` · **Size:** M.

### D3.4 — "Add a teacher-copilot action"
New action in `services/creator/prompts.ts` + `parse`/`validate`, model routing, the `callCreator` bridge, the preview→accept UI, and the "never raw TipTap JSON" rule. **Lives:** `server/docs/` + `client/docs/` cross-link · **Size:** S–M.

### D3.5 — "Add a store / page / route"
Zustand `create()` conventions, `persist` opt-in, wiring a lazy route + guard in `App.tsx`, the `@` alias rule, i18n via `useAppI18n`. **Lives:** `client/docs/` · **Size:** S.

### D3.6 — "Change a prompt safely" (the tone-gated path)
The single most dangerous edit in the repo: edit `persona.ts` → update snapshot tests → **append to `prompt-notes.md`** → owner is tone referee → `AI_DEBUG_PROMPTS` to inspect → `measure-context.ts` to size. Mostly a formalization of the existing working agreement. **Lives:** `server/docs/` (link from `prompt-notes.md`) · **Size:** S.

### D3.7 — Testing strategy & philosophy
What the offline harness guarantees (in-memory Mongo + mocked Gemini, no key/network), where tests live in each half, the "every phase ships its own tests / `npm test` green" agreement, when `test:ai` is warranted, and how to write a test for each layer (unit prompt-builder, supertest endpoint, happy-dom store/component). **Lives:** `docs/testing.md` (cross-cutting) · **Size:** S–M.

### D3.8 — "Add / change an env var"
Where it's read, the client `VITE_` exposure rule, the `.env` + README + `CLAUDE.md` table + (if AI) `asking-flow.md` update checklist, and the "unset behavior" documentation habit (e.g. `CORS_ORIGINS` fail-open). **Lives:** short section in `server/CONTRIBUTING.md` + `client/CONTRIBUTING.md` · **Size:** S.

---

## 8. Tier D4 — Cross-cutting explanation & operations ⚙️

**Theme: capture the *why* behind decisions, the security posture, and the run-it-in-production knowledge — the things currently living only in the owner's head or scattered across gotcha lists.**

### D4.1 — Architecture Decision Records (`docs/decisions/NNNN-*.md`)
- **Purpose:** the highest-leverage "intuition" artifact — one short record per non-obvious decision so nobody re-litigates or accidentally reverts it. A lightweight ADR (context · decision · consequences · status), ~1 page each.
- **Seed the backlog from decisions already made and scattered:**
  1. Two separate git repos (not a monorepo) + the home-dir git quirk.
  2. Server-owned lesson context (client doesn't ship lesson text).
  3. TipTap node as single source of truth.
  4. `optionalAuth` everywhere for AI/public reads (Golden Rule 2 as architecture).
  5. Never ration tokens; rate limits bot-only + per-surface (Golden Rule 1).
  6. In-memory observability, wiped on restart (no paid SaaS; free-tier constraint).
  7. CSS `zoom` over `transform: scale` (double-scrollbar bug).
  8. App-wide font size vs viewer zoom kept as separate mechanisms.
  9. Preview→accept + "AI never emits raw TipTap JSON" for creator copilot.
  10. Markdown-only AI output + strip guards (no headings/tables/labels).
  11. Deploy-last-in-one-shot (Tier 5 discipline).
  12. Personality layered *under* teacher `agent_settings`.
  13. Loanword accessibility rule (`คำไทย (english)`).
  14. Route-level code splitting to protect low-bandwidth students.
- **Lives:** `docs/decisions/` · **Size:** M for the set (S each; write the top ~6 first) · **Depends on:** nothing.

### D4.2 — `docs/security-model.md`
- **Purpose:** consolidate the security posture that's currently spread across `server/CLAUDE.md` gotchas + the README security section into one intentional model.
- **Outline:** the guard matrix (open/optionalAuth/protect/admin) and *why each route sits where it does* · the two Golden Rules as *security invariants* (adding `protect` to an AI path is a regression) · injection defenses (the "data, not instructions" lesson-context framing; the *known-open* `$regex` in search flagged in `notes.md`) · password hashing + Google ID-token verification · the structured-401 contract · rate-limiting model · what is admin-only · secrets/logging discipline (no student messages/tokens logged; `AI_DEBUG_PROMPTS` dev-only). Cross-link the future `SECURITY.md` (D5.3) which is the *reporting* file.
- **Lives:** `docs/security-model.md` · **Size:** M · **Depends on:** D1.3.

### D4.3 — `docs/operations.md` (deploy & ops runbook)
- **Purpose:** the production-operation knowledge a solo owner needs at 2am — more than the README's deploy blurb.
- **Outline:** the env matrix (which var on Render vs Vercel vs Google Console) · the launch checklist (lift from `ROADMAP.md` Tier 5, but as a *repeatable* runbook, not a one-time plan) · cold-start behavior + what "slow" means · reading the `/status` page + the in-memory error buffer · `npm run promote` admin bootstrap · rollback (both platforms auto-deploy from git — how to revert) · the graph-rebuild + doc-freshness sweep. · **Lives:** `docs/operations.md` · **Size:** M · **Depends on:** nothing.

### D4.4 — `docs/performance.md` (optional, small)
- Why the bundle is split (TipTap/Fabric/KaTeX in lazy chunks; entry ~138 kB gzip), the low-bandwidth mission driving it, self-hosted fonts, and the "don't regress the entry chunk" measurement habit. Could fold into ARCHITECTURE.md if kept short. **Size:** S.

---

## 9. Tier D5 — Professional / OSS hygiene 📦

**Theme: the standard files a serious open project has — the ones a stranger (or a future employer looking at the repo) expects to find.** One set per sub-repo (they must render on each GitHub repo).

| Doc | Purpose | Lives | Size |
| --- | --- | --- | --- |
| **D5.1 `CONTRIBUTING.md`** | Branch/commit conventions, "run `npm test` green", code style, PR expectations, link to the how-to guides (D3). | `client/` **and** `server/` | S each |
| **D5.2 `CHANGELOG.md`** | Human-readable release history (Keep-a-Changelog style). The `prompt-notes.md` habit, generalized to the whole repo. | `client/` **and** `server/` | S (ongoing) |
| **D5.3 `SECURITY.md`** | How to report a vulnerability + the "never log secrets / student data" posture. Pairs with `docs/security-model.md` (D4.2). | `server/` (primary) | S |
| **D5.4 `LICENSE`** | The root README says "not yet specified"; `server/README.md` says ISC. **Resolve the contradiction** and add a real `LICENSE` file per repo (owner decision — likely ISC or MIT). | both | XS (decision) |
| **D5.5 Code-style guide** | The conventions already stated in `CLAUDE.md` (TS everywhere, inline validation, `@` alias, `cn()`, file naming) pulled into one short "house style" doc, or a section of CONTRIBUTING. | both | S |
| **D5.6 GitHub templates** | `.github/PULL_REQUEST_TEMPLATE.md` + issue templates (bug/feature) — reinforce "tests green? both auth states checked?". | both | S |
| **D5.7 Diagrams pass** | A small set of **rendered Mermaid** diagrams embedded in the docs above: request lifecycle (ARCHITECTURE), ER (data-model), question-feedback state machine (answer flow), SSE streaming sequence (asking-flow). Prose-first; diagrams only where they beat words. | in-doc | S |
| **D5.8 (optional) API contract artifact** | An OpenAPI spec or committed Postman/`.http` collection for `/api/*`, generated or hand-kept, so consumers have a machine-readable contract. Nice-to-have; the README route tables already cover humans. | `server/` | M |

---

## 10. Suggested order & sizing

Priority mirrors the owner's usual instinct: **make it navigable → document the flows people break → give them vocabulary → give them recipes → capture the why → polish.** D-tiers are mostly independent; within a tier, top-listed items first.

| ID | Doc | Layer | Size | Lives | Depends on |
| --- | --- | --- | --- | --- | --- |
| D0.1 | Human docs index (`docs/README.md`) | A | S | root | — |
| D0.2 | `ARCHITECTURE.md` | C | M | root | — |
| D1.0 | Adopt `asking-flow.md` as the template | C | XS | root | — |
| D1.1 | Lesson lifecycle | C | M | root | D0.2 |
| D1.2 | Answer & feedback lifecycle | C | M–L | root | D1.0 |
| D1.3 | Auth & session flow | C | M | root | D0.2 |
| D1.4 | Rendering pipeline | C | M | root | D1.6 |
| D1.5 | Teacher copilot flow | C | M | root | D1.1 |
| D1.6 | Editor internals (expand) | C | M–L | client | — |
| D1.7 | AI service internals | C | M | server | D1.0 |
| D2.1 | Data model + ERD (replace `schema.md`) | B/C | M | root | — |
| D2.2 | Glossary | B | S–M | root | — |
| D2.3 | Frontend state architecture | C | S–M | client | D1.3 |
| D3.1 | Onboarding runbook | D | S–M | root | — |
| D3.2–3.8 | How-to recipes (endpoint / node / action / store / prompt / testing / env) | D | S each | half-specific | relevant D1 |
| D4.1 | ADR set (seed ~6, then grow) | C | M | root | — |
| D4.2 | Security model | C | M | root | D1.3 |
| D4.3 | Operations runbook | D | M | root | — |
| D4.4 | Performance notes (optional) | C | S | root | — |
| D5.1–5.6 | CONTRIBUTING / CHANGELOG / SECURITY / LICENSE / style / templates | A/hygiene | S each | per repo | — |
| D5.7 | Mermaid diagrams pass | C | S | in-doc | the docs they sit in |
| D5.8 | OpenAPI/Postman (optional) | B | M | server | — |

**Recommended path:** D0.1 → D0.2 → D2.2 (glossary early — everything else references it) → D1.1 → D1.2 → D1.3 → the rest of D1 → D2.1/D2.3 → D3.* recipes → D4.1 ADRs → D4.2/D4.3 → D5 hygiene as a finishing pass.

**Minimum viable "hand-off-able" set (if time is short):** D0.1, D0.2, D2.2, D1.1, D1.2, D1.3, D3.1, and the top 6 ADRs (D4.1). That alone takes the project from "only the author understands it" to "a competent stranger can extend it."

---

## 11. Conventions & keeping docs alive

- **Freshness header on every doc:** `> Updated YYYY-MM-DD · verified against client `<hash>` / server `<hash>`.` A doc that can't state its truth-date is untrustworthy — say so or fix it.
- **The existing working agreement extends to docs:** every code phase already "updates the relevant `CLAUDE.md`". Add: *if a phase changes an end-to-end flow, update its flow doc; if it makes a non-obvious decision, add an ADR.* Cheap at write-time, priceless later.
- **Link, never fork.** Intuition docs point at the reference (route tables, env lists, data fields in `CLAUDE.md` / READMEs). Duplicated facts drift; linked facts don't.
- **Tie doc-freshness to the graph rebuild.** The launch-tier graph rebuild (`/graphify --update`) is the natural checkpoint to also sweep these docs' freshness headers.
- **Owner remains the referee for tone/persona docs** (Thai-first, `prompt-notes.md` discipline) — those are product, not just docs.

---

## 12. Anti-goals — what NOT to write (so the docs don't rot)

- **Don't duplicate the knowledge graph.** graphify already answers "what connects to what." These docs answer "why and how" — never re-list the module map as prose.
- **Don't write line-by-line code walkthroughs.** They break on the next refactor. Document intent, invariants, and failure modes.
- **Don't re-state reference tables** (routes, env vars, model fields). They live in `CLAUDE.md` / READMEs; link them.
- **Don't over-diagram.** Prose first; a Mermaid diagram only where it genuinely beats a paragraph (lifecycles, ER, state machines, sequences). No ASCII art.
- **Don't document aspirational behavior.** Only what the code does *today* (verified against a commit). Wishlist items belong in `notes.md` / `ROADMAP.md`, not intuition docs.
- **Don't gold-plate the OSS hygiene (D5).** This is a solo, free-tier project — a real `LICENSE`, a `CONTRIBUTING.md`, and a `CHANGELOG.md` matter; a 12-stage PR pipeline does not.

---

*TL;DR: reference and agent-rules are already strong; this roadmap fills the **intuition** gap. Start with the docs index + `ARCHITECTURE.md` + a glossary, then write one `asking-flow.md`-quality flow doc per core loop (lesson, answer/feedback, auth, rendering, copilot, editor), give contributors recipes, capture the "why" as ADRs, and finish with standard OSS hygiene files — all prose-first, link-don't-fork, and dated against a commit.*
