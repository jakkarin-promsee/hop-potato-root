# ROADMAP-guide.md — Hot Potato Guide & Tutorial Roadmap (v1)

The dedicated plan for rebuilding `/guide` into a real **tutorial system**: a hub page plus two step-by-step showcase pages (student side + teacher side). Written 2026-07-11 with the owner. This roadmap **expands ROADMAP.md Tier 6.B** ("Guide refresh") into its own track — owner decision D5 said "untouched until last"; the owner has now green-lit building it as its own workstream.

> Companion execution spec: [`plan/guide.md`](plan/guide.md) — the full feature inventory (verified against code 2026-07-11) and the scene-by-scene presentation scripts for both showcase pages. **Implement from the plan file; this roadmap is the summary + priorities.**
>
> Maintenance recipe: [`update-guide.md`](update-guide.md) — how a future agent **renames the brand and re-renders every screenshot** (content unchanged). Read that when the product is finished and the guide just needs a refresh.
>
> Read [`CLAUDE.md`](CLAUDE.md) and [`client/CLAUDE.md`](client/CLAUDE.md) first. ⚠️ `client/` and `server/` are separate git repos; never run git from `Hot-Potato/`.

---

## 1. Vision

**A brand-new teacher or student lands on `/guide` and, within one page of reading, knows exactly what to press.** The target users are explicitly low-tech (teachers got their first ChatGPT training a week ago; students are AI newbies on phones with bad internet), so the guide is:

- **A hub + two showcases.** `/guide` becomes a small router page ("ฉันเป็นนักเรียน / ฉันเป็นครู") linking to two dedicated walkthrough pages — **learning-showcase** (the whole student flow: explore → read → answer → ask AI → history) and **creating-showcase** (the whole teacher flow: create → write → blocks → questions → AI copilot → publish/share). Plus a direct "ไปลองเลย" link to `/explore`.
- **Scene-by-scene, not a game tour.** Each showcase is an ordered sequence of *scenes*: one annotated screenshot + short numbered steps in plain Thai. No interactive overlay tutorial in v1 (decision G2 below).
- **Cheap to keep true.** Screenshots come from a repeatable Playwright capture script, so when the UI changes we rerun one command instead of re-capturing 40 images by hand.
- **As light as the rest of the app.** The audience has bad internet; Tier 2's bundle work must not be undone by a guide page full of heavy PNGs.

The two Golden Rules apply unchanged: the guide and both showcases are **public pages** (no `protect`, no login wall), and nothing here rations anything.

---

## 2. Method decisions (agreed with the owner 2026-07-11 — do not re-litigate)

