# Tier 0 Execution Plan — SEO & Link Previews (Phase 0.A)

> **For the implementing agent (Cursor).** This plan was written 2026-07-10 by Claude after verifying every fact below against the actual code. Follow it literally — exact file contents are provided. Where this plan says "exact", paste, don't improvise. If reality differs from the "Verified facts" section, **stop and report to the owner** instead of adapting silently.
>
> Source of truth for scope: [`ROADMAP.md`](../ROADMAP.md) §4 (Tier 0, Phase 0.A). Read [`CLAUDE.md`](../CLAUDE.md) and [`client/CLAUDE.md`](../client/CLAUDE.md) before starting.

---

## 0. Ground rules (read twice)

1. **All work happens inside `client/`.** This phase touches zero server code. `cd client` first; run every command from there.
2. **Git quirk:** `client/` is its own git repo. **Never** run git from `Hot-Potato/` or above (a stray repo at the user's home dir will show foreign files — if `git status` shows anything like `ChronoForge-FPGA-Engine`, you are in the wrong directory; stop).
3. **Do NOT push.** Owner decision D1: nothing is pushed until Tier 5. Commit locally only.
4. **Do NOT touch the Google Fonts `<link>` lines** in `index.html`. Self-hosting fonts is Tier 2.B, a separate owner-gated phase. Preserve those three lines byte-for-byte.
5. **Do NOT add environment variables** (no `%VITE_SITE_URL%` tricks in index.html). ROADMAP §12: no new env vars through Tier 5.
6. **Do NOT fix the known lint error** (`__dirname` no-undef in `client/vite.config.js`). That is Tier 4's job. `npm run lint` showing exactly that one error is the expected state.
7. **Do NOT install any npm packages.** This phase needs none.
8. **Scope is site-wide tags only.** Per-lesson OG tags (`/view/:id` showing the lesson's own title/image) are impossible in a client-rendered SPA without SSR/prerendering — that is deliberately parked as ROADMAP Tier 6.D. Do not attempt it.

---

## 1. Verified facts (do not re-explore; trust these)

Checked 2026-07-10 against the working tree:

| Fact | Value |
| --- | --- |
| `client/index.html` today | 18 lines: `lang="en"`, favicon `<link rel="icon" type="image/svg+xml" href="./src/assets/logo.png">` (PNG mislabeled as SVG, pointing into `src/`), 3 Google-Fonts lines, `<title>Hot Potato</title>`, **zero** description/OG/twitter tags, and `<script type="module" src="/src/main.jsx">` |
| Real entry file | `client/src/main.tsx` (**exists**; `main.jsx` does **not** — Vite resolves it by fallback luck) |
| Logo | `client/src/assets/logo.png`, **584×584 px** square PNG, ~160 KB. No 1200×630 OG card exists anywhere → use `twitter:card = summary` (square), **not** `summary_large_image` |
| `client/public/` today | contains only `vite.svg` (Vite template leftover; **grep confirmed nothing references it** — safe to delete; it **is** git-tracked, so its deletion must be staged in Step 8) |
| `client/` git state at plan time | clean working tree, branch `main` |
| Production URL | **Unknown/not recorded anywhere in the repo.** Nothing is deployed from this codebase yet. Therefore `og:image` stays **root-relative** (`/og-image.png`) with a `TODO(Tier 5)` comment; `og:url`/canonical are deliberately omitted until launch |
| Vercel + robots.txt | `client/vercel.json` = `{"rewrites":[{"source":"/(.*)","destination":"/index.html"}]}`. Vercel serves real files from `public/` **before** applying rewrites, so `public/robots.txt`, `public/favicon.png`, `public/og-image.png` will be served at the site root correctly. Same for `vite dev` and `vite build` (public/ → dist/ root). No vercel.json change needed |
| Runtime `lang` | `client/src/stores/language.store.ts` (line ~16) already sets `document.documentElement.lang` on load and on toggle, defaulting to `"th"`. The static `lang="th"` we set is the crawler-facing default; the store legitimately overwrites it at runtime. **Do not change the store** |
| Brand color | `--primary: hsl(252 75% 58%)` in `client/src/index.css` ≈ `#6444e4` (used for `theme-color`) |
| Test setup | Vitest + happy-dom, config `client/vitest.config.ts`, default include pattern (any `*.test.ts`). Existing suite: 66 tests green. Tests live in `__tests__` folders next to code; a top-level `client/src/__tests__/` does not exist yet — creating it is fine |
| Build status | `npm run build` green with **one expected warning**: a ~2,095 kB chunk ("Some chunks are larger than 500 kB"). That is Tier 2's problem. Ignore it |

---

## 2. Task list (in order)

### Step 1 — Preflight baseline

```powershell
cd client
git status          # expect a clean tree on main; if dirty, STOP and report
npm test            # expect 66/66 green
npm run build       # expect green (with the known big-chunk warning)
```

If any of these fail **before you changed anything**, stop and report — do not fix unrelated breakage.

### Step 2 — Static assets in `public/`

From inside `client/`:

```powershell
Copy-Item src/assets/logo.png public/favicon.png
Copy-Item src/assets/logo.png public/og-image.png
Remove-Item public/vite.svg
```

Why copies instead of referencing `src/assets/logo.png`: Vite fingerprints `src/` assets (hashed filenames that change every build). Crawlers and browsers need **stable root URLs** — that's what `public/` is for.

### Step 3 — Replace `client/index.html` (exact content)

Replace the entire file with exactly this (the three Google-Fonts lines are preserved verbatim from the current file):

```html
<!doctype html>
<html lang="th">

<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  <link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&family=DM+Sans:wght@400;500;600;700&display=swap" rel="stylesheet">

  <title>Hot Potato — เรียนรู้อย่างเข้าใจ กับติวเตอร์ AI</title>
  <meta name="description"
    content="Hot Potato — แพลตฟอร์มบทเรียนออนไลน์ที่คุณครูสร้างเอง พร้อมติวเตอร์ AI ใจดีคอยชวนคิด ชวนตั้งคำถาม ใช้ฟรีไม่ต้องเข้าสู่ระบบ" />

  <!-- Open Graph — site-wide only. Per-lesson OG needs SSR/prerender (ROADMAP Tier 6.D); see client/CLAUDE.md. -->
  <meta property="og:type" content="website" />
  <meta property="og:site_name" content="Hot Potato" />
  <meta property="og:title" content="Hot Potato — เรียนรู้อย่างเข้าใจ กับติวเตอร์ AI" />
  <meta property="og:description"
    content="Hot Potato — แพลตฟอร์มบทเรียนออนไลน์ที่คุณครูสร้างเอง พร้อมติวเตอร์ AI ใจดีคอยชวนคิด ชวนตั้งคำถาม ใช้ฟรีไม่ต้องเข้าสู่ระบบ" />
  <meta property="og:locale" content="th_TH" />
  <!-- TODO(Tier 5 launch): make og:image + twitter:image ABSOLUTE with the production URL
       (e.g. https://<prod-domain>/og-image.png) — Facebook's crawler does not resolve relative URLs reliably. -->
  <meta property="og:image" content="/og-image.png" />
  <meta property="og:image:width" content="584" />
  <meta property="og:image:height" content="584" />

  <!-- Twitter/X — "summary" (square) because the only art is the 584×584 logo. Upgrade to
       summary_large_image only if a real 1200×630 card lands in public/. -->
  <meta name="twitter:card" content="summary" />
  <meta name="twitter:title" content="Hot Potato — เรียนรู้อย่างเข้าใจ กับติวเตอร์ AI" />
  <meta name="twitter:description"
    content="Hot Potato — แพลตฟอร์มบทเรียนออนไลน์ที่คุณครูสร้างเอง พร้อมติวเตอร์ AI ใจดีคอยชวนคิด ชวนตั้งคำถาม ใช้ฟรีไม่ต้องเข้าสู่ระบบ" />
  <meta name="twitter:image" content="/og-image.png" />

  <meta name="theme-color" content="#6444e4" />

  <link rel="icon" type="image/png" href="/favicon.png" />
  <link rel="apple-touch-icon" href="/favicon.png" />
</head>

<body>
  <div id="root"></div>
  <script type="module" src="/src/main.tsx"></script>
</body>

</html>
```

Notes on deliberate choices (don't "improve" them):
- `og:url` and `<link rel="canonical">` are **omitted on purpose** — there is no canonical production URL until Tier 5; crawlers fall back to the fetched URL.
- The Thai copy above is the owner-facing default. If the owner supplies different wording, use theirs; otherwise ship this.
- `lang="th"` matches the app's Thai-first default (`language.store.ts` default is `"th"`).

### Step 4 — Create `client/public/robots.txt` (exact content)

```
User-agent: *
Allow: /
```

(No sitemap line — no sitemap exists; an SPA with 15 routes doesn't need one yet.)

### Step 5 — Add the test file `client/src/__tests__/seo.test.ts` (exact content)

The working agreement: every phase ships tests for its own acceptance criteria. These tests pin the static contract so a future refactor can't silently drop the tags.

```ts
import { describe, expect, it } from "vitest";
import { existsSync, readFileSync } from "node:fs";
import { fileURLToPath } from "node:url";

// Resolve files relative to this test file so the suite works regardless of cwd.
const p = (rel: string) => fileURLToPath(new URL(rel, import.meta.url));
const read = (rel: string) => readFileSync(p(rel), "utf-8");

const html = read("../../index.html");

describe("index.html SEO / link-preview tags (Tier 0.A)", () => {
  it("declares Thai as the document language", () => {
    expect(html).toContain('<html lang="th">');
  });

  it("has a non-empty meta description", () => {
    const m = html.match(/<meta name="description"\s+content="([^"]+)"/);
    expect(m?.[1]?.trim().length ?? 0).toBeGreaterThan(20);
  });

  it("has the core Open Graph tags", () => {
    expect(html).toContain('property="og:type" content="website"');
    expect(html).toContain('property="og:title"');
    expect(html).toContain('property="og:description"');
    expect(html).toMatch(/property="og:image" content="[^"]*og-image\.png"/);
  });

  it("has a twitter card", () => {
    expect(html).toMatch(/name="twitter:card" content="summary(_large_image)?"/);
  });

  it("declares the favicon as PNG served from public/", () => {
    expect(html).toContain('<link rel="icon" type="image/png" href="/favicon.png"');
    expect(html).not.toContain('image/svg+xml');
  });

  it("loads the real entry module (main.tsx, not main.jsx)", () => {
    expect(html).toContain('src="/src/main.tsx"');
    expect(html).not.toContain("main.jsx");
  });
});

describe("public/ static SEO files (Tier 0.A)", () => {
  it("robots.txt exists and allows all crawlers", () => {
    const robots = read("../../public/robots.txt");
    expect(robots).toMatch(/User-agent: \*/);
    expect(robots).toMatch(/Allow: \//);
  });

  it("og-image.png and favicon.png exist in public/", () => {
    expect(existsSync(p("../../public/og-image.png"))).toBe(true);
    expect(existsSync(p("../../public/favicon.png"))).toBe(true);
  });
});
```

### Step 6 — Document the SPA limitation in `client/CLAUDE.md`

Append this section to `client/CLAUDE.md`, directly **before** the `## Gotchas` section (exact text):

```md
## SEO & link previews (Tier 0.A, 2026-07-10)

- `index.html` carries the **site-wide** meta/OG/Twitter tags (Thai description; `og:image` → `/og-image.png`; `twitter:card = summary` because the only art is the square 584×584 logo). `public/robots.txt` allows all crawlers. `public/favicon.png` and `public/og-image.png` are copies of `src/assets/logo.png` served at stable root URLs — never point crawlers at `src/assets/*` (Vite fingerprints those paths, they change every build).
- **Known limitation (deliberate):** this is a client-rendered SPA and social crawlers don't run JS, so **every `/view/:id` lesson link shares the same generic site card**. Real per-lesson OG tags need SSR/prerendering/edge functions — parked as ROADMAP Tier 6.D. Don't try to "fix" it inside `index.html`.
- `og:image`/`twitter:image` are **root-relative until launch**. Tier 5 launch step: switch them to the absolute production URL and validate with the Facebook Sharing Debugger / LINE / Discord paste. `og:url` + canonical are also deferred to Tier 5 for the same reason.
- The static `lang="th"` is the crawler-facing default; `language.store.ts` overwrites `document.documentElement.lang` at runtime when the user toggles language — both are correct, don't unify them.
```

### Step 7 — Verify

All from inside `client/`:

1. `npm test` → **all green** (66 existing + the 8 new SEO tests).
2. `npm run build` → green; then confirm the artifacts landed:
   ```powershell
   Get-ChildItem dist/robots.txt, dist/favicon.png, dist/og-image.png
   Select-String -Path dist/index.html -Pattern "og:title", "main.jsx"
   ```
   `og:title` must match; `main.jsx` must return **nothing**.
3. `npm run dev` → open `http://localhost:5173`:
   - View source: all meta tags present, `lang="th"`.
   - The potato logo shows as the tab favicon.
   - `http://localhost:5173/robots.txt` returns the file (not the SPA).
   - Landing (`/`) and Explore (`/explore`) render exactly as before, no console errors.
   - If the API server happens to be running on :5000, also open the test lesson `/view/69e39d0b60d467bd515a4945` and confirm it loads identically. If the server isn't running, skip this — nothing in this phase can affect it (the check exists to satisfy the ROADMAP acceptance line "lesson pages still load identically").
4. `npm run lint` → still exactly **one** error (`__dirname` in vite.config.js). Zero new errors.
5. Real share-preview debuggers (LINE/Facebook/Discord) **cannot fetch localhost** — that validation is explicitly deferred to the Tier 5 production smoke. Passing steps 1–4 completes this phase's local acceptance.

### Step 8 — Commit (locally, no push)

From inside `client/` only:

```powershell
git add index.html public/robots.txt public/favicon.png public/og-image.png public/vite.svg src/__tests__/seo.test.ts CLAUDE.md
# note: public/vite.svg was tracked; adding it stages its DELETION (verified — this is correct)
git status                                 # review: exactly these files, nothing else
git commit -m "feat(seo): site-wide meta/OG tags, robots.txt, favicon + entry-script fix (Tier 0.A)"
```

**Do not `git push`** (owner decision D1). Do not commit from any directory above `client/`.

### Step 9 — Fill in the Execution log

Complete the log at the bottom of this file so the owner can audit the run.

---

## 3. Acceptance checklist (mirror of ROADMAP §4)

- [x] `index.html` has `lang="th"`, Thai meta description, `og:title/og:description/og:type/og:image`, twitter card.
- [x] `client/public/robots.txt` exists, allow-all, served at `/robots.txt`.
- [x] Favicon declared with correct MIME (`image/png`) from a stable `public/` URL.
- [x] Entry script normalized to `/src/main.tsx`.
- [x] SPA per-lesson-OG limitation documented in `client/CLAUDE.md`.
- [x] `npm test` green (existing + new tests), `npm run build` green, no new lint errors.
- [x] Lesson viewer / all routes render identically (nothing behavioral changed).
- [x] One local commit in the `client` repo; **nothing pushed**.

## 4. Explicit non-goals (things a diligent agent might wrongly add)

- ❌ Self-hosting fonts / touching the Google-Fonts links (Tier 2.B, owner-gated).
- ❌ Creating a 1200×630 OG card image (needs owner-approved art; the square-logo `summary` card is the accepted v1 — see ROADMAP §4 which allows either).
- ❌ `og:url`, canonical link, sitemap.xml, structured data / JSON-LD.
- ❌ Per-route/per-lesson meta tags, react-helmet, vite-plugin-html, SSR of any kind.
- ❌ New env vars, new npm dependencies, vercel.json changes.
- ❌ Fixing the `__dirname` lint error (Tier 4) or the 2 MB chunk warning (Tier 2).
- ❌ Touching `server/`, `language.store.ts`, or any `.tsx` component.

## 5. When to stop and ask the owner

- `git status` in `client/` is dirty before you start, or shows foreign files (wrong repo!).
- `index.html` does not match the "Verified facts" description (someone changed it since this plan was written).
- Any pre-existing test fails at Step 1.
- The owner wants different Thai wording for title/description (use theirs, then re-run Step 7.1).

---

## Execution log (implementing agent fills this in)

| Step | Status | Notes |
| --- | --- | --- |
| 1. Preflight baseline | ✅ | clean tree; 66/66 tests; build green |
| 2. public/ assets | ✅ | favicon.png + og-image.png copied; vite.svg removed |
| 3. index.html replaced | ✅ | Thai meta/OG/Twitter tags; main.tsx entry |
| 4. robots.txt | ✅ | allow-all |
| 5. seo.test.ts | ✅ | 8 new tests |
| 6. client/CLAUDE.md note | ✅ | SPA limitation documented |
| 7. Verify (tests/build/dev) | ✅ | test count: 74/74 / build: green / lint: 1 expected (`__dirname`) |
| 8. Local commit (no push) | ✅ | commit hash: `e3c53ec` |
