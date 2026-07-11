# Tier 2 — Performance (bundle split + self-hosted fonts): Execution Plan

> **Who this is for:** an implementing agent (Cursor or any model) executing Tier 2 of [`../ROADMAP.md`](../ROADMAP.md).
> **How to use:** read §0–§2 completely before writing any code — they are verified ground truth (traced against the actual code and a real `npm run build` on 2026-07-10; **re-verified 2026-07-10 after Tier 0.A shipped** — client HEAD `e3c53ec`, every §2 claim still holds, test baseline now **74**). Then execute Phase 2.A → Phase 2.B in order, step by step. Every code snippet in §3–§4 is written against the real files — you should mostly be able to apply them as-is.
> **Why this matters:** the target students are on bad phones with bad internet. Today they download a **2,094.64 kB JS file (646.24 kB gzip)** to see the landing page — including the entire teacher editor, a canvas library, and a math typesetter they may never use. Tier 2 makes the common pages cheap. **Do not improvise beyond this plan;** when a real ambiguity appears, prefer the smallest change that satisfies the acceptance criteria and note it in `client/CLAUDE.md`.

---

## 0. Ground rules (read once, obey always)

### Golden Rules (override "best practice" — non-negotiable)
1. **Never ration tokens for real students.** (Not directly touched by this tier — but never "optimize" by limiting AI calls.)
2. **Anonymous users keep FULL access.** `/view/:id` and the tutor chat must work logged-out after every change here. The final smoke test runs in incognito too.

### Scope fence — Tier 2 touches ONLY the client
- **All work happens in `client/`.** Do not edit anything in `server/`.
- Files you will touch: `client/src/App.tsx`, `client/src/main.tsx`, `client/vite.config.js`, `client/index.html`, `client/src/index.css`, `client/package.json`, one new component, new test files, `client/CLAUDE.md`. Nothing else (except the optional stretch step A8).