| # | Decision |
| --- | --- |
| G1 | **Two showcase pages, split by role.** `learning-showcase` (student) and `creating-showcase` (teacher) as separate routes under `/guide/*`. The old single feature-list Guide page is replaced by the hub. Rationale: the teacher editor alone is too button-dense ("ปุ่มเยอะมาก") to share a page with the student flow. |
| G2 | **Scene-by-scene screenshots + steps, NOT an interactive overlay tour.** A driver.js/react-joyride-style tour was considered and rejected for v1: it pins to DOM selectors (brittle while UI still moves), needs per-step app state, and tooltip positioning on the phone-first layout is a known pain. Revisit post-launch as a deferred item for 1–2 high-friction flows only (Tier G6). |
| G3 | **Screenshots are generated, never hand-captured.** A checked-in Playwright script drives local dev against known demo data and captures every scene from a manifest. UI changed? Rerun the script. This kills the "stale screenshots" failure mode of tutorial pages. |
| G4 | **Bandwidth-first assets.** WebP, width-capped (~800 px content width is plenty at the app's 400 px lesson width), `loading="lazy"` on everything below the fold, and both showcase pages are lazy routes — the entry chunk (137 kB gzip after Tier 2.A) must not grow. |
| G5 | **Thai-first copy, bilingual via `useAppI18n`** like every page since Tier 1.B. The current Guide page is English-only — that is one of the things being fixed. |
| G6 | **The student showcase teaches the anonymous path first.** Golden Rule 2 is the product's superpower — the walkthrough shows everything working *without* login, then has one scene on "what login adds" (history, memory, saved answers). Never present login as a prerequisite. |
| G7 | **A "demo lesson" (learn-by-doing tutorial lesson inside the platform itself) is deferred, not dropped.** It was the strongest idea in the method discussion — the student tutorial *as* a lesson with real questions + real AI. It needs owner-authored content, so it lands in Tier G6; the learning-showcase links to it when it exists. |
| G8 | **This track does not block launch.** It is independent of ROADMAP.md Tiers 0–4 and can ship before or after Tier 5 — same local-only workflow (nothing pushes until the launch pass). If launch happens first, the old Guide page ships and this replaces it later. |

**Open naming questions for the owner** (placeholders are fine until answered — flagged in the plan file):

1. Route/display names — plan uses `/guide/learning` + `/guide/creating` with working titles "เรียนยังไง" / "สร้างบทเรียนยังไง". Rename freely.
2. **Brand name — answered 2026-07-11: "Hot Potato"** (owner decision; matches the repo and the tutor น้องมันฝรั่ง). `BRAND_NAME` in `client/src/lib/brand.ts` carries it everywhere; the last "Intuita" remnant is the **logo art** (`client/src/assets/logo.png`) — asset swap is an owner task.

---

## 3. Current state (verified 2026-07-11)

- `/guide` today (`client/src/pages/Guide.tsx`): a static, **English-only marketing feature list** — 7 alternating icon sections (editor, viewer, questions, tutor, explore, history, share) + CTA buttons to `/explore` and `/create`. No steps, no screenshots, mixes both roles. Owner: "ของเก่า ทำไว้แบบเอาไปที" — replace it.
- TopNav already links Guide (`คู่มือ`, `BookOpen` icon) for everyone, so the hub gets traffic on day one; no nav work needed.
- All 15 routes are real and shipped (old-roadmap Tiers 0–3 + 3.5); the feature surface to document is stable. Full inventory: [`plan/guide.md`](plan/guide.md) §2–§3.
- Playwright MCP + headless Chromium are already wired into the dev environment (AGENT.md §7) and smoke-tested against the client — the capture pipeline builds on the same stack (plain `playwright` as a client devDependency for the script itself).

---

## 4. Tier G0 — Content plan & scene scripts 📋 ✅

Produce the full feature inventory (both roles, button-level) and the ordered scene scripts for both showcases, plus the screenshot manifest each scene needs.

**Shipped 2026-07-11:** this file + [`plan/guide.md`](plan/guide.md). Inventory was compiled from the code (all 15 pages + the editor/AI-copilot surfaces) — not from memory.

---

## 5. Tier G1 — Screenshot pipeline 📸 ✅ (shipped 2026-07-11)

**Theme: one command regenerates every guide image.** Do this before building pages so the pages are born with real assets.

### Phase G1.A — Demo data prep ✅

> **Shipped with a deviation:** instead of editing the owner's test lesson, `client/scripts/seed-guide-demo.mjs` creates a **dedicated link-only demo lesson** ("ทำไมท้องฟ้าเป็นสีฟ้า? (บทเรียนเดโม)") containing **every** question block type + a formula block, under demo accounts `guide.teacher@hotpotato.local` / `guide.student@hotpotato.local` (learner role, password documented in the script header, overridable via `--password`/`GUIDE_DEMO_PASSWORD`). The student account gets history (2 lessons) + a real StudentMemory via 4 live tutor turns. Ids land in `client/scripts/guide-demo.json`. Rerunnable; **must be rerun against production at launch** (ids differ per DB).

**Acceptance:** met — every scene's app state is scripted, not hand-made; the demo lesson demonstrates all question types.
**Size:** S · **Repos:** client (scripts only)

### Phase G1.B — Capture script ✅

- `client/scripts/capture-guide.mjs` (devDeps `playwright` + `sharp`): inline **scene manifest** (viewport, auth, seeded localStorage for login/bookmark/Thai, pre-actions, AI flag), drives local dev, waits out SSE streaming on AI scenes, captures element or viewport shots → **WebP q82** into `client/public/guide/` + regenerates `src/pages/guide/guideImages.ts` (dimensions manifest for CLS-free `<img>`).
- Phone 390×844 @2x for all 12 learning scenes (desktop 1280 viewport ready for G4). `--scene <id>` recaptures one; AI scenes (05, 06) cost real Gemini tokens.
- Field notes: lesson title "การเคลื่อนที่เเละเเรง" spells แ as `เเ` — search by a safe substring; the blank-drag scene ships pre-drag (HTML5 DnD is flaky headless; the shot still shows the mechanic).

**Acceptance:** met — full run produced 12/12 deterministically; `--scene` partial reruns keep other entries.
**Size:** M · **Repos:** client
**Shipped:** client commit `b619a99`.

---

## 6. Tier G2 — `/guide` hub rebuild 🏠

- Replace `Guide.tsx` content with the hub: short intro (one paragraph, what this site is), **two big role cards** — "ฉันเป็นนักเรียน → เรียนยังไง" / "ฉันเป็นครู → สร้างบทเรียนยังไง" — linking to the showcases, plus a third smaller "ไปลองเองเลย → Explore" card and a Help/Status footer row (reuse the Settings help-popup contact pattern).
- Bilingual (`useAppI18n`), phone pass at 390 px, keep the route public.
- The two showcase routes (`/guide/learning`, `/guide/creating`) are added as **lazy routes** in `App.tsx` in this tier (pages can be skeletons until G3/G4 fill them) — so navigation, chunks, and tests land once.

**Acceptance:** hub renders both languages; role cards navigate; entry-chunk size unchanged (`npm run build` compared before/after); tests cover hub render + links; lint/build/test green.
**Size:** S · **Repos:** client
**Shipped 2026-07-11:** client commit `32892de` — hub + both lazy routes + `lib/brand.ts` (`BRAND_NAME = "Hot Potato"` working name, owner has no final brand yet). Teacher card carries an honest "กำลังจัดทำ" chip until G4. Entry chunk 138 kB gzip (was 137).

---

## 7. Tier G3 — learning-showcase 🎒 (student walkthrough)

Build `/guide/learning` from the scene script in [`plan/guide.md`](plan/guide.md) §4 — **9 scenes**: เข้าเว็บ → Explore → เปิดบทเรียน → ตอบคำถาม 4 แบบ → อ่าน feedback + คุยต่อ → ถาม AI อิสระ → เลือกบุคลิกติวเตอร์ → login ได้อะไรเพิ่ม → ตั้งค่าตามใจ.

- One shared **showcase shell** component (sticky scene TOC / progress, scene = image + numbered steps + tip box, prev/next, final CTA) — built here, reused by G4.
- Images from `public/guide/` with `loading="lazy"`, width/height attrs (no CLS), phone image as default `src` with desktop via `<picture>` where the layouts differ.
- Each scene ends with the "ไปลองเลย" deep link where applicable (`/explore`, `/settings`).
- Anonymous-first framing per decision G6.

**Acceptance:** a first-time phone user can follow scene 1→9 and actually operate the app; all images lazy + WebP; 390 px pass incl. largest font size; showcase chunk lazy (entry unchanged); tests cover shell + scene rendering; build/lint/test green.
**Size:** M · **Repos:** client
**Shipped 2026-07-11:** client commit `32892de` (same commit as G2) — 9 scenes with real screenshots, sticky scene TOC, scroll-to-hash for `#scene-n` deep links (landed early from G6), 190 client tests green, headless-browser pass: no console errors / no failed requests / no horizontal overflow at 390 px (phone + desktop + dark). `/guide/creating` ships as a friendly placeholder until G4.

---

## 8. Tier G4 — creating-showcase 🧑‍🏫 (teacher walkthrough) ✅

Build `/guide/creating` from the scene script in [`plan/guide.md`](plan/guide.md) §5 — **10 scenes**: เตรียมตัว (login + ⚠️ ใช้คอมพิวเตอร์) → สร้างบทเรียนแรก (ได้หน้าว่างทันที) → รู้จักหน้าจอ 4 ส่วน → เขียนเนื้อหา (H1/H2/toolbar) → แทรกรูป (Media panel; **กระดานแคนวาส omitted** — plan §9 Q7) → สูตรคณิต (Formula AI) → สร้างคำถาม 5 แบบ + แนวเฉลย → AI copilot ทั้ง 4 กลุ่ม → ตรวจ + เผยแพร่ + ตั้งค่าติวเตอร์ → แชร์ (QR / ลิงก์ / 3 ระดับการมองเห็น) + ดูผ่านตานักเรียน. The guide teaches `/create` as the teacher's home (not the English-only `/dashboard` near-duplicate — plan §9 Q4).

- Reuses the G3 shell. The AI-copilot scene mirrors the editor's own 4-group workflow (เริ่มบทเรียน → เขียนและเกลา → คำถาม → ก่อนเผยแพร่) so the guide teaches the same mental model as the sidebar hub.
- The share scene explains the three `access_type` levels in plain Thai + how the `/view/:id` link is what you paste into LINE/Facebook (and that previews show a generic card for now — Tier 6.D honesty).
- Desktop screenshots are primary for editor scenes (the editor is a desktop-leaning surface); phone shown where relevant.

**Acceptance:** a low-tech teacher can go blank → published following the scenes; every editor tool group appears in at least one scene; same technical gates as G3.
**Size:** M–L · **Repos:** client
**Shipped 2026-07-11:** client commit `1abb1ef` — 10 scenes with **13 real desktop screenshots** (4 of them real-Gemini: formula AI, guide-answer draft, outline draft, lesson critic). The seed script grew a private **scratch lesson** (reset to a baseline doc between editing shots via `resetScratch`) + an untitled **blank lesson** (empty-doc AI CTA) + 4 vault images; lesson docs moved to shared `scripts/guide-demo-docs.mjs`. `SceneImage`/`SceneSection` gained a **wide/stacked layout** + aspect-aware sizing so the landscape editor shots read well at 390 px and on desktop. Verified headless (10/10 scenes, no console errors / no failed requests / no h-scroll at 390 px, dark + light, `#scene-n` deep links). Entry chunk unchanged (138 kB gzip); `CreatingShowcase` is its own ~5.5 kB gzip lazy chunk. 194 client tests green. Q5 (YouTube) **confirmed**: still no insert button in the editor rail → media scene teaches images only, videos out of v1. **Post-ship (2026-07-11):** scene 5 **กระดานแคนวาส** removed per plan §9 Q7 — Fabric draw board is too complex and rarely used; `creating-05b-canvas` shot + copy dropped from `creatingScenes.ts`.

---

## 9. Tier G5 — Integration & polish 🔗

- **Cross-links:** ~~Landing links to the hub~~ **superseded 2026-07-11 — Landing and the hub are now one page** (see the merge note in §12): `/` carries the role cards itself and `/guide` redirects there. Remaining: Settings Help popup gains a "อ่านคู่มือ" row → `/` (or straight to the two showcases); the showcases' audience cross-links already shipped with the shell (G3/G4).
- **SEO:** title/meta description for the three guide routes (static, site-wide OG stays as-is per Tier 0.A).
- Final passes: both languages, both themes, 390 px + largest font size, lint/build/test, and a fresh `npm run build` size comparison logged in the phase notes.
- Update `client/CLAUDE.md` (routing table + guide section) and this file's status lines.

**Acceptance:** all entry points reach the hub; docs updated; all suites green; bundle budget intact.
**Size:** S · **Repos:** client

---

## 10. Tier G6 — Deferred ✨ (after the above, owner-gated)

| Item | What | Why deferred |
| --- | --- | --- |
| **Demo lesson** (decision G7) | A real lesson in the platform that *teaches the platform* — questions + AI tutor about using the app; learning-showcase links to it as "ลองทำจริง" | Needs owner-authored/approved content (tone + Thai copy); infra already exists |
| **Interactive spotlight tour** (decision G2) | driver.js-style 3–5 step overlay for 1–2 flows that real usage shows people get stuck on (candidate: first `/canvas/:id` visit) | Brittle pre-launch; needs real post-launch friction data to justify |
| **Motion clips** | Short GIF/WebM loops for inherently-motion scenes (drag-a-blank, streaming reply) | Bandwidth cost vs. audience; stills must prove insufficient first |
| **Per-scene deep links** | `/guide/learning#scene-4` anchors teachers can send students | Cheap, but wait until scene list stabilizes |

---

## 11. Careful-not-to-break list

| Thing | Why it's fragile here |
| --- | --- |
| **Entry bundle size** (Tier 2.A: 137 kB gzip) | Showcase pages must be lazy routes; guide images live in `public/guide/` (static, unhashed), never `import`ed into the entry chunk. Compare `npm run build` before/after every guide phase. |
| **Golden Rule 2** | `/guide`, `/guide/learning`, `/guide/creating` are public. No `protect`, no login-gated content. The login scene *describes* login; it never requires it. |
| **Bilingual + phone-first** | Every new string through `useAppI18n`; every layout judged at 390 px first (owner principle 5) — including with the Tier 3.A largest font size. |
| **Read-only feature** | The guide *documents* the app; guide phases must not modify editor/tutor/store code. If a screenshot reveals a bug, file it separately — don't fix it inside a guide phase. |
| **`vercel.json` SPA rewrite** | Already rewrites all paths → new nested routes work with zero config; don't add per-route Vercel config. |

---

## 12. Sizing & suggested order

| Phase | What | Size | Repos | Depends on |
| --- | --- | --- | --- | --- |
| **G0** | Inventory + scene scripts (`plan/guide.md`) | M | docs | — | ✅ 2026-07-11 |
| **G1.A** | Demo data prep (seed script + demo lesson) | S | client | — | ✅ 2026-07-11 (`b619a99`) |
| **G1.B** | Playwright capture script → WebP pipeline | M | client | G1.A | ✅ 2026-07-11 (`b619a99`) |
| **G2** | Hub rebuild + lazy routes | S | client | — | ✅ 2026-07-11 (`32892de`) |
| **G3** | learning-showcase (9 scenes + shared shell) | M | client | G1, G2 | ✅ 2026-07-11 (`32892de`) |
| **G4** | creating-showcase (10 scenes) | M–L | client | G1, G3 (shell) | ✅ 2026-07-11 (`1abb1ef`) |
| **G5** | Cross-links, SEO, polish, docs | S | client | G3, G4 |
| **G6** | Demo lesson · spotlight tour · clips · anchors (scene deep links already shipped in G3) | ? | client (+owner) | after G5 |

**2026-07-11 — hub merged into Landing (owner decision, post-G4):** `/` is now the single front door — hero pitch (bilingual, `BRAND_NAME`) + the hub's three role cards + contact footer, rendered inside `AppLayout`. The old marketing Landing (fictional showcase cards, hardcoded "Intuita", English-only) was deleted; `pages/Guide.tsx` deleted; `/guide` client-redirects (`Navigate replace`) to `/`; TopNav's "คู่มือ" item became "หน้าแรก"; the showcases' back-link points at `/`. Landing stays the eager route — entry chunk unchanged (138 kB gzip). **Rider:** `learning-01-landing.webp` still shows the old landing — recapture scene 1 via `scripts/capture-guide.mjs --scene` when convenient. ROADMAP.md decision D7 (fictional showcase cards) is moot — the cards no longer exist.

**Remaining path:** **G5** to close (Settings Help-popup link, SEO titles, final passes). Launch rider: rerun `seed-guide-demo.mjs` + `capture-guide.mjs` against production during ROADMAP.md Tier 5 (demo/scratch/blank lesson ids differ per DB).

**Working agreement (inherited from ROADMAP.md):** every phase ships tests for its own acceptance criteria; a phase is not done until `npm test`, `npm run build`, and `npm run lint` (no *new* errors) are green in the client, and the relevant `CLAUDE.md` is updated.
