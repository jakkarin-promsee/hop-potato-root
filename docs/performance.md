# performance.md — the speed budget & why it exists

> Updated 2026-07-11. Short by design. The mission audience is **students on bad connections**, so performance here is a *product* constraint, not vanity. The decisions live in [`ideas.md`](ideas.md) (ADR-014); this is the running budget + how not to regress it.

---

## 1. The one number that matters: the entry bundle

A student who only *reads* a lesson must not download the teacher's editor. Before Tier 2 the whole app shipped as one **646 kB gzip** chunk (TipTap + Fabric + KaTeX + lowlight) to every visitor. After splitting: the entry chunk for `/`, `/explore`, `/login` is **~138 kB gzip**.

**How it's kept small:**
- Every route except `Landing`/`NotFound` is `React.lazy` inside one `<Suspense>` (`App.tsx`).
- Heavy vendors are pinned to their own chunks via `vite.config.js` `manualChunks` (`tiptap`, `fabric`, `katex`) so they load only on editor/viewer routes and cache across deploys.
- Editor-only CSS (`indexTiptap.css`) is imported from `TipTapEditor`/`TiptapViewer`, **not** `main.tsx`.
- Guide screenshots are static files under `public/guide/` referenced by **URL, never `import`ed** — importing them would pull them into a JS chunk.

**The rule:** don't statically import a heavy lib into shared/entry code, and re-check the entry chunk size on every build (`npm run build` prints it). A regression here hurts exactly the users the product is for.

---

## 2. Fonts — self-hosted, no CDN

Fonts ship with the bundle via `@fontsource/*` (Geist Variable UI, Lora headings, JetBrains Mono, Inter for canvas text) — **no `fonts.googleapis.com`/`gstatic` requests** (a CDN font stalls first paint on bad links, and neither Google family had Thai glyphs anyway). Enforced by a regression test (`src/__tests__/no-cdn-fonts.test.ts`). Don't reintroduce CDN `@import`s. (`client/CLAUDE.md` §Fonts.)

---

## 3. Server-side performance notes

- **Lesson context is cached.** `lessonContext.service` caches the serialized lesson by `updatedAt` (LRU-ish, 100 entries) so repeated AI calls on the same lesson don't re-parse the TipTap JSON.
- **Model routing for latency/cost.** Quick-check feedback → fast model (`gemini-2.5-flash-lite`, ~1 s); coaching/free-chat → tutor model (`gemini-3.5-flash`, ~2 s). (ADR-018.)
- **Streaming hides latency.** SSE streams tokens as they arrive so the student sees text immediately instead of waiting for the full reply.
- **Cold starts dominate first-request time** (Render free tier) — no code fix; handled with UX loading states (see [`operations.md`](operations.md) §4).

---

## 4. What is *not* optimized (accepted)

No CDN in front of the API, no Redis (in-memory rate-limit buckets + lesson cache are per-instance — fine at one instance), no image CDN beyond Cloudinary's own, no SSR/prerender (so per-lesson link previews are deferred — ROADMAP Tier 6.D). These are deliberate for a solo free-tier project; revisit only if traffic demands it.
