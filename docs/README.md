# docs/ — Hot Potato system intuition

> Updated 2026-07-11 · reflects the code read on this date (client + server local HEAD, ahead of origin). When you change a mechanism described here, update the doc and bump this date.

**This folder is the "intuition" layer.** It answers *how the system works end-to-end and why it's built this way* — the things you can't get from reading 1,700 graph nodes or a route table. It exists so a new developer (including future-you) can hold the system in their head **without re-deriving it from the code**.

Three different tools, three different jobs — use the right one:

| When you need… | Go to |
| --- | --- |
| "What connects to what" (find the file) | the **knowledge graph** — `graphify query "..."` (see [`../AGENT.md`](../AGENT.md) §2) |
| "The exact route / field / env var" (a fact) | the **reference** — [`../server/CLAUDE.md`](../server/CLAUDE.md), [`../client/CLAUDE.md`](../client/CLAUDE.md), the READMEs |
| **"How does this flow work, and why is it like this"** (the mental model) | **these docs** ↓ |

---

## Read in this order (first day)

1. [`../README.md`](../README.md) — what the product is (5 min).
2. **[`ARCHITECTURE.md`](ARCHITECTURE.md)** — the whole system as one mental model + the five load-bearing ideas. **Start here.**
3. [`GLOSSARY.md`](GLOSSARY.md) — the vocabulary the codebase assumes you know. Skim once, refer back constantly.
4. The one **map** for the side you're touching:
   - Frontend → [`ui-structure.md`](ui-structure.md) (pages, guards, editor tree) + [`ui-state.md`](ui-state.md) (stores, persistence, axios).
   - Backend/data → [`data-model.md`](data-model.md) (entities + lifecycles).
5. The **flow** for your task → [`data-flow.md`](data-flow.md) (lesson save, answers, auth, AI call — with sequence diagrams) and [`../asking-flow.md`](../asking-flow.md) (the AI prompt build, in depth).
6. **Why** something is the way it is → [`ideas.md`](ideas.md) (the decision records).
7. Then the half's `CLAUDE.md` for the rules you must not break, and the graph to jump to files.

---

## The full map

**Understand the system (explanation):**

| Doc | Read it when |
| --- | --- |
| [`ARCHITECTURE.md`](ARCHITECTURE.md) | Before your first non-trivial change — the system's mental model + 5 load-bearing ideas. |
| [`data-flow.md`](data-flow.md) | Before touching save, answers, auth, or an AI call — end-to-end sequences (Mermaid). |
| [`../asking-flow.md`](../asking-flow.md) | Before touching the tutor prompt — predict the exact prompt Gemini sees. |
| [`ui-structure.md`](ui-structure.md) | Frontend: the page map, route guards, layout nesting, editor/viewer component tree. |
| [`ui-state.md`](ui-state.md) | Frontend: the ~15 Zustand stores, what persists where, the axios/token flow. |
| [`data-model.md`](data-model.md) | Backend: the 9 models, their relationships, and each entity's lifecycle. |
| [`GLOSSARY.md`](GLOSSARY.md) | Anytime a term (block, mode, diagnostics, agent_settings…) is unclear. |
| [`ideas.md`](ideas.md) | When something looks weird — the *why* (19 ADRs) + what breaks if you "fix" it. |

**Do the work (how-to & operations):**

| Doc | Read it when |
| --- | --- |
| [`onboarding.md`](onboarding.md) | Day 0 — clone to a working AI reply, then what to read. |
| [`testing.md`](testing.md) | Before writing tests — the offline strategy + a test recipe per layer. |
| [`security-model.md`](security-model.md) | Before adding any guard — the intentional "open by default" posture. |
| [`operations.md`](operations.md) | Deploying or debugging prod — env matrix, smoke, incident table. |
| [`performance.md`](performance.md) | Touching the bundle/fonts — the speed budget and why it exists. |

**Half-specific deep-dives (live inside their repo):**

| Doc | Read it when |
| --- | --- |
| [`../server/docs/ai-internals.md`](../server/docs/ai-internals.md) | Editing the tutor engine — a module-by-module tour of `services/tutor/*`. |
| [`../client/docs/editor-internals.md`](../client/docs/editor-internals.md) | Editing the lesson editor — the node system, question blocks, Fabric bridge, AI hub. |
| [`../client/CONTRIBUTING.md`](../client/CONTRIBUTING.md) · [`../server/CONTRIBUTING.md`](../server/CONTRIBUTING.md) | Contributor conventions + recipes (add an endpoint / node / store / action). |

**Not in this folder (but related):** [`../ROADMAP.md`](../ROADMAP.md) (product work plan) · [`../ROADMAP-docs.md`](../ROADMAP-docs.md) (this documentation's own roadmap) · [`../notes.md`](../notes.md) (tech-debt backlog) · [`../server/prompt-notes.md`](../server/prompt-notes.md) (prompt change log).

---

## Conventions for editing these docs

- **Explain intent and invariants, not line-by-line code.** Code moves; the *why* and the *what-must-stay-true* survive refactors. Point at a *file + concept*, never a line number.
- **Prose and tables first.** The one exception is **Mermaid** diagrams (they render on GitHub) for lifecycles, the ER model, and sequences — never ASCII art.
- **Link, don't fork.** The route table, env vars, and model fields live in `CLAUDE.md`/READMEs. Reference them; never copy them here (copies drift).
- **Date every doc** against the day you verified it. A doc that can't say when it was last true is a trap.
- **Thai-first only for tone/persona/user-facing copy** ([`../asking-flow.md`](../asking-flow.md), `prompt-notes.md`). The structural docs here are English with Thai glosses for domain terms — matching [`../ROADMAP.md`](../ROADMAP.md).
