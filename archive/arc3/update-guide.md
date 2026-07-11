# update-guide.md — Refreshing the guide pages (brand rename + re-render)

For a **future agent** (or the owner) who needs to update the `/guide` walkthroughs once the product is finished — most likely to **change the brand name** and **re-render every screenshot**, while keeping the written content exactly the same.

Read the root [`CLAUDE.md`](CLAUDE.md) and [`client/CLAUDE.md`](client/CLAUDE.md) first (git layout, Golden Rules). The full design/spec of the guide is in [`ROADMAP-guide.md`](ROADMAP-guide.md) + [`plan/guide.md`](plan/guide.md); this file is just the **maintenance recipe**.

> ⚠️ **Git reality (do not forget):** `client/` and `server/` are separate git repos; `Hot-Potato/` is NOT a repo; the home dir `C:\Users\BTCOM` is itself a stray git repo. **Always run git as `git -C client …`** — a `git add` from `Hot-Potato/` once landed a commit in the wrong repo. Never push until the ROADMAP.md Tier 5 launch pass (owner decision D1).

---

## 0. The mental model (read this before touching anything)

The guide is split into **content** and **images**, on purpose:

| Layer | Where | Changes when… |
| --- | --- | --- |
| **Scene copy** (titles, steps, tips, alt text) | `client/src/pages/guide/learningScenes.ts` (9 scenes) + `creatingScenes.ts` (10 scenes), as `{en, th}` data | you want to reword the guide |
| **Screenshots** | `client/public/guide/*.webp` (static files, referenced by URL — **never imported**) | the app's UI changed, or the brand name changed |
| **Image dimensions** | `client/src/pages/guide/guideImages.ts` (**auto-generated**) | regenerated automatically by the capture script |
| **Which image belongs to which scene** | the `images:` array inside each scene in the `*Scenes.ts` files | the manifest changed |

**Re-rendering the screenshots does NOT touch the copy.** The two `capture-guide.mjs` commands only rewrite the `.webp` files and regenerate `guideImages.ts`. So "keep the content the same, just re-render" = run the pipeline and commit — the scene text is untouched. ✅

---

## 1. Rename the brand

The product ships with a working name **"Hot Potato"** but the UI art still says **"Intuita"** (the owner had not picked a final brand as of 2026-07-11). When the real name is chosen, change it in **all** of these, then re-render (Part 2) so the screenshots pick up the new chrome:

| # | File | What to change | Shows up in… |
| --- | --- | --- | --- |
| 1 | `client/src/lib/brand.ts` | `export const BRAND_NAME = "…"` | the `/guide` hub H1 (English side: "How to use {BRAND_NAME}") + any new copy that imports it |
| 2 | `client/src/pages/Landing.tsx` | the `<span>…Intuita…</span>` wordmark (search `Intuita`) | **`learning-01-landing.webp`** (visible text — must re-render) |
| 3 | `client/src/components/TopNav.tsx` | `alt="Intuita"` on the logo `<img>` (search `Intuita`) | the top bar of nearly every screenshot (alt text only; the visible mark is the icon asset — see #4) |
| 4 | `client/src/assets/logo.png` | the actual logo icon, **only if** the new brand has new art | the top-left icon in every screenshot |

After changing 1–4, **grep to be sure nothing is left**:

```bash
cd client && grep -rn "Intuita" src/    # expect: zero hits (or only a code comment you intend to keep)
```

Then go to Part 2 and re-render — otherwise the old name stays baked into the screenshots.

> The **scene copy** deliberately avoids hardcoding the brand (it says "เว็บนี้" / "this site"), so you normally do **not** need to edit `learningScenes.ts` / `creatingScenes.ts` for a rename. Double-check with `grep -n "Intuita\|Hot Potato" src/pages/guide/*Scenes.ts` and only edit if a hit appears.

---

## 2. Re-render the screenshots

### 2.1 Prerequisites

- **Both halves running locally**, pointed at whatever DB you want the demo data in:
  - `cd server && npm run dev` → `http://localhost:5000` (needs a **real `GEMINI_API_KEY`** in `server/.env` — some scenes call Gemini for real).
  - `cd client && npm run dev` → `http://localhost:5173`.
- The owner often leaves these running already — check before starting new ones. **Vite ignores an injected `PORT`**, so if `:5173` is taken the capture will hit the wrong page; pass `--base http://localhost:<port>` to both scripts if the client is elsewhere.
- Playwright's Chromium installed: `cd client && npx playwright install chromium` (one-time; ~110 MB).

### 2.2 The two commands

```bash
cd client
node scripts/seed-guide-demo.mjs      # 1. create/refresh demo data → writes scripts/guide-demo.json
node scripts/capture-guide.mjs        # 2. capture all shots → public/guide/*.webp + regenerate guideImages.ts
```

That's the whole refresh. `capture-guide.mjs` reads `guide-demo.json` for the lesson ids, drives headless Chromium at `deviceScaleFactor: 2`, and writes WebP (q82) into `public/guide/`, then rebuilds `guideImages.ts` from what's on disk.

**Partial re-render** (after a targeted UI change): `node scripts/capture-guide.mjs --scene creating-06` captures just the matching scene(s) and keeps every other image + its `guideImages.ts` entry.

### 2.3 What the seed makes (and cleanup candidates)

`seed-guide-demo.mjs` logs in or registers two demo accounts and upserts their lessons (docs live in `scripts/guide-demo-docs.mjs`, shared with the capture script):

| Account / lesson | Role in the shots | Notes |
| --- | --- | --- |
| `guide.teacher@hotpotato.local` | owns the teacher lessons | password default `HotPotato-guide-2026` (in the script header; override with `--password` or `GUIDE_DEMO_PASSWORD`) — **never committed** |
| `guide.student@hotpotato.local` | history + tutor-memory shots | |
| Demo lesson "ทำไมท้องฟ้าเป็นสีฟ้า?" (`link-only`) | all **learning** shots — has every question type + a formula block | |
| **Scratch** lesson "แรงและการเคลื่อนที่ (ฉบับร่างเดโม)" (`private`) | the **creating** editing shots | the capture script resets it to a baseline doc between shots via `resetScratch()`, so runs are deterministic |
| **Blank** untitled lesson (`private`) | `creating-02-blank-editor` (empty-doc AI CTA) | |
| 4 vault images | `creating-05a-media` grid | seeded from `placehold.co` URLs — **the server validates image URLs with a HEAD request**, and most photo hosts (picsum, wikimedia) reject HEAD; `placehold.co` answers it, so keep using it |

All of the above are safe to delete from the DB afterward. The ids land in `scripts/guide-demo.json` (committed; no secrets).

`--skip-ai` on the **seed** skips the 4 tutor turns that build the student's memory (saves tokens); the memory shot (`learning-08b`) will then be empty, so only use it when you don't need that scene.

### 2.4 AI-gated scenes (cost real Gemini tokens)

Six scenes call Gemini for real — rerun them sparingly. If Gemini hiccups, each falls back to a still-usable pre-AI screenshot (the script catches the error), so a failure never aborts the run — just re-run that `--scene`:

`learning-05-feedback`, `learning-06-askai`, `creating-06-formula`, `creating-07b-question-creator`, `creating-08b-draft-preview`, `creating-09a-critic`.

### 2.5 Scene ↔ image map (current)

Learning (phone, portrait): `learning-01-landing` · `-02-explore` · `-03-viewer` · `-04a-choice`/`-04b-write`/`-04c-blankdrag` · `-05-feedback` · `-06-askai` · `-07-personality` · `-08a-history`/`-08b-memory` · `-09-settings`.

Creating (desktop, `wide: true`): `creating-01-create-page` · `-02-blank-editor` · `-03-editor-overview` · `-04-writing` · `-05a-media` · `-06-formula` · `-07a-question-panel`/`-07b-question-creator` · `-08a-ai-hub`/`-08b-draft-preview` · `-09a-critic`/`-09b-publish-modal` · `-10-share`.

> **Known orphan:** `creating-05b-canvas.webp` is still captured by the manifest and on disk, but scene 5 was simplified to images-only, so **no scene references it** anymore. Harmless (static file, never imported). If you want it tidy, either drop the `creating-05b-canvas` entry from the `SCENES` array in `capture-guide.mjs` (and delete the `.webp`), or add it back into `creatingScenes.ts` scene 5's `images`.

---

## 3. If a screenshot comes out wrong (UI moved)

The capture drives the real UI, so a moved/renamed control breaks a selector. The script prints `▸ <file> … FAILED — <reason>` per scene and exits non-zero. Iterate with `--scene <id>` and fix the selector in `capture-guide.mjs`. Patterns that this codebase relies on (learned the hard way):

- **Left-rail categories** are clicked by Thai label: `page.getByTitle("สื่อ", { exact: true })` (labels: `AI` · `ข้อความ` · `สื่อ` · `สูตร` · `คำถาม`).
- **Editor blocks** (formula, question) are located as `page.locator("[data-node-view-wrapper]").filter({ hasText: /…/ })` — **not** by `data-type`.
- **`AiDraftDialog`'s "ร่างโครง"** names *both* the tab and the submit button — use `.last()` for the submit.
- **Thai search gotcha:** the public smoke lesson title "การเคลื่อนที่เเละเเรง" spells แ as **เ + เ** (two characters). Search by a safe substring, not "แรง".
- **Element vs viewport shot:** a scene's `run(page)` returns a Playwright locator for an element screenshot, or `null` for a full-viewport shot.
- **Editor is desktop-only** (mobile gate below `md`) — creating scenes already use the `desktop` viewport in the manifest; keep it.

---

## 4. Verify → commit → (later) launch

After re-rendering (and any rename), from `client/`:

```bash
npm test          # expect all green (guide tests pin scene counts + assert every img is /guide/… and lazy)
npm run lint      # no new errors
npm run build     # entry chunk must stay ~138 kB gzip; each showcase is its own lazy chunk
```

Eyeball the new `.webp`s (open the ones you changed) and, if you touched layout, do a quick headless pass at 390 px + desktop + dark mode to confirm no horizontal scroll and no broken images.

Commit in the **client repo only**, phase-style, and **do not push**:

```bash
git -C client add -A
git -C client commit -m "chore(guide): re-render screenshots (+ brand rename)"
```

**Launch rider:** at the ROADMAP.md Tier 5 deploy pass, run the two pipeline commands (§2.2) once more **against the production DB** — the demo/scratch/blank lesson ids differ per database, so `guide-demo.json` (and therefore the shots) must be regenerated there. Then commit + deploy the fresh images.

---

## 5. Guardrails (don't break these)

- **Bundle budget:** guide images live in `public/guide/` and are referenced by URL — **never `import`** them; keep both showcase pages lazy routes. Tests in `client/src/pages/__tests__/` guard this.
- **Golden Rule 2:** `/guide`, `/guide/learning`, `/guide/creating` are **public** — no `protect`, no login wall. The login scene *describes* login; it never requires it.
- **Read-only over the app:** the guide documents the app; refreshing it must not modify editor/tutor/store code. If a screenshot reveals a real bug, file it separately.
- **Bilingual + phone-first:** every string goes through `useAppI18n` `t(en, th)`; judge every layout at 390 px first (the `wide` teacher scenes still must not h-scroll on a phone).
