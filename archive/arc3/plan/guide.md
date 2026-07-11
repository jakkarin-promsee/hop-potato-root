# plan/guide.md — Guide & Showcase Pages: Execution Plan

Written 2026-07-11 for an implementing agent (or the owner) to execute **without access to the planning conversation**. The roadmap summary is [`../ROADMAP-guide.md`](../ROADMAP-guide.md) (tiers G0–G6); this file is the full content plan: the verified feature inventory, the **scene-by-scene presentation scripts** for both showcase pages, the screenshot manifest, and the page architecture. Read [`../CLAUDE.md`](../CLAUDE.md) and [`../client/CLAUDE.md`](../client/CLAUDE.md) before touching code.

Everything in §2–§3 was compiled **from the code on 2026-07-11** (all 15 pages + every editor/AI surface read directly), not from memory. If something looks off, re-verify against the file cited.

---

## 0. Ground rules (non-negotiable)

1. **Golden Rule 2 governs this feature.** `/guide`, `/guide/learning`, `/guide/creating` are public — no `protect`, no login wall, no content hidden behind auth. The login scene *describes* login; it never requires it.
2. **Bundle budget.** Entry chunk is 137 kB gzip (Tier 2.A). Both showcase pages are `React.lazy` routes; guide images live in `client/public/guide/` (static, unhashed) and are referenced by URL — **never `import`ed** into any chunk. Compare `npm run build` output before/after every phase.
3. **Bilingual via `useAppI18n`** (`t("English", "ไทย")`), Thai written first-class, not translated-sounding. Phone pass at 390 px for every layout (creating-showcase scenes *show* desktop screenshots but the page itself must still read fine on a phone).
4. **This feature is read-only over the app.** It documents; it does not modify editor/tutor/store code. A screenshot session revealing a bug → file it separately, don't fix it in a guide phase.
5. **Method decisions G1–G8** in [`../ROADMAP-guide.md`](../ROADMAP-guide.md) §2 are settled — most importantly: scene-by-scene screenshots (no interactive tour in v1), Playwright-generated assets, anonymous-first framing.
6. **Git reality:** work lands in the `client/` repo only. Never run git from `Hot-Potato/`. No pushes (owner decision D1 — one deploy pass at launch).
7. **Definition of done per phase:** acceptance criteria pass, `cd client && npm test` + `npm run build` green, `npm run lint` has no *new* errors, `client/CLAUDE.md` updated where routing/structure changed.

---

## 1. Verified current state

- **`/guide` today** (`client/src/pages/Guide.tsx`): static English-only marketing list — 7 alternating icon sections (editor / viewer / questions / tutor / explore / history / share) + a bonus "AI remembers you" section + CTA row (`/explore`, `/create`). No steps, no screenshots. Owner verdict: throwaway — replace with the hub.
- **TopNav** already shows `Guide / คู่มือ` (`BookOpen`) to everyone — the hub gets traffic with zero nav work.
- **Routes to add** in `App.tsx`: `/guide/learning` and `/guide/creating`, both lazy, both inside `AppLayout` (same chrome as the current Guide route). `vercel.json` SPA rewrite already covers them.
- **⚠️ Brand-name caveat (owner decision needed before copy-writing):** the UI wordmark rendered everywhere (logo `alt`, Landing header, current Guide headline "What Intuita can do") is **"Intuita"** — while the repo/docs say **Hot Potato** and the tutor persona is **น้องมันฝรั่ง**. The guide must use one name consistently. Use a `BRAND_NAME` constant in the guide copy so the answer is a one-line change.
- **Editor is desktop-only:** `/canvas/:id` replaces itself below `md` with a full-screen "Editing needs a larger screen" dialog. The creating-showcase must say this in scene 1, not let a phone teacher discover it at scene 3.
- **Test lesson** for demo/screenshots: `/view/69e39d0b60d467bd515a4945` ("การเคลื่อนที่และแรง", ~14 questions) — already the manual-smoke lesson in `AGENT.md` §8.

---

## 2. Feature inventory — student side (source for learning-showcase)

Full route map is in `client/CLAUDE.md`. What matters for the tutorial, per surface:

| Surface | Route / guard | What a student can do (button-level) |
| --- | --- | --- |
| **Landing** | `/` · public | Own header (not TopNav): `Log in`, `Explore`. Hero CTAs **"Start exploring →"** → `/explore`, **"I'm a teacher"** → `/create`. "How it works" 3 steps; 4 showcase cards are **fictional/decorative** (owner decision D7 — don't screenshot them as if real). |
| **Explore** | `/explore` · public | Search box (debounced, "ค้นหาบทเรียนสาธารณะ..."); pill tabs **ทั้งหมด / บุ๊กมาร์ก / ล่าสุด** (client-side; ล่าสุด = updated ≤14 d); "เรียนต่อ (Continue learning)" rail from history (logged-in only, anonymous sees empty-state copy); lesson grid of `ContentCard`s — card click → `/view/:id`, **bookmark icon** top-right (device-local, `bookmark-storage`, works anonymously, never synced). |
| **Lesson viewer** | `/view/:id` · public (optionalAuth) | Auto-hiding TopNav on scroll; single centered card; **zoom = Ctrl/⌘+wheel, Ctrl +/−, Ctrl 0** (auto-fits on load; no on-screen buttons); H2 sections auto-numbered. Question blocks render inline. FABs bottom-right: **"Ask AI"** (always) + "Edit" (owner only, desktop). Private lessons → "login required" error state with Log in button. |
| **4 gradeable question cards** | inside viewer | Shared skeleton: question → input → **ส่ง (Submit)** → streamed AI feedback → suggestion chips → follow-up thread ("ส่งคำตอบเพิ่มเติม?") → **ลองใหม่ (Try again)** resets. Types: **เลือกตอบ** (single/multi, rows recolor เขียว/แดง + "🎉 ถูกทั้งหมด!"), **เขียนตอบ** (write_evaluation — "การประเมินเชิงลึกจาก AI", no right/wrong coloring), **เติมคำแบบลากตัวเลือก** (drag chips into `[Q-n]` blanks, single-use, X to remove), **เติมคำแบบพิมพ์** (AI judges meaning — `ai_judge`, no hard grading). |
| **AI feedback shared UI** | inside cards | `SuggestionChips` (≤3 tappable pills — tap = send); `FeedbackDiscussionPanel` follow-up thread (Enter to send; follow-ups are plain conversation — "สวัสดี" is small talk, not graded); `AiErrorRetry` ("ตอนนี้ AI ตอบไม่ได้ ลองกดส่งอีกครั้งนะ 🙏"); cold-start hint after 6 s ("ปลุก AI แป๊บนึงนะ เซิร์ฟเวอร์เพิ่งตื่น 😴"). |
| **Ask-AI modal** | viewer FAB | Bottom-sheet (phone) / dialog (desktop). Header: Clear chat (2-step confirm), X. **PersonalityPicker row.** Streaming replies (markdown via `MarkdownMessage`), suggestion chips above input, Enter to send. Sends reading-position hint (`currentSection`) — the AI knows which section you're on. Works fully anonymous. |
| **QuestionAgent block** | inside lesson | Teacher-placed free-chat block: "ถาม AI ได้เลย..." → expands to thread; แสดง/ซ่อนแชต; ล้างประวัติ. |
| **Tutor personalities** | Settings + Ask-AI modal | 6 presets (`tutorPersonality.store`): 🥔 น้องมันฝรั่งคลาสสิก · 🌷 สุภาพ อ่อนโยน · 😏 แสบๆ ขี้แซว · 🔍 อธิบายละเอียด · ⚡ สั้น กระชับ · 🎯 จริงจัง โฟกัส. Persisted locally, works anonymously, applies to every AI chat. |
| **History** | `/history` · RequireLogin | Day-grouped ContentCards (วันนี้ / เมื่อวาน / dates), up to 100. Written automatically on lesson open (logged-in only). |
| **Profile** | `/profile` · RequireLogin | Avatar upload (≤5 MB), display name / ชื่อเล่น / bio (500 chars), change-password link. **"ความจำของติวเตอร์" card**: chips (สิ่งที่สนใจ/จุดแข็ง/กำลังพัฒนา/สไตล์ที่ชอบ/เรื่องล่าสุด) + **ลบความจำ** with confirm ("น้องมันฝรั่งจะลืมทุกอย่าง… ลบแล้วกู้คืนไม่ได้นะ"). Personality shown read-only + "เปลี่ยน" → Settings. |
| **Settings** | `/settings` · public | ธีม toggle; **ขนาดตัวอักษร** 4 ระดับ (เล็ก 87.5% → ใหญ่มาก 125%, whole-app); ภาษา EN/TH; **PersonalityPicker**; รหัสผ่าน (logged-in); Help popup ("เว็บนี้ทำฟรีโดยคนคนเดียว 😄" + Facebook + `/status` link); ออกจากระบบ (logged-in). |
| **Login/Register** | `/login` · PublicRoute | One page, toggle sign-in ⇄ create account (password ≥8). Redirect-back to where you were (`state.from`). Calm Thai session-expiry banners. |
| **Status** | `/status` · public | 6 cards (server/db/env×2/AI tutor/recent errors), 30 s poll — the "เว็บล่มหรือเปล่า" page. |

**Persistence story (the login-value scene):** anonymous = everything works, nothing saved (answers/chats reset per session; bookmarks + personality DO persist locally). Logged-in = answers autosync (30 s), per-block chat history on the server, learning history, tutor memory across lessons.

---

## 3. Feature inventory — teacher side (source for creating-showcase)

The editor (`/canvas/:id`, desktop-only) is the button-dense core. Numbers that justify a dedicated page: 5 left-rail categories, ~12 header controls, 7 right-sidebar modes, 6 AI dialogs + 2 in-block AI assists, 5 question types, plus a full Fabric canvas sub-editor and an Image Vault.

| Surface | What it holds |
| --- | --- |
| **Lesson lists** | `/create` (bilingual: "เนื้อหาของคุณ", สร้างบทเรียนใหม่, search) and `/dashboard` (English-only near-duplicate + per-card delete). **The tutorial teaches `/create` only** (it's TopNav's "สร้าง" target and bilingual); Dashboard gets one passing mention for delete. New lesson = created blank instantly → straight into editor (no upfront form; metadata comes later in Publish). |
| **Editor header** | ‹ back to `/create` · inline title input · save-state (กำลังบันทึก… / บันทึกแล้ว / **บันทึก?**) · ย้อนกลับ/ทำซ้ำ · **ปรับข้อความ** (AI writing dropdown) · theme/language toggles · Live⇄Static (`สด/คงที่`) toggle · zoom −/%/+ · link-click-mode toggle · **ตรวจบทเรียน** (AI critic) · **เผยแพร่ (Publish)**. |
| **Left rail (5 panels)** | **AI** (`Sparkles`, leads, tinted — the hub); **Text** (โครงสร้าง H1/H2/H3/quote, จัดวาง, สีข้อความ, ไฮไลต์, เส้นคั่น/lists, ตาราง 3×3, เช็กลิสต์, **กระดานแคนวาส**); **Media** (upload อุปกรณ์/URL, progress, จัดการคลังภาพ → Image Vault modal, category filter = upload target, click-to-insert grid); **Formula** (เพิ่มบล็อกสูตร + symbol groups: กำลัง/โครงสร้าง/ตรีโกณ/log/ค่าคงที่/ตัวดำเนินการ/ฟิสิกส์/ครอบข้อความ); **Question** (สร้างคำถามด้วย AI + 5 insert cards). |
| **Right sidebar (contextual)** | เอกสาร (word count/read time) · text (**Search & Replace** + Bold/Italic/Underline/Strike/Quote/**Link**) · ลิงก์ (URL, new-tab, remove) · รูปภาพ (**ครอป** with ratio presets, resize 10–200%, จัดวาง, px + lock ratio, alt, link, ลบ) · ตาราง (±แถว/คอลัมน์, รวม/แยกเซลล์, ลบ) · โค้ดบล็อก (language select). |
| **Main-area behaviors** | H1 = ชื่อเรื่อง; H2 auto-numbered `1. 2. 3.` (typed prefixes stripped); drag-drop/paste image auto-uploads to Cloudinary; click empty space focuses end; Ctrl+scroll zoom. |
| **Autosave/conflict** | Autosave 30 s + `beforeunload` + tab-return >1 min; **Ctrl+S** manual; 409 → yellow banner "⚠️ มีเวอร์ชันใหม่กว่าอยู่แล้ว!" with โหลดเวอร์ชันล่าสุด / เขียนทับด้วยของฉัน. |
| **Fabric canvas** | Sidebars swap when active. Left: Templates / Elements (shapes + connectors) / Text / Uploads / Draw (brush size + 8 colors). Right: per-object properties (text fonts/shadow, shape fill/stroke/radius, line endpoints, group/ungroup, opacity, layer order). |
| **FormulaBlock** | Header + move controls + **ดูตัวอย่างแบบผู้เรียน** eye; 8 quick templates; LaTeX textarea; live KaTeX; **AI: "ให้ AI เขียนสูตร (ไม่ต้องรู้ LaTeX)"** — type `s = ut + 1/2at^2` + คำอธิบาย → LaTeX via the same `persistLatex` path. |
| **5 question types (creator)** | Shared: feedback-mode toggle (**เข้าใจแบบย่อ** ⇄ **สะท้อนความคิดแบบละเอียด**), preview-eye, move controls. **เลือกตอบ**: single⇄multi, choice rows + correct toggles, **AI ตัวลวง** chips (needs a correct ticked first). **เขียนตอบ**: question + แนวเฉลย textarea, **AI ร่างแนวเฉลย** → ใช้อันนี้/ทิ้ง. **เติมคำ (ตัวเลือก)**: เพิ่มช่องว่าง `[Q-n]` tokens, choice bank, per-blank correct mapping, live preview. **เติมคำ (เขียนตอบ)**: tokens + per-blank answers (AI judges meaning). **ถามผู้ช่วย AI**: title only — a free-chat block for students. Guide answers matter beyond grading: they feed the student tutor's context (`lessonContext`). |
| **AI hub (`AiToolsPanel`)** | 4 workflow groups: **1 เริ่มบทเรียน** (ร่างโครง · วางเนื้อหาเดิม) → **2 เขียนและเกลา** (เติมเนื้อหาในหัวข้อ · ปรับข้อความที่เลือก) → **3 คำถามชวนคิด** (สร้างคำถามด้วย AI) → **4 ก่อนเผยแพร่** (ตรวจบทเรียน). Same dialogs as header/panel twins. Universal rules: **preview → accept** everywhere; busy error "AI ไม่ว่างแป๊บนึง ลองอีกทีนะ 🥔"; empty-doc CTA "หน้าว่างอยู่ใช่ไหม ให้ AI ช่วยเริ่มได้นะ ✨". |
| **AI dialogs** | **AiDraftDialog** 3 tabs — ร่างโครง (topic+ป.1–ม.6+objectives → outline at caret), เติมเนื้อหา (pick heading + style hint → insert below heading + optional suggested questions), วางเนื้อหาเดิม (paste ≤20k → structured draft + questions). **AiQuestionDialog** — scope/types/count 1–10/difficulty → preview cards → เพิ่มลงบทเรียน per card or เพิ่มทั้งหมด. **AiWritingAssistant** — 6 presets (แก้คำผิด · จัดย่อหน้า/หัวข้อ · เกลาให้อ่านง่าย · ย่อ · ขยายความ · ปรับระดับชั้น) → before/after ("ก่อน/หลัง") → ใช้เลย. **AiCriticDialog** — warm summary + เช็กลิสต์ ✓/○ + จุดที่ปรับได้ by area; informational, never blocks publish. |
| **PublishSettingsModal** | ให้ AI ช่วยกรอก (title/description/topics, overwrite-confirm) · ชื่อเรื่อง · รูปปก (gallery/URL/upload) · **ประเภทการเข้าถึง: สาธารณะ / เฉพาะลิงก์ / ส่วนตัว** · ผู้ร่วมงาน (by user ID) · หัวข้อ chips · คำอธิบาย · **AI ติวเตอร์ section**: ให้ AI แนะนำการตั้งค่า + persona_note (≤500) + สไตล์คำตอบ (ชวนคิดก่อน ⇄ บอกตรงๆ ได้) + ขอบเขต (เฉพาะบทเรียน ⇄ +ความรู้ทั่วไป) + แนวทางเพิ่มเติม (≤1000) · **การแชร์**: QR code (tap-to-fullscreen) + copy `/view/:id` link · footer: ลบ / บันทึกการตั้งค่า / **เผยแพร่ตอนนี้** (→ public + save + open viewer). Close guard asks to save. |
| **Sharing model** | No separate "published" flag — sharing **is** the `/view/:id` URL (QR, copy-link, or Explore listing when `public`). `link-only` = reachable by URL, not listed. Per-lesson link previews are generic site cards for now (ROADMAP Tier 6.D) — the guide says so honestly. |
| **Image Vault** | `/uploadimage` + embedded modal: groups sidebar (+ New group), file/URL upload with drop zone, gallery grid (delete, assign category), detail panel (URL copy, quick transforms). English-only, dark theme. |

---

## 4. learning-showcase — scene script (9 scenes)

Route `/guide/learning` · working title **"เรียนกับ{BRAND}ยังไง"**. Audience: a student on a phone, possibly never used an AI app. Framing rule (decision G6): **anonymous-first** — scenes 1–7 never mention needing an account; scene 8 presents login as a bonus.

Every scene = hero screenshot (phone 390 primary) + 2–5 numbered steps + optional 💡 tip box + optional "ไปลองเลย →" deep link. Copy below is **draft Thai first**; EN via `t()`. Tone: warm, short sentences, zero jargon — same voice as น้องมันฝรั่ง.

| # | Scene (title draft) | Steps to show | Screenshot(s) | Deep link |
| --- | --- | --- | --- | --- |
| 1 | **เริ่มได้เลย ไม่ต้องสมัคร** | เว็บนี้คือที่อ่านบทเรียน + ถาม AI ได้ฟรี · เข้ามาแล้วกด "Start exploring" ได้ทันที · ไม่ login ก็ใช้ได้ครบทุกอย่าง | `learning-01-landing` | `/explore` |
| 2 | **หาบทเรียนที่อยากอ่าน** | ช่องค้นหา · แท็บ ทั้งหมด/บุ๊กมาร์ก/ล่าสุด · กดรูป bookmark มุมการ์ดเพื่อเก็บไว้ (จำไว้ในเครื่องเรา ไม่ต้อง login) · แตะการ์ดเพื่อเปิด | `learning-02-explore` (with ≥1 bookmarked card) | `/explore` |
| 3 | **เปิดบทเรียนแล้วอ่านสบายๆ** | เลื่อนอ่าน แถบบนหลบให้เอง · หัวข้อมีเลขให้อัตโนมัติ · (คอม) Ctrl+ลูกกลิ้ง = ซูม, Ctrl+0 = พอดีจอ | `learning-03-viewer` | test lesson |
| 4 | **เจอคำถามในบทเรียน — ลองตอบเลย** | คำถามมี 4 แบบ: เลือกตอบ · เขียนตอบ · ลากคำเติมช่องว่าง · พิมพ์เติมช่องว่าง · เลือก/เขียน/ลากแล้วกด **ส่ง** · ตอบผิดไม่มีโดนดุ — AI ชมความคิดเราก่อนเสมอ | `learning-04a-choice`, `learning-04b-write`, `learning-04c-blankdrag` (3-up or carousel) | — |
| 5 | **อ่านคำแนะนำจาก AI แล้วคุยต่อได้** | หลังส่ง AI จะค่อยๆ พิมพ์คำแนะนำ (ครั้งแรกอาจช้าหน่อย เซิร์ฟเวอร์เพิ่งตื่น 😴) · ชิปม่วงๆ ใต้คำตอบ = คำถามชวนคิดต่อ แตะส่งได้เลย · "ส่งคำตอบเพิ่มเติม?" = คุยกับ AI ต่อในข้อนั้น · "ลองใหม่" = เริ่มข้อนั้นใหม่หมด | `learning-05-feedback` (streamed feedback + chips + open thread) | — |
| 6 | **สงสัยอะไร ถาม AI ได้ตลอด** | ปุ่ม **Ask AI** ลอยมุมขวาล่างทุกบทเรียน · AI รู้ว่าเรากำลังอ่านตรงไหนอยู่ · บางบทเรียนครูฝังกล่อง "ถาม AI" ไว้ในเนื้อหาเลย · ปุ่ม Clear chat เริ่มบทสนทนาใหม่ | `learning-06-askai` (modal open, streamed reply, chips) | — |
| 7 | **เลือกนิสัยติวเตอร์ให้ถูกจริต** | 6 บุคลิก: 🥔 คลาสสิก · 🌷 อ่อนโยน · 😏 ขี้แซว · 🔍 ละเอียด · ⚡ กระชับ · 🎯 จริงจัง · เปลี่ยนได้ในหน้าตั้งค่า หรือบนหัวกล่อง Ask AI · มีผลกับ AI ทุกจุด จำไว้ให้อัตโนมัติ | `learning-07-personality` (PersonalityPicker ใน Settings) | `/settings` |
| 8 | **login แล้วได้อะไรเพิ่ม (ไม่บังคับนะ)** | ทุกอย่างข้างบนใช้ได้โดยไม่มีบัญชี · สมัครแล้วได้เพิ่ม: ประวัติการเรียน (`/history`) · คำตอบถูกบันทึกข้ามเครื่อง · **ติวเตอร์จำเราได้** ข้ามบทเรียน — ดู/ลบความจำได้ที่โปรไฟล์ | `learning-08a-history`, `learning-08b-memory` (TutorMemoryCard) | `/login` |
| 9 | **ปรับแอปให้เป็นของเรา** | ธีมมืด/สว่าง · ขนาดตัวอักษร 4 ระดับ (ทั้งแอป) · ภาษาไทย/อังกฤษ · เว็บมีปัญหา? หน้า `/status` + ปุ่ม Help ติดต่อคนทำ | `learning-09-settings` | `/settings` |

Final CTA block: **"พร้อมแล้ว ไปเลือกบทเรียนเลย →"** (`/explore`) + cross-link "เป็นครูอยากสร้างบทเรียนเอง? → คู่มือครู" (`/guide/creating`).

💡 Tip boxes worth planting: scene 4 — "ข้อเขียนตอบไม่มีถูก/ผิด AI ดูวิธีคิดเรา"; scene 5 — cold-start เป็นเรื่องปกติ ไม่ใช่เว็บพัง; scene 8 — bookmark กับบุคลิกติวเตอร์อยู่ในเครื่องเราอยู่แล้ว ไม่ต้อง login.

---

## 5. creating-showcase — scene script (10 scenes)

Route `/guide/creating` · working title **"สร้างบทเรียนยังไง"**. Audience: a Thai teacher who "got their first ChatGPT training a week ago". The spine follows the editor's own AI-hub numbering (เริ่ม → เขียน → คำถาม → เผยแพร่) so the guide teaches the same mental model as the sidebar. Screenshots: desktop 1280 primary (editor is desktop-only); the page still lays out fine on phones.

| # | Scene | Steps to show | Screenshot(s) |
| --- | --- | --- | --- |
| 1 | **เตรียมตัวก่อนสร้าง** | สมัคร/เข้าสู่ระบบ (ฟรี ทุกบัญชีสร้างบทเรียนได้) · ⚠️ **หน้าสร้างบทเรียนใช้บนคอมพิวเตอร์เท่านั้น** (มือถือใช้อ่านได้อย่างเดียว) · เมนู "สร้าง" บนแถบบน → หน้ารวมบทเรียนของเรา | `creating-01-create-page` |
| 2 | **สร้างบทเรียนแรก** | กด "สร้างบทเรียนใหม่" · ได้บทเรียนว่างทันที ไม่ต้องกรอกอะไรก่อน (ชื่อ/คำอธิบายค่อยใส่ตอนเผยแพร่) · เข้าสู่หน้า editor อัตโนมัติ | `creating-02-blank-editor` (with the "หน้าว่างอยู่ใช่ไหม ให้ AI ช่วยเริ่มได้นะ ✨" CTA visible) |
| 3 | **รู้จักหน้าจอ 4 ส่วน** | บน: ชื่อเรื่อง + สถานะบันทึก + ย้อนกลับ/ทำซ้ำ + ซูม + ตรวจบทเรียน + **เผยแพร่** · ซ้าย: 5 หมวดเครื่องมือ (AI นำ) · กลาง: กระดาษของเรา · ขวา: คุณสมบัติของสิ่งที่เลือกอยู่ · **ระบบบันทึกให้เองทุก 30 วิ** (Ctrl+S ได้ถ้าใจร้อน) | `creating-03-editor-annotated` (arrows/labels overlay on the 4 regions) |
| 4 | **เขียนเนื้อหา** | บรรทัดแรก = H1 ชื่อบทเรียน · หัวข้อใหญ่ใช้ Heading 2 — เลข 1. 2. 3. มาเอง · แผง Text ซ้าย: โครงสร้าง/จัดวาง/สี/ไฮไลต์/ลิสต์/ตาราง/เช็กลิสต์ · เลือกข้อความแล้วแผงขวาจะมี ตัวหนา/เอียง/ขีดเส้น/ลิงก์ + ค้นหา-แทนที่ | `creating-04-writing` |
| 5 | **แทรกรูปในบทเรียน** | แผง Media: อัปโหลดจากเครื่อง/URL หรือคลิกจากคลังภาพ (ลากรูปวางในเอกสารก็ได้) · คลิกรูปแล้วครอป/ย่อขยาย/จัดวางจากแผงขวา · คลังภาพเต็มๆ อยู่ที่ `/uploadimage` · ตาราง/เช็กลิสต์อยู่ในแผง Text (ฉาก 4) | `creating-05a-media` only |
| 6 | **สูตรคณิต — ไม่ต้องรู้ LaTeX** | แผง Formula → "เพิ่มบล็อกสูตร" · กดปุ่มสัญลักษณ์ประกอบเอง หรือ · **"ให้ AI เขียนสูตร"**: พิมพ์แบบคน (`s = ut + 1/2at^2`) + บอกว่าคือสูตรอะไร → ได้สูตรสวยทันที · แก้มือต่อได้เสมอ | `creating-06-formula` (AI panel open + rendered KaTeX) |
| 7 | **สร้างคำถามชวนคิด (หัวใจของเว็บนี้)** | แผง Question: 5 แบบ — เลือกตอบ / เขียนตอบ / เติมคำ(ลาก) / เติมคำ(พิมพ์) / กล่องถาม AI · **แนวเฉลยสำคัญมาก**: AI ติวเตอร์ใช้มันสอนนักเรียน (ให้ AI ร่างให้ได้ · ตัวลวงก็ให้ AI เสนอได้) · โหมดคำแนะนำ: เข้าใจแบบย่อ ⇄ สะท้อนละเอียด · ปุ่มรูปตา = ดูตัวอย่างแบบนักเรียน | `creating-07a-question-panel`, `creating-07b-question-creator` (QuestionWrite with AI guide-answer preview) |
| 8 | **ให้ AI ช่วยทั้งเส้นทาง** | หมวด AI (ไอคอน ✨ บนสุดของแถบซ้าย) เรียงตามขั้นตอน: **1 เริ่ม** ร่างโครง / วางเนื้อหาเดิมที่มีอยู่แล้ว (เวิร์ดเก่า, ชีท, แชต GPT) → **2 เกลา** เติมเนื้อหาในหัวข้อ + ปรับข้อความ 6 แบบ (แก้คำผิด/จัดหัวข้อ/เกลา/ย่อ/ขยาย/ปรับระดับชั้น) → **3 คำถาม** สร้างเป็นชุด เลือกเก็บทีละข้อ → กติกาเดียวที่ต้องรู้: **AI เสนอ เราเลือก** — ไม่มีอะไรลงเอกสารจนกว่าเราจะกดรับ | `creating-08a-ai-hub`, `creating-08b-draft-preview` (before/after or preview cards) |
| 9 | **ตรวจ แล้วเผยแพร่** | "ตรวจบทเรียน" = AI รีวิวความครบ/อ่านง่าย/เหมาะกับวัย (แค่คำแนะนำ ไม่บังคับแก้) · กด **เผยแพร่** → ตั้งชื่อ+รูปปก+หัวข้อ+คำอธิบาย (ปุ่ม "ให้ AI ช่วยกรอก" มี) · **การมองเห็น 3 แบบ**: สาธารณะ (ขึ้นหน้า Explore) / เฉพาะลิงก์ (เห็นเฉพาะคนมีลิงก์) / ส่วนตัว · ตั้งนิสัย AI ติวเตอร์ประจำบทเรียนได้ (ชวนคิดก่อน ⇄ ตอบตรง, ขอบเขตการคุย) — ปุ่ม "ให้ AI แนะนำ" ก็มี | `creating-09a-critic`, `creating-09b-publish-modal` |
| 10 | **แชร์ให้นักเรียน แล้วดูผ่านตาเขา** | ใน "การแชร์": **QR code** ให้นักเรียนสแกน (แตะเพื่อขยายเต็มจอ ฉายขึ้นโปรเจกเตอร์ได้) หรือ **คัดลอกลิงก์** ไปวางใน LINE/Facebook · นักเรียนเปิดได้เลย **ไม่ต้องมีบัญชี** · กด "เผยแพร่ตอนนี้" ระบบพาไปหน้าที่นักเรียนเห็นจริง — เจอจุดแก้ กดปุ่มดินสอ (มุมขวาล่าง) กลับมาแก้ได้ · ลบบทเรียน: ปุ่มลบในหน้าต่างเผยแพร่ | `creating-10-share` (QR + copy link visible) |

Final CTA block: **"เริ่มบทเรียนแรกของคุณเลย →"** (`/create`) + cross-link "อยากเห็นมุมนักเรียนก่อน? → คู่มือนักเรียน" (`/guide/learning`).

💡 Tip boxes: scene 2 — งานหายไม่ได้ มี autosave + ปุ่มย้อนกลับ; scene 8 — วางเนื้อหาเดิม (import) คือทางลัดที่สุดสำหรับครูที่มีชีทอยู่แล้ว; scene 10 — ลิงก์แชร์ตอนนี้ขึ้น preview เป็นการ์ดกลางของเว็บ (การ์ดรายบทเรียนมาทีหลัง — ตรงกับ ROADMAP Tier 6.D).

**Deliberately left out of v1** (too deep for a first-run tutorial; the UI teaches them in place): Live/คงที่ toggle, link-click-mode toggle, collaborators-by-user-ID, code block, YouTube embed (no insert button today — see §9 Q5), **Fabric draw board / กระดานแคนวาส** (see §9 Q7 — complex sub-editor, rarely used; no `creating-05b-canvas` shot), Fabric object-property details, Image Vault transforms.

---

## 6. Screenshot manifest (drives `capture-guide.mjs`)

Naming: `<page>-<NN><letter?>-<slug>.webp` in `client/public/guide/`. Phone = 390×844, desktop = 1280×800. States assume local dev (`client :5173`, `server :5000`), test lesson `69e39d0b60d467bd515a4945`, demo teacher account via env `GUIDE_DEMO_EMAIL`/`GUIDE_DEMO_PASSWORD`. Scenes flagged **AI** hit real Gemini (needs `GEMINI_API_KEY`; costs tokens — rerun sparingly, support `--scene <id>`).

| File | URL / state | Auth | Viewport | Pre-actions |
| --- | --- | --- | --- | --- |
| learning-01-landing | `/` | anon | phone | — |
| learning-02-explore | `/explore` | anon | phone | bookmark 1 card first |
| learning-03-viewer | `/view/<lesson>` | anon | phone | scroll to a numbered H2 |
| learning-04a-choice | lesson choice block | anon | phone | select an answer (pre-submit) |
| learning-04b-write | lesson write block | anon | phone | type a short answer |
| learning-04c-blankdrag | blank-choice block | anon | phone | drag 1 chip into a blank |
| learning-05-feedback **AI** | choice block, post-submit | anon | phone | submit → wait stream end → open follow-up |
| learning-06-askai **AI** | viewer + modal | anon | phone | tap Ask AI FAB → send 1 question → wait |
| learning-07-personality | `/settings` | anon | phone | scroll to AI Tutor section |
| learning-08a-history | `/history` | **login** | phone | visit 2–3 lessons first |
| learning-08b-memory | `/profile` | **login** | phone | needs non-empty StudentMemory |
| learning-09-settings | `/settings` | anon | phone | top of page |
| creating-01-create-page | `/create` | login | desktop | ≥1 existing lesson |
| creating-02-blank-editor | `/canvas/<new>` | login | desktop | fresh blank lesson (AI CTA visible) |
| creating-03-editor-annotated | `/canvas/<lesson>` | login | desktop | annotate 4 regions post-capture |
| creating-04-writing | editor, text panel active | login | desktop | select text so right panel shows format |
| creating-05a-media | editor, Media panel | login | desktop | vault has ≥4 images |
| ~~creating-05b-canvas~~ | — | — | — | **Omitted v1** (§9 Q7): Fabric draw board too complex / rarely used |
| creating-06-formula **AI** | formula block + AI panel | login | desktop | run formula AI on `s = ut + 1/2at^2` |
| creating-07a-question-panel | editor, Question panel | login | desktop | — |
| creating-07b-question-creator **AI** | QuestionWrite block | login | desktop | trigger AI guide-answer preview |
| creating-08a-ai-hub | editor, AI panel | login | desktop | — |
| creating-08b-draft-preview **AI** | AiDraftDialog outline tab | login | desktop | generate outline for "การเคลื่อนที่" |
| creating-09a-critic **AI** | AiCriticDialog open | login | desktop | run critic on test lesson |
| creating-09b-publish-modal | PublishSettingsModal | login | desktop | fill some fields; access = สาธารณะ |
| creating-10-share | Publish modal, sharing section | login | desktop | saved lesson → QR + link visible |

Post-process (same script): PNG → WebP q≈80, cap width 800 px (phone) / 1280 px (desktop), emit width/height into a generated `guideImages.ts` manifest so `<img>` tags get dimensions (no CLS) without hand-typing.

---

## 7. Page architecture

### Routes & files

```
client/src/pages/guide/
├── GuideHub.tsx            # replaces the body of pages/Guide.tsx (route /guide)
├── LearningShowcase.tsx    # route /guide/learning  (lazy)
├── CreatingShowcase.tsx    # route /guide/creating  (lazy)
├── learningScenes.ts       # typed scene data (9 scenes, {en, th} strings)
├── creatingScenes.ts       # typed scene data (10 scenes)
└── components/
    ├── ShowcaseShell.tsx   # page frame: header, scene TOC/progress, prev/next, final CTA
    ├── SceneSection.tsx    # one scene: number badge, title, image, steps, tip, deep link
    └── SceneImage.tsx      # <img loading="lazy"> + width/height from guideImages.ts
```

- `App.tsx`: two new lazy routes inside `AppLayout`, same `<Suspense>` as the rest (Tier 2.A pattern). `/guide` keeps rendering `Guide.tsx` whose content becomes the hub.
- **Scene data over JSX**: scenes are data (`{ id, title, steps[], tip?, image(s), cta? }` with `t()`-ready `{en, th}` pairs) so copy edits never touch layout, and tests can assert scene counts/ids.
- **Hub** (`GuideHub.tsx`): intro paragraph → two big role cards (นักเรียน → `/guide/learning`, ครู → `/guide/creating`) → smaller "ไปลองเองเลย → Explore" card → footer row (Help popup pattern + `/status`). No screenshots needed — ships before the pipeline if convenient.
- **TOC/progress**: in-page anchor nav (scene numbers) — sticky horizontal chip row on phone, side rail on desktop. Anchors double as future deep links (`#scene-4`, Tier G6).
- **Images**: static URLs (`/guide/learning-01-landing.webp`), `loading="lazy"` below the fold, explicit dimensions. Creating-page desktop shots render in an `overflow-x` safe container capped to content width — never force horizontal page scroll at 390 px.

### Tests (vitest + happy-dom, alongside pages)

- Hub renders both role cards + links (both languages).
- Each showcase renders all scene ids in order; scene count pinned (9/10); every scene image has dimensions + lazy attr.
- Deep-link CTAs point at real routes (guards against route renames).
- A tiny lint-style test that entry chunk files never import from `public/guide` (budget guard, mirrors `no-cdn-fonts.test.ts` spirit).

---

## 8. Copy & tone guidelines

- **Voice = น้องมันฝรั่ง's world**: warm, short Thai sentences, emoji sparingly (🥔 allowed), zero tech jargon. Say "กดปุ่มส่ง" not "submit the form". English side of `t()` is plain and friendly, not a literal translation.
- **Never shame the reader**: same product principle as the tutor — no "แค่กดตรงนี้ก็ยังไม่รู้อีกเหรอ" energy. Steps state what to press and what happens next.
- **Honesty beats polish**: cold starts ("เซิร์ฟเวอร์เพิ่งตื่น 😴"), generic link previews, editor-needs-desktop — say them plainly; surprises erode trust with exactly this audience.
- **Quote the UI verbatim**: button names in the copy must match the rendered strings (e.g. "ส่งคำตอบเพิ่มเติม?", "ให้ AI เขียนสูตร") — the inventory tables in §2–§3 carry the verified strings; recheck against the live UI during capture.
- **{BRAND} placeholder** everywhere the product is named, until the Intuita-vs-Hot-Potato decision (§9 Q1) lands.

---

## 9. Open questions for the owner (placeholders fine until answered)

| # | Question | Status |
| --- | --- | --- |
| Q1 | **Brand name in guide copy** — UI says "Intuita", docs say "Hot Potato", tutor is น้องมันฝรั่ง. | **Answered 2026-07-11: no brand exists yet** 😂 — working name "Hot Potato" via `client/src/lib/brand.ts` `BRAND_NAME` (one-line rename later). |
| Q2 | Route + display names — `/guide/learning` "เรียนยังไง" / `/guide/creating` "สร้างบทเรียนยังไง" OK? | Shipped with these names; still renamable. |
| Q3 | Demo lesson (Tier G6 / decision G7) — owner authors "บทเรียนสอนใช้เว็บ" later. | Slot still reserved. NB: G1.A already created a *screenshot* demo lesson ("ทำไมท้องฟ้าเป็นสีฟ้า?", link-only) — could be its seed. |
| Q4 | `/dashboard` vs `/create` duplication. | **Answered 2026-07-11: Dashboard is dead** — replaced by `/create`; guide teaches `/create` only. Removal is a separate cleanup task. |
| Q5 | YouTube — extension registered but **no insert button**; embeds only via markdown/paste. Verify during G4 capture; if unverified, videos stay out of the media scene. | **Answered 2026-07-11 (during G4): confirmed no insert button** in the editor rail (Media panel = upload device/URL/gallery; Text panel = table/checklist/canvas). Media scene 5 teaches images only — videos stay out of v1. |
| Q6 | Owner cameo — Help popup says "เว็บนี้ทำฟรีโดยคนคนเดียว 😄"; want the same personal touch on the hub? | Hub footer links "ทักหาคนทำเว็บ" (Facebook) + status — cameo copy still owner's call. |
| Q7 | **Fabric draw board (กระดานแคนวาส)** — include in creating-showcase scene 5? | **Answered 2026-07-11 (post-G4): omitted from v1.** The Fabric canvas sub-editor swaps both sidebars, has its own templates/shapes/pen workflow, and is rarely used in practice — too complex for a first-run teacher tutorial. Scene 5 teaches images/media only (`creating-05a-media`); tables/checklists stay in scene 4. The `creating-05b-canvas` screenshot and copy were removed from `creatingScenes.ts`. |

**Side findings from the inventory sweep** (not guide scope — parked for the owner): `ChangePassword` page is English-only (only non-bilingual student page left); Dashboard is English-only; the editor's Live/Static and link-click-mode toggles are the least self-explanatory controls in the app (candidates for the future spotlight tour, Tier G6).

---

## 10. Phase mapping (execute in this order)

| Roadmap phase | What this file gives it | Ship gate |
| --- | --- | --- |
| **G1.A** demo data | §6 state column: verify test lesson has all 4 gradeable types + 1 QuestionAgent block; create demo teacher account | checklist done, documented in script README |
| **G1.B** capture script | §6 manifest → `client/scripts/capture-guide.mjs` (playwright devDep) + WebP post-process + `guideImages.ts` emit | one command → full image set; `--scene` works |
| **G2** hub | §7 hub spec | hub live, lazy routes wired, bundle unchanged, tests |
| **G3** learning-showcase | §4 scenes + §7 shell + §8 tone | 9 scenes, phone-first pass, tests |
| **G4** creating-showcase ✅ | §5 scenes (shell reused) | 10 scenes, desktop shots safe at 390 px, tests — shipped `1abb1ef` |
| **G5** polish | cross-links (Landing, Settings help popup), SEO titles, docs | all entry points reach hub; CLAUDE.md updated |

Working agreement: one phase = one commit in `client/`, message style `feat(guide): <phase>`, tests green before commit, no pushes.