### Things you will see and must NOT fix (owned by other tiers)
| Thing | Where | Owner |
| --- | --- | --- |
| `<script type="module" src="/src/main.jsx">` pointing at a file that is really `main.tsx` | `client/index.html` | Tier 0.A (SEO phase). Leave it — Vite resolves it. (If Tier 0.A already landed and it says `main.tsx`, also fine — just don't touch the tag.) |
| Missing meta/OG tags, `lang="en"`, favicon MIME | `client/index.html` | Tier 0.A |
| ESLint error: `'__dirname' is not defined` | `client/vite.config.js` | Tier 4. You WILL edit this file (step A4) — keep using the existing `__dirname` line as-is, do not refactor it, do not add eslint-disable comments. `npm run lint` having exactly this one pre-existing error is expected. |
| Duplicate file: `src/components/CloudinaryUpload.tsx` vs `src/pages/Cloudinaryupload.tsx` (near-identical copies) | client | Nobody yet — leave both alone. The `components/` copy is imported by editor sidebars; the `pages/` copy is the `/uploadimage` route. |

### Git reality ⚠️
- `client/` and `server/` are **two separate git repositories**. There is **no repo** at the `Hot-Potato/` root. If `git status` shows foreign files (e.g. `ChronoForge-FPGA-Engine`), you are in the wrong directory — stop and `cd client`.
- Always `cd client` before any git command.
- **Pre-existing dirty file (verified 2026-07-10):** `src/components/editor/EditorLeftSidebar.tsx` has an uncommitted owner edit (category label "Special" → "Question"). **It is not yours.** Do not revert it, do not commit it. At preflight (A0), run `git status --short` and note every already-modified file; when committing, **stage only the files this plan touched by explicit path** (`git add src/App.tsx src/main.tsx …`) — never `git add -A`/`git add .`. If the owner has committed it by the time you start, even better — nothing to dodge.
- **One commit per phase**, message style:
  - `perf(tier2.A): route-level code splitting + vendor chunks`
  - `perf(tier2.B): self-host fonts, drop Google Fonts CDN`
- **Never push** (owner decision D1: nothing is pushed until Tier 5 launch day).

### Commands & definition of done
```bash
cd client && npm install && npm run dev   # web on :5173 (vite)
cd client && npm test                     # vitest + happy-dom — must stay green (baseline: 74 tests / 18 files, verified 2026-07-10)
cd client && npm run build                # must be green; you will compare its size table before/after
```
- Server for manual smoke: `cd server && npm run dev` (API on :5000). Server tests are not your concern (no server changes).
- Manual smoke lesson: `http://localhost:5173/view/69e39d0b60d467bd515a4945` — test **both auth states** (logged in + incognito).
- **Definition of done per phase:** new tests for the phase's acceptance criteria written and passing, `npm test` green, `npm run build` green, before/after bundle numbers recorded in §7 of this file, `client/CLAUDE.md` updated, one commit in `client/`.
- Env vars: **add none.**

### Test conventions in this repo (match them exactly)
- Vitest + **happy-dom**. There is **no `@testing-library/react`** — do not install it. Tests render with raw `react-dom/client` `createRoot` + `act` from `react`, and use `MemoryRouter` or `window.history.pushState` for routing. Stores/axios are stubbed with `vi.mock`. Copy the harness pattern from `client/src/components/__tests__/guards.test.tsx` (render helper + `afterEach` cleanup).
- Test files live in `__tests__/` folders next to the code (`src/pages/__tests__/`, `src/components/__tests__/`, `src/lib/__tests__/`).

---

## 1. Mission & verified baseline

`npm run build` output on 2026-07-10 (record your own preflight run in §7 — it should match within a few kB):

| Asset | Raw | Gzip |
| --- | --- | --- |
| `dist/assets/index-*.js` (the ONLY JS chunk) | **2,094.64 kB** | **646.24 kB** |
| `dist/assets/index-*.css` (the only CSS) | 226.15 kB | 41.40 kB |
| KaTeX font files (~30 of them) + Geist woff2 + logo.png | static assets, lazy-fetched | — |
| Build warning | `(!) Some chunks are larger than 500 kB` | must be gone for the entry |

**What's inside the 2 MB chunk (import-graph, traced):** React 19 + react-router 7 + TanStack Query + zustand + axios + lucide-react + radix/shadcn + sonner (the "app shell", genuinely needed everywhere) **plus** TipTap 3 + `@tiptap/pm` (ProseMirror) + tiptap-markdown, Fabric.js 7, KaTeX, react-markdown + remark-gfm (only needed by the editor `/canvas/:id`, the viewer `/view/:id`, and `/uploadimage`).

**Target (roadmap acceptance):** total JS downloaded to render `/` and `/explore` ≤ **323 kB gzip** (half of today's 646). Expected realistic result after this plan: **~150–250 kB gzip** for the entry. The heavy vendor chunks (`tiptap`, `fabric`, `katex`) will only ever be fetched when a lesson viewer/editor route mounts.

---

## 2. Verified current state (trust this over guesses; re-verify against code if something looks off)

### 2.1 Routing — `client/src/App.tsx`
All 15 pages are **statically imported** (lines 10–25) and **all are default exports**. Only `App.tsx` imports from `./pages/` (verified by grep — no component leaks a page import, so lazy-splitting pages cleanly splits the graph). Route map:

| Path | Page file | Guard | Weight |
| --- | --- | --- | --- |
| `/` | `pages/Landing.tsx` | — | light (lucide + Button only) — **keep eager** (first paint) |
| `*` | `pages/NotFound.tsx` | — | light — **keep eager** |
| `/status` | `pages/Status.tsx` | — | lazy |
| `/dashboard` | `pages/Dashboard.tsx` | `ProtectedRoute` | lazy |
| `/canvas/:id` | `pages/TipTapCanvas.tsx` | `ProtectedRoute` | lazy — **the heavy one** (TipTap+Fabric+KaTeX) |
| `/uploadimage` | `pages/Cloudinaryupload.tsx` | `ProtectedRoute` | lazy |
| `/view/:id` | `pages/TiptapView.tsx` | — (public!) | lazy — heavy (viewer needs TipTap+Fabric+KaTeX to render lessons — expected, fine) |
| `/login` | `pages/Login.tsx` | `PublicRoute` | lazy |
| `/explore` | `pages/Explore.tsx` | — | lazy |
| `/guide` | `pages/Guide.tsx` | — | lazy (static content page) |
| `/history` | `pages/History.tsx` | `RequireLogin` | lazy |
| `/create` | `pages/Create.tsx` | `ProtectedRoute` | lazy |
| `/profile` | `pages/Profile.tsx` | `RequireLogin` | lazy |
| `/change-password` | `pages/ChangePassword.tsx` | `RequireLogin` | lazy |
| `/settings` | `pages/Setting.tsx` ← note: file is **`Setting`** singular | — | lazy |

Guards (`ProtectedRoute`, `PublicRoute` — default exports; `RequireLogin` — **named** export) and `layouts/AppLayout.tsx` (**named** export, wraps `/login`…`/settings` with `TopNav` + `<Outlet/>`) stay **statically imported** — they're tiny and needed for routing decisions.

### 2.2 CSS entry — `client/src/main.tsx`
`main.tsx` imports `./index.css` (Tailwind v4 + theme tokens) **and** `./indexTiptap.css` (~660 lines of editor/ProseMirror styles that every page currently downloads). `katex/dist/katex.min.css` is imported inside `components/editor/FormulaBlock/FormulaCanvas.tsx` — after route-splitting, Vite automatically moves it into the lazy chunk's CSS; no action needed.

### 2.3 Fonts — messier than the roadmap says (all verified)
Three font sources exist today:

1. **`client/index.html`** loads **Inter (300–700) + DM Sans (400–700)** from Google Fonts CDN (2 preconnects + 1 stylesheet link). **DM Sans is referenced NOWHERE in the codebase** — pure dead weight.
2. **`client/src/index.css` line 1** has a second Google Fonts `@import url(...)` loading **Lora + Inter + JetBrains Mono** — this survives into the built CSS as a render-blocking external request.
3. **`@fontsource-variable/geist`** (index.css line 9) — already self-hosted, already in `package.json`, ships as woff2 in `dist/`.

Which fonts are ACTUALLY used:
- **Body/UI sans = "Geist Variable"** (self-hosted, already fine). The `@theme` block at index.css:14 declares `--font-sans: "Inter"…` but the **`@theme inline` block at index.css:116 overrides it** to `"Geist Variable", sans-serif`. So Inter is *not* the UI font.
- **`font-serif` = Lora** — used for headings on Landing, Explore, Guide, History, Dashboard, Login, Create, Profile, Settings, ChangePassword, RequireLogin, ContentCard. Really used; must keep working.
- **`font-mono` = JetBrains Mono** — used all over `pages/Status.tsx`, code-ish UI, and `indexTiptap.css` (lines 631, 648). Really used.
- **Inter** — used ONLY as the `fontFamily: "Inter"` default for **Fabric.js canvas text objects** (`hooks/useFabric.ts:784`, `components/design/AssetDrawer.tsx` ×9, `PropertiesPanel.tsx`, `CanvasLeftSidebar.tsx` ×9, `CanvasRightSidebar.tsx` ×4). ⚠️ **Saved lessons store `fontFamily: "Inter"` inside their canvas JSON** — the family name "Inter" must remain resolvable or existing lesson canvases change appearance. Self-host it; do not rename it.
- **No font has Thai glyphs** — Thai text falls back to system fonts today. That stays true after 2.B (the Noto Sans Thai idea is PARKED, owner-gated — see §4).

### 2.4 Dead dependencies (verified by grep — zero imports anywhere in `src/`)
`lowlight`, `@tiptap/extension-code-block-lowlight` (StarterKit is configured with `codeBlock: false`), `cloudinary` (the *Node* SDK — the client's real upload code is the local `src/lib/cloudinary.ts`, which doesn't import it), and `tiptap` (a 2018-era v0.15 package; the real editor uses `@tiptap/*` v3; `tiptap-markdown` is a different, used package). These do NOT bloat the bundle (never imported → tree-shaken out entirely), so removing them changes no build numbers — it just removes confusion and install weight. Step A1 removes them with verification.

### 2.5 Careful-not-to-break (from ROADMAP §11, the ones Tier 2 can plausibly break)
| Thing | Why Tier 2 could break it | Guard |
| --- | --- | --- |
| **SSE streaming** (`tutorApi.ts` `callTutorStream` — deliberately fetch-based, bypasses axios) | It lives in the viewer chunk after splitting. Lazy loading must not double-load or re-evaluate module state. | Full tutor conversation with visible token streaming in the manual smoke (A7). |
| **Autosave + optimistic concurrency** (`PUT /content/:id` with `clientUpdatedAt`, 409 on conflict) | The editor is now a lazy chunk. | Edit a lesson, watch autosave fire, in A7. |
| **Structured 401 contract** (`lib/axios.ts` interceptor) | `axios.ts` stays in the entry chunk (imported by stores) — do not move it. | Nothing to do; just don't "optimize" store/lib imports. |
| **Viewer zoom via CSS `zoom`** | Only if you do stretch step A8 (CSS move). | Visual check of `/view/:id` in A8. |
| **Golden Rule 2** | A chunk-load failure on `/view/:id` for an anonymous user = an access regression. | Incognito smoke + the `vite:preloadError` reload guard (A3). |

---

## 3. Phase 2.A — Route-level code splitting

### Step A0 — Preflight
1. `cd client && npm install && npm test` → expect **74 passed** (or more if another tier landed tests first). If red, STOP — report, don't build on a broken base.
2. `git status --short` → note every already-modified file (expected: `src/components/editor/EditorLeftSidebar.tsx`, see §0 Git reality). These files are off-limits for your commits.
3. `npm run build` → record the full JS/CSS size table into §7 ("Before").

### Step A1 — Remove dead dependencies (recommended, low-risk)
1. Re-verify they are dead (must return no matches in `client/src`):
   ```bash
   # run from client/ — each must print nothing:
   grep -rE "from ['\"]lowlight|from ['\"]@tiptap/extension-code-block-lowlight|from ['\"]cloudinary['\"]|from ['\"]tiptap['\"]" src
   ```
2. `npm uninstall lowlight @tiptap/extension-code-block-lowlight cloudinary tiptap`
3. `npm test && npm run build` → both green, sizes unchanged (they were never bundled). If ANY grep in (1) matched, skip the matched package and note it in §7.

### Step A2 — Lazy routes in `App.tsx` + shared fallback

**New file `client/src/components/PageLoader.tsx`** (mirrors the existing `Loader2` idiom, e.g. `Profile.tsx:125`):
```tsx
import { Loader2 } from "lucide-react";

export function PageLoader() {
  return (
    <div className="flex min-h-screen items-center justify-center">
      <Loader2 className="h-6 w-6 animate-spin text-muted-foreground" />
    </div>
  );
}
```

**Rewrite the imports in `client/src/App.tsx`:** keep `Landing` and `NotFound` static (first paint + 404 must never flash a loader); keep the guard/layout imports (`ProtectedRoute`, `PublicRoute`, `RequireLogin`, `AppLayout`) static; convert the other 13 pages:

```tsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { lazy, Suspense, useEffect } from "react";
import { BrowserRouter, Routes, Route } from "react-router-dom";
import { useAuthStore } from "./stores/auth.store";

import ProtectedRoute from "./components/ProtectedRoute";
import PublicRoute from "./components/PublicRoute";
import { RequireLogin } from "./components/RequireLogin";
import { AppLayout } from "./layouts/AppLayout";
import { PageLoader } from "./components/PageLoader";

import Landing from "./pages/Landing";
import NotFound from "./pages/NotFound";

const TipTapCanvas = lazy(() => import("./pages/TipTapCanvas"));
const Login = lazy(() => import("./pages/Login"));
const Dashboard = lazy(() => import("./pages/Dashboard"));
const TiptapView = lazy(() => import("./pages/TiptapView"));
const Status = lazy(() => import("./pages/Status"));
const CloudinaryUpload = lazy(() => import("./pages/Cloudinaryupload"));
const Explore = lazy(() => import("./pages/Explore"));
const Profile = lazy(() => import("./pages/Profile"));
const Settings = lazy(() => import("./pages/Setting")); // file really is "Setting"
const Guide = lazy(() => import("./pages/Guide"));
const History = lazy(() => import("./pages/History"));
const Create = lazy(() => import("./pages/Create"));
const ChangePassword = lazy(() => import("./pages/ChangePassword"));
```

Wrap the existing `<Routes>` (do not reorder/edit any `<Route>` element or guard) in one `Suspense`, inside `BrowserRouter`:
```tsx
<BrowserRouter>
  <Suspense fallback={<PageLoader />}>
    <Routes>
      {/* …all existing routes exactly as they are… */}
    </Routes>
  </Suspense>
</BrowserRouter>
```
Everything else in `App.tsx` (QueryClient, `recheckToken` effect) stays byte-identical.

### Step A3 — Stale-deploy chunk guard in `client/src/main.tsx`
After a future redeploy, users with an open tab hold HTML that references old chunk hashes → lazy imports 404. Vite fires a `vite:preloadError` event for exactly this. Add to `main.tsx` (after the imports, before `createRoot`):
```ts
// After a redeploy, old chunk hashes 404 for already-open tabs — reload once to pick up the new build.
window.addEventListener("vite:preloadError", (event) => {
  event.preventDefault();
  window.location.reload();
});
```

### Step A4 — Vendor chunks in `client/vite.config.js`
Add a `build` key to the existing `defineConfig` object (touch nothing else in the file — including the `__dirname` line, see §0):
```js
build: {
  rollupOptions: {
    output: {
      // Pin the three huge editor vendors to stable named chunks so browser
      // caching survives app-code redeploys. They are only imported by lazy
      // routes, so they stay lazy. Do NOT add react/react-dom/router here.
      manualChunks(id) {
        if (!id.includes("node_modules")) return;
        if (/[\\/]node_modules[\\/]fabric[\\/]/.test(id)) return "fabric";
        if (/[\\/]node_modules[\\/]katex[\\/]/.test(id)) return "katex";
        if (/[\\/]node_modules[\\/](@tiptap|prosemirror-|tiptap-markdown)/.test(id)) return "tiptap";
      },
    },
  },
},
```
⚠️ **Escape hatch:** if the build errors or the dev/preview app white-screens with a chunk initialization error (circular-chunk symptom), delete the whole `manualChunks` function and rebuild — route splitting alone (A2) already satisfies the entry-size acceptance. Note the removal in §7 if you take it.

### Step A5 — Tests (this phase's acceptance criteria as code)

**New file `client/src/pages/__tests__/lazyRoutes.test.tsx`** — three tests:

1. **Static-import regression guard** (deterministic, no DOM): read `src/App.tsx` with `node:fs` and assert the only static `from "./pages/…"` imports are `Landing` and `NotFound`:
   ```ts
   import { readFileSync } from "node:fs";
   import path from "node:path";

   const appSrc = readFileSync(path.resolve(process.cwd(), "src/App.tsx"), "utf8");
   const staticPageImports = [...appSrc.matchAll(/^import .+ from "\.\/pages\/(\w+)"/gm)].map(m => m[1]);
   expect(staticPageImports.sort()).toEqual(["Landing", "NotFound"]);
   ```
2. **Eager landing renders with no Suspense round-trip:** render `<App />` (use the `createRoot` + `act` harness from `guards.test.tsx`; set the URL first with `window.history.pushState({}, "", "/")`) and assert Landing content is present synchronously (e.g. text from `pages/Landing.tsx` — `"Manga-style lessons"`).
3. **A lazy route resolves through Suspense:** `window.history.pushState({}, "", "/guide")`, render `<App />`, then
   ```ts
   await vi.waitFor(() => {
     expect(container!.textContent).toContain("Ready to start?"); // real string at pages/Guide.tsx:138
   });
   ```
   `/guide` is chosen because it's static content (no network). If it proves flaky in happy-dom, switch to `/login` and reuse the mocks from `pages/__tests__/Login.test.tsx`. Wrap renders in `act`; adapt to the existing harness patterns rather than installing anything new.

   ⚠️ happy-dom's default URL can be `about:blank`, which breaks `BrowserRouter` (tests 2–3 render the real `<App />`, unlike existing tests that use `MemoryRouter`). If pushState/route matching misbehaves, set a real URL first — happy-dom exposes `window.happyDOM.setURL("http://localhost/")` (call it in `beforeEach` before `pushState`).

`npm test` → all green (74 baseline + new).

### Step A6 — Build & measure
`npm run build`, record the full chunk table in §7 ("After 2.A"). Check:
- entry `index-*.js` no longer triggers the 500 kB warning (the lazy `tiptap`/`fabric` chunks may still exceed 500 kB — that's fine, they're lazy vendor code; **do not** raise `chunkSizeWarningLimit` to silence them, just list them in §7);
- entry gzip ≤ **323 kB** (expect far less);
- named chunks `tiptap-*.js`, `fabric-*.js`, `katex-*.js` exist;
- a `Cloudinaryupload`/`TipTapCanvas`/`TiptapView` chunk each exist;
- KaTeX CSS moved out of the entry CSS (entry `index-*.css` should drop vs. the 226 kB baseline).

### Step A7 — Manual smoke (server on :5000, client `npm run dev` on :5173)
With DevTools Network open (Disable cache):
1. **`/` and `/explore`:** confirm **no** `tiptap`/`fabric`/`katex` chunk is fetched.
2. **`/view/69e39d0b60d467bd515a4945` logged-out (incognito):** lesson renders (canvases, formulas, question cards), then run a **full tutor conversation** — streaming tokens appear live, suggestion chips render and are tappable, question-card feedback works. This is Golden Rule 2 + the SSE careful-not-to-break in one pass.
3. **Same lesson logged in:** repeat a short tutor exchange.
4. **`/canvas/:id` (editor, needs the owner's teacher login — ask the owner to drive this one if you have no credentials):** open a lesson, type, watch autosave fire (Network: `PUT /content/:id`), draw on a canvas, insert a formula.
5. **`/uploadimage`, `/settings`, `/status`, `/login`:** each renders after a brief `PageLoader` spinner, no console errors.
6. Phone pass: DevTools ~390 px on `/` and `/view/:id` — unchanged layout.

### Step A8 — OPTIONAL stretch (only if A0–A7 are all green): move `indexTiptap.css` out of the entry
`indexTiptap.css` (~660 lines of editor-only styles) is imported in `main.tsx`, so every visitor downloads it. Move it into the editor graph:
1. Delete the `import "./indexTiptap.css";` line from `src/main.tsx`.
2. Add `import "@/indexTiptap.css";` at the top of **both** `src/components/editor/TipTapEditor.tsx` and `src/components/editor/TiptapViewer.tsx` (Vite dedupes; the CSS attaches to the lazy chunks).
3. Rebuild + re-run smoke items 2 and 4 (viewer and editor must look pixel-identical; check the follow-up chat threads and formula blocks specifically), and confirm `/` renders unchanged.
4. **If anything looks off, revert this step entirely** — it's a bonus, not an acceptance criterion. Record either way in §7.

### Step A9 — Docs + commit
1. Update `client/CLAUDE.md`: in the routing section, note that pages are `React.lazy` (except Landing/NotFound), the `PageLoader` fallback, the `vite:preloadError` reload guard in `main.tsx`, and the `manualChunks` vendor pinning in `vite.config.js` (and the A1 dep removals / A8 CSS move if taken).
2. `cd client`, then stage **only the files you touched, by explicit path** (see §0 Git reality — the pre-existing `EditorLeftSidebar.tsx` edit must stay out): `git add src/App.tsx src/main.tsx src/components/PageLoader.tsx src/pages/__tests__/lazyRoutes.test.tsx vite.config.js package.json package-lock.json CLAUDE.md` (plus the two editor files if you took A8), then `git commit -m "perf(tier2.A): route-level code splitting + vendor chunks"` (adjust message if you skipped/added steps). **Do not push.**

**Phase 2.A acceptance (from ROADMAP):** entry JS for `/` + `/explore` ≤ 323 kB gzip; entry chunk-size warning gone; every route renders; editor + viewer + full tutor conversation smoke-tested; `npm test` green.

---

## 4. Phase 2.B — Self-host fonts (kill the Google Fonts CDN)

**Strategy: keep every font family byte-identical, just serve it ourselves.** That makes the roadmap's "owner eyeballs the look" gate trivial — nothing changes visually. (The only *look-changing* idea, a Thai-capable face, is parked below.)

### Step B1 — Install the three used families (see §2.3 for why exactly these)
```bash
cd client && npm i @fontsource/lora @fontsource/jetbrains-mono @fontsource/inter
```
(Static packages, not `-variable`, so the family names stay exactly `"Lora"`, `"JetBrains Mono"`, `"Inter"` — critical for Inter, whose name is baked into saved lesson canvas JSON. **DM Sans is intentionally not replaced** — it was loaded but never used.)

### Step B2 — `client/index.html`
Delete exactly these three lines (and nothing else in the file — see §0 for the tags you must leave alone):
```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&family=DM+Sans:wght@400;500;600;700&display=swap" rel="stylesheet">
```

### Step B3 — `client/src/index.css`
1. Delete line 1 (the `@import url("https://fonts.googleapis.com/css2?family=Lora…JetBrains+Mono…")`).
2. Next to the existing `@import "@fontsource-variable/geist";` (line ~9), add:
   ```css
   @import "@fontsource/lora/400.css";
   @import "@fontsource/lora/600.css";
   @import "@fontsource/lora/700.css";
   @import "@fontsource/jetbrains-mono/400.css";
   @import "@fontsource/jetbrains-mono/500.css";
   @import "@fontsource/inter/400.css";
   @import "@fontsource/inter/500.css";
   @import "@fontsource/inter/600.css";
   @import "@fontsource/inter/700.css";
   ```
   (Weights chosen from real usage: Lora renders `font-serif` headings at regular/semibold/bold; JetBrains Mono was loaded at 400/500; Inter covers canvas text incl. bold. All `@import` lines must stay **above** `@custom-variant`/`@theme` — CSS requires imports first.)
3. Do **not** touch the `--font-*` tokens — family names are unchanged.

### Step B4 — Regression-guard test
**New file `client/src/__tests__/no-cdn-fonts.test.ts`:** read `index.html` and `src/index.css` (via `node:fs`, `path.resolve(process.cwd(), …)`) and assert neither contains `fonts.googleapis.com` or `fonts.gstatic.com`. This is the phase's acceptance criterion as code — it stops anyone reintroducing a CDN font in a copy-paste.

### Step B5 — Verify
1. `npm test` green; `npm run build` green — Lora/JetBrains Mono/Inter woff2 files now appear in `dist/assets/`.
2. `grep -ri "googleapis\|gstatic" dist/assets/*.css dist/index.html` → **no matches**.
3. `npm run dev`, DevTools Network filtered to "font": zero requests to `fonts.googleapis.com`/`fonts.gstatic.com`; woff2s come from localhost. In the console: `document.fonts.check('16px Lora')` → `true` after loading a page with a serif heading (e.g. `/explore`).
4. Eyeball `/` (serif hero heading), `/status` (mono text), and a lesson canvas with text (Inter) — identical to before. Record entry CSS/font sizes in §7.

### Step B6 — Docs + commit
1. `client/CLAUDE.md`: note fonts are fully self-hosted via `@fontsource` (Geist Variable = UI sans, Lora = serif, JetBrains Mono = mono, Inter = Fabric-canvas text only, name must never change), no CDN fonts allowed (test-enforced), DM Sans removed as unused.
2. `cd client`, stage by explicit path again: `git add index.html src/index.css src/__tests__/no-cdn-fonts.test.ts package.json package-lock.json CLAUDE.md`, then `git commit -m "perf(tier2.B): self-host fonts, drop Google Fonts CDN"`. **Do not push.**

### PARKED — Thai-capable UI font (do NOT implement)
Adding e.g. `@fontsource/noto-sans-thai` to the `--font-sans`/`--font-serif` stacks would change how all Thai text renders — **owner must see and approve first** (it changes the app's face). Out of scope for this execution; leave Thai on system-font fallback exactly as today. If the owner asks later: install the package, add it to the stacks after the Latin face (`"Geist Variable", "Noto Sans Thai", sans-serif`), and get a look-approval before committing.

**Phase 2.B acceptance (from ROADMAP):** no requests to `fonts.googleapis.com`/`gstatic.com`; visuals unchanged (identical families ⇒ trivially approved); build + tests green.

---

## 5. Final whole-tier checklist
- [ ] `cd client && npm test` — green (74 baseline + new lazy-route tests + font test).
- [ ] `cd client && npm run build` — green; §7 table filled with before/after; entry ≤ 323 kB gzip (expect ~150–250).
- [ ] `/` + `/explore` fetch no tiptap/fabric/katex chunks; no CDN font requests anywhere.
- [ ] Full tutor conversation (streaming + chips + question feedback) works on `/view/69e39d0b60d467bd515a4945` **logged-in and incognito**.
- [ ] Editor autosave (`PUT /content/:id`) observed working.
- [ ] ~390 px phone pass on `/` and `/view/:id`.
- [ ] `client/CLAUDE.md` updated; two commits in `client/` (A, B); **nothing pushed**.
- [ ] No server files touched; `index.html` script tag and `vite.config.js` `__dirname` left as found.

## 6. Failure playbook
| Symptom | Likely cause | Fix |
| --- | --- | --- |
| White screen / `Cannot access X before initialization` in preview | `manualChunks` created a circular chunk | Delete the `manualChunks` function (escape hatch in A4) |
| A route 404s its chunk in dev after a rebuild | stale HTML in an open tab | expected in prod (guarded by A3); in dev just hard-reload |
| `vi.waitFor` test flaky on `/guide` | happy-dom async timing | switch the lazy-route test to `/login` with `Login.test.tsx`'s mocks |
| Serif/mono text looks different after 2.B | a weight you didn't import is being synthesized | add the missing `@fontsource/<family>/<weight>.css` import |
| Canvas text in old lessons changed | Inter family name broken | you used a `-variable` package or renamed the family — go back to `@fontsource/inter` statics (§4 B1) |

## 7. Phase notes (implementing agent fills this in)

### Before (preflight A0)
| Asset | Raw | Gzip |
| --- | --- | --- |
| `dist/assets/index-*.js` (single JS chunk) | 2,100.55 kB | 647.62 kB |
| `dist/assets/index-*.css` | 226.86 kB | 41.50 kB |
| Build warning | entry > 500 kB | yes |

### After 2.A
| Asset | Raw | Gzip |
| --- | --- | --- |
| `dist/assets/index-*.js` (entry) | 418.76 kB | **137.38 kB** |
| `dist/assets/tiptap-*.js` | 587.90 kB | 198.07 kB (lazy, >500 kB OK) |
| `dist/assets/fabric-*.js` | 285.60 kB | 86.63 kB |
| `dist/assets/katex-*.js` | 258.80 kB | 76.91 kB |
| `dist/assets/TipTapCanvas-*.js` | 159.74 kB | 41.15 kB |
| `dist/assets/TiptapView-*.js` | 11.00 kB | 4.23 kB |
| `dist/assets/Cloudinaryupload-*.js` | 16.02 kB | 4.31 kB |
| `dist/assets/index-*.css` (entry) | 105.40 kB | 17.46 kB |
| `dist/assets/editorExtensions-*.css` | 95.86 kB | 16.61 kB |
| `dist/assets/katex-*.css` | 29.27 kB | 8.05 kB |
| Entry chunk-size warning | gone | — |

**Steps taken:** A1 removed `lowlight`, `@tiptap/extension-code-block-lowlight`, `cloudinary`, `tiptap`. A8 taken — `indexTiptap.css` moved to `TipTapEditor.tsx` + `TiptapViewer.tsx`. `manualChunks` kept (no circular-chunk issues).

### After 2.B
| Asset | Raw | Gzip |
| --- | --- | --- |
| `dist/assets/index-*.css` (entry, incl. font faces) | 149.43 kB | 34.56 kB |
| Self-hosted woff2 | Lora, JetBrains Mono, Inter, Geist Variable | in `dist/assets/` |
| `grep googleapis\|gstatic` on `dist/` | no matches | — |

### Deviations from this plan
- **History.test.tsx midnight flakiness:** pinned `vi.setSystemTime` + fixed `last_accessed` ISO (test used `Date.now()` at module load; failed at 00:00 and produced negative minutes). Not tier-2 scope but required for green `npm test` baseline.
- **npm SSL on Windows:** `@fontsource/*` install needed `NODE_OPTIONS=--use-system-ca` (UNABLE_TO_VERIFY_LEAF_SIGNATURE).
