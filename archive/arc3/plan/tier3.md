# Tier 3 — Settings & Profile completeness: Execution Plan

**For the implementing agent (Cursor).** This is a complete, self-contained spec for **Tier 3 of [`../ROADMAP.md`](../ROADMAP.md)** (launch-readiness v3): Phase 3.A (`/settings` cleanup + a real app-wide font-size control) and Phase 3.B (`/profile` completion — the tutor-memory UI students have never seen). Execute top to bottom. Every file path, contract, code block, and test case you need is in this document — **but always read the referenced source file before editing it**; this plan describes the code as of 2026-07-10 and the file is the truth.

> **Read first, in this order:** [`../AGENT.md`](../AGENT.md) → [`../CLAUDE.md`](../CLAUDE.md) → [`../client/CLAUDE.md`](../client/CLAUDE.md). Short, and they stop you from breaking things this plan assumes you know.
> **Theme of this tier:** *no dead buttons anywhere a student can tap.* Today `/settings` renders five rows that do nothing, and the server has remembered things about students since Phase 3 (`StudentMemory`) **with zero UI** — students can't see or delete what the AI knows about them. Tier 3 fixes both.
> **Do not improvise beyond this plan.** When a real ambiguity appears, prefer the smallest change that satisfies the acceptance criteria and note it in §7.

---

## 0. Ground rules (non-negotiable)

### The two Golden Rules
1. **Never ration tokens for real students.** Not directly touched by this tier — never add any limit/quota logic while here.
2. **Anonymous users keep FULL access.** `/settings` is a **public** page (no route guard — verified in `App.tsx`) and must stay public. After 3.A, an anonymous visitor must see **zero** dead or broken rows: theme, language, font size, personality picker, and Help & Support all work logged-out. `/profile` is `RequireLogin` (soft gate) — the memory card only ever mounts for logged-in users, which is exactly right.

### Owner decisions that shape this tier (do not re-litigate)
| # | Decision |
| --- | --- |
| D3 | No PDPA/privacy page. The delete-**account** flow is deferred to Tier 6 — that's why the *Delete account* row gets **cut**, not wired. |
| D4 | `/settings` keeps only what works. Appearance (font size) is the one thing to *build*; Help & Support becomes a popup with the owner's Facebook contact; dead rows get cut. |
| — | The owner supplies the real Facebook URL later — use the placeholder constant (§2 Step A2) and add the swap to the Tier 5 launch checklist (§4). |

### Scope fence — Tier 3 touches ONLY the client
- **All work happens in `client/`. Zero server changes.** Both server endpoints 3.B needs (`GET`/`DELETE /api/chat/memory`) have existed since Phase 3 and are verified in §1.4.
- Files you will create: `src/stores/appearance.store.ts`, `src/stores/tutorMemory.store.ts`, `src/lib/contact.ts`, `src/components/TutorMemoryCard.tsx`, plus 4 test files.
- Files you will edit: `src/pages/Setting.tsx` (full rewrite, §2), `src/pages/Profile.tsx` (surgical, §3), `src/pages/__tests__/Profile.test.tsx`, `client/CLAUDE.md`, `../ROADMAP.md`. Nothing else.
- **Never touch:** anything in `server/`, `persona.ts`/prompts (owner is tone referee — none planned), `example_copy/`, `example_project/`, `test/` (root), `Ptest/`, `archive/`, `graphify-out/`.

### Things you will see and must NOT fix (owned elsewhere)
| Thing | Where | Owner |
| --- | --- | --- |
| ESLint error `'__dirname' is not defined` | `client/vite.config.js` | Tier 4. `npm run lint` having exactly this one pre-existing error is expected (unless Tier 4 already landed). |
| Viewer zoom via CSS `zoom` on the card container | `TiptapViewer` | **Careful-not-to-break list.** The font-size control is a *separate, app-wide* mechanism. Do not merge them, do not touch the viewer's zoom, do not use `transform: scale` anywhere (it reintroduces the double-scrollbar bug). |
| `tutorPersonality.store.hydrateFromServer` reads `GET /chat/memory` | `src/stores/tutorPersonality.store.ts` | Nobody — leave it exactly as-is. Your new memory store reads the same endpoint; that's fine, they don't conflict. |
| `--app-nav-height: 56px` (px, not rem) | `src/index.css` (:root, both themes) + `indexTiptap.css` grid | Deliberate. The nav bar will NOT grow with font size — its *content* will. Do not convert it to rem (the editor grid depends on it). If nav content overflows at the largest size, let text truncate; never grow the var. |
| Duplicate `CloudinaryUpload` files | client | Nobody yet — leave both. |

### Tier-order reality (verified 2026-07-10)
As of writing, **only Tier 0.A has shipped** (client HEAD `e3c53ec`, 74 tests; server HEAD `0f748eb`, 203 tests). Tiers 1 and 2 have their own plans in this folder and may or may not have landed before you run this one — **Tier 3 depends on neither.** Consequences:
- Do not hard-pin test counts. Whatever `npm test` shows green at preflight is your baseline; it must not decrease.
- If Tier 2.A landed, `App.tsx` imports pages via `React.lazy` — irrelevant to this tier; the settings page file is still `src/pages/Setting.tsx` (singular) either way.
- If `npm run lint` is fully green (Tier 4 landed), keep it green.

### Git reality ⚠️
- `client/` and `server/` are **two separate git repositories**; no repo at the `Hot-Potato/` root. If `git status` shows foreign files (e.g. `ChronoForge-FPGA-Engine`), you are in the wrong directory — stop and `cd client`.
- **One commit per phase**, in `client/` only:
  - `feat(tier3.A): settings cleanup — cut dead rows, help popup, app-wide font size`
  - `feat(tier3.B): profile completion — tutor memory card + personality shortcut`
- **Never push** (owner decision D1: nothing is pushed until Tier 5 launch day).

### Commands & definition of done
```bash
cd client && npm install && npm run dev   # web on :5173
cd client && npm test                     # vitest + happy-dom — green before and after
cd client && npm run build                # must stay green
```
- Server for manual smoke: `cd server && npm run dev` (API on :5000). Test lesson: `http://localhost:5173/view/69e39d0b60d467bd515a4945`.
- **Definition of done per phase:** new tests for the phase's acceptance criteria written and passing, `npm test` + `npm run build` green, manual pass done in **both auth states** (logged in + incognito) at desktop **and ~390 px**, `client/CLAUDE.md` updated, one commit in `client/`.
- Env vars: **add none.**

### Test conventions in this repo (match them exactly)
- Vitest + **happy-dom**. There is **no `@testing-library/react`** — do not install it, or anything else. Tests render with `react-dom/client` `createRoot` + `act` from `react`, wrap in `MemoryRouter` when the component uses `Link`/`useNavigate`, and stub stores/axios with `vi.mock`. Copy the harness from `src/pages/__tests__/Profile.test.tsx` (render helper, `setInputValue`, `afterEach` cleanup).
- Test files live in `__tests__/` folders next to the code.
- **Radix portals render into `document.body`**, not your container — dialog assertions must check `document.body.textContent` (see §2 Step A3 and §6).

---

## 1. Verified current state (trust this over guesses; re-verify against the file if something looks off)

### 1.1 `/settings` — `src/pages/Setting.tsx` (234 lines)

Route: `<Route path="/settings" element={<Settings />} />` inside `AppLayout` — **no guard, public**. The page renders, in order:

| Section | Row | Status |
| --- | --- | --- |
| Appearance (hand-built) | Theme + `ThemeToggle` | ✅ real — **keep** |
| Language (hand-built) | EN/TH segmented buttons via `language.store` | ✅ real — **keep** |
| AI Tutor (hand-built) | `PersonalityPicker` | ✅ real — **keep** |
| "Preferences" (from `sections` array) | *Notifications* ("Push & email alerts") | ❌ dead — **cut** |
| "Preferences" | *Appearance* ("Theme, font size, display") | ❌ dead duplicate — **cut** (the row's promise, font size, is what Step A2 actually builds) |
| "Account" | *Privacy & Security* ("Password, 2FA, sessions") | ❌ dead — **wire** to `/change-password`, rename to Password, logged-in only |
| "Account" | *Help & Support* ("FAQ, contact us") | ❌ dead — **becomes the popup** |
| "Danger zone" (token-gated) | *Log out* | ✅ real — **keep** |
| "Danger zone" | *Delete account* | ❌ dead destructive — **cut** (returns with Tier 6.A) |

Mechanics worth knowing before the rewrite: the page uses a **local** `t()` helper (not `useAppI18n`), and the logout row is dispatched by the fragile comparison `item.label === t("Log out", …)`. The rewrite (Step A2) kills both: `useAppI18n` for copy, explicit `onClick` per row.

### 1.2 `/profile` — `src/pages/Profile.tsx` (299 lines)

Route: `RequireLogin` soft gate (anonymous users get an inline sign-in prompt — the memory card never mounts for them). The page is real (Tier 2.A of the old roadmap): avatar upload → Cloudinary, name/nickname/bio → `PUT /users/me/profile` via `profile.store`, change-password button, `savedFlash` inline confirmation pattern (state + 2 s `setTimeout` — reuse this idiom for "memory cleared", there is **no sonner `<Toaster>` mounted anywhere**, so `toast()` calls would silently no-op).

To remove: the **Theme row** (first block inside the form `div`, lines ~197–209: `{t("Theme", "ธีม")}` + `ThemeToggle`) — it duplicates Settings; one source of truth (roadmap 3.B). With it go the now-unused `ThemeToggle` / `useThemeStore` imports and `const { theme } = useThemeStore();`.

### 1.3 Store patterns to copy

- **`src/stores/theme.store.ts` is the exact template for the new appearance store**: plain `create()` (no `persist` middleware), module-level `detectInitial…()` + `apply…()` run at import time so the setting takes effect before React mounts, manual `localStorage` write in the setter. Mirror it precisely.
- `src/stores/profile.store.ts` is the template for the new memory store (fetch/save + `isLoading`/`error` state consumed by a page).
- `src/lib/i18n.ts` → `useAppI18n()` returns `{ language, isThai, t }` where `t(english, thai)`.

### 1.4 The memory API contract (verified against `server/src/controllers/memory.controller.ts` + `models/student_memory.model.ts`)

- **`GET /api/chat/memory`** (auth `protect`): returns the **raw StudentMemory doc** (`.lean()`, minus `__v`) — or **`{}`** when the student has no memory yet. Fields:
  - Public, render these: `interests: string[]`, `strengths: string[]`, `growth_areas: string[]`, `preferences: string[]` (each capped at 10 items ≤ 120 chars), `recent_topics: { content_id, summary, updatedAt }[]` (capped at 15).
  - **Internal, never render:** `_id`, `user_id`, `tutor_personality`, `createdAt`, `updatedAt`. The store normalizes them away (Step B1).
- **`DELETE /api/chat/memory`** (auth `protect`): deletes the **whole doc**, returns `200 { message: "Memory cleared" }`.
- ⚠️ **Known interaction — expected, do not "fix":** deleting the doc also wipes the server-synced `tutor_personality` field. Harmless: the student's choice lives client-side in `tutor-personality-storage` (persisted zustand) and the server re-syncs it on the next tutor call (`tutor.controller.ts` `$set`s it on every call). Say nothing, change nothing.
- Memory content is written by an **async background digest** after tutor conversations — a freshly logged-in student who has never chatted gets `{}`; a student who chatted seconds ago may not see updates until the next fetch. The empty state (Step B2) is a first-class state, not an error.

### 1.5 UI primitives available (verified)

- **`radix-ui` umbrella package v1.4.3** is installed. `src/components/ui/` has: `alert-dialog.tsx` (full shadcn set: `AlertDialog`, `AlertDialogContent/Header/Title/Description/Footer/Action/Cancel/Trigger`), `button.tsx` (variants `default | outline | ghost | destructive`, sizes `default | sm | lg | icon`), `card`, `dropdown-menu`, `input`, `label`, `select`, `textarea`. There is **no `dialog.tsx` — do not add one.** Both popups in this tier (Help & Support, delete-memory confirm) use `AlertDialog`, controlled via `open`/`onOpenChange` state so tests can drive them.
- Icons: `lucide-react` (use `Type`, `Brain`, `Trash2`, `Loader2`, `Shield`, `HelpCircle`, `Globe`, `LogOut`, `ChevronRight` — all standard).
- Page card idiom is hand-rolled `rounded-lg border border-border bg-card px-4 py-3` — match it, don't reach for the `Card` component the pages don't use.

### 1.6 Why the font-size mechanism works (and its edges)

Tailwind sizes are rem-based and `index.css` never sets an explicit `font-size` on `html` (line 103 is only `@apply font-sans`) — so an **inline `style.fontSize` on `document.documentElement`** rescales the entire app: UI, chat, lesson text. Percentages (not px) respect the user's own browser-default setting, and clearing the inline style for "normal" keeps the browser fully in charge. Edges to verify by hand, not code around: the px-based nav height (§0 table) and the viewer's independent zoom.

---

## 2. Phase 3.A — `/settings` cleanup + Appearance (font size)

**Goal:** every visible row on `/settings` does something, for both auth states; a persisted 4-step font-size control scales the whole app. Size **M**, repo: `client/` only.

### Step A0 — Preflight
1. `cd client && npm install && npm test` → record the passing count in §7 ("Baseline"). If red, STOP — report, don't build on a broken base.
2. `npm run build` → green.

### Step A1 — New file `src/stores/appearance.store.ts` (complete content)

```ts
import { create } from "zustand";

export type FontSize = "small" | "normal" | "large" | "xlarge";

export const FONT_SIZE_OPTIONS: readonly {
  id: FontSize;
  css: string;
  labelTh: string;
  labelEn: string;
}[] = [
  { id: "small", css: "87.5%", labelTh: "เล็ก", labelEn: "Small" },
  { id: "normal", css: "100%", labelTh: "ปกติ", labelEn: "Normal" },
  { id: "large", css: "112.5%", labelTh: "ใหญ่", labelEn: "Large" },
  { id: "xlarge", css: "125%", labelTh: "ใหญ่มาก", labelEn: "Extra large" },
] as const;

interface AppearanceState {
  fontSize: FontSize;
  setFontSize: (size: FontSize) => void;
}

const FONT_SIZE_STORAGE_KEY = "app-font-size";

const isFontSize = (value: unknown): value is FontSize =>
  FONT_SIZE_OPTIONS.some((option) => option.id === value);

const detectInitialFontSize = (): FontSize => {
  if (typeof window === "undefined") return "normal";
  const saved = window.localStorage.getItem(FONT_SIZE_STORAGE_KEY);
  return isFontSize(saved) ? saved : "normal";
};

const applyFontSize = (size: FontSize) => {
  if (typeof document === "undefined") return;
  const option = FONT_SIZE_OPTIONS.find((o) => o.id === size);
  // "normal" clears the inline style so the browser default stays in charge
  // (rem-based Tailwind sizes then follow the user's own browser setting).
  document.documentElement.style.fontSize =
    !option || option.id === "normal" ? "" : option.css;
};

const initialFontSize = detectInitialFontSize();
applyFontSize(initialFontSize);

export const useAppearanceStore = create<AppearanceState>((set) => ({
  fontSize: initialFontSize,
  setFontSize: (size) => {
    applyFontSize(size);
    if (typeof window !== "undefined") {
      window.localStorage.setItem(FONT_SIZE_STORAGE_KEY, size);
    }
    set({ fontSize: size });
  },
}));
```

Design decisions baked in (don't reopen): percent values so the control *stacks* with the user's browser preference; `87.5% → 14px / 112.5% → 18px / 125% → 20px` at the 16 px default; module-level apply so the size is right before first paint (same trick as `theme.store`). The store is imported by `Setting.tsx`, which is enough for Vite to include the module-level init on every page that can reach settings — but to guarantee it on **every** route (a student who set ใหญ่มาก must get it on `/view/:id` too), add one import line to `src/main.tsx` next to the other imports:

```ts
import "./stores/appearance.store"; // applies persisted font size before first paint
```

### Step A2 — New file `src/lib/contact.ts` + rewrite `src/pages/Setting.tsx`

**`src/lib/contact.ts`** (complete content — a lib file, not the page, so the `react-refresh/only-export-components` lint rule can't complain):

```ts
/**
 * Owner's public contact. PLACEHOLDER — the owner supplies the real page URL
 * before launch (Tier 5 checklist item in ROADMAP.md).
 */
export const OWNER_FACEBOOK_URL = "https://www.facebook.com/";
```

**Replace the entire contents of `src/pages/Setting.tsx` with:**

```tsx
import { useState } from "react";
import {
  Globe,
  Shield,
  HelpCircle,
  LogOut,
  ChevronRight,
  Type,
} from "lucide-react";
import { Link, useNavigate } from "react-router-dom";
import { ThemeToggle } from "@/components/ThemeToggle";
import { useThemeStore } from "@/stores/theme.store";
import { useAuthStore } from "@/stores/auth.store";
import { useLanguageStore } from "@/stores/language.store";
import {
  useAppearanceStore,
  FONT_SIZE_OPTIONS,
} from "@/stores/appearance.store";
import PersonalityPicker from "@/components/editor/extensions/PersonalityPicker";
import { useAppI18n } from "@/lib/i18n";
import { OWNER_FACEBOOK_URL } from "@/lib/contact";
import {
  AlertDialog,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
} from "@/components/ui/alert-dialog";

export default function Settings() {
  const { theme } = useThemeStore();
  const token = useAuthStore((s) => s.token);
  const logout = useAuthStore((s) => s.logout);
  const language = useLanguageStore((s) => s.language);
  const setLanguage = useLanguageStore((s) => s.setLanguage);
  const fontSize = useAppearanceStore((s) => s.fontSize);
  const setFontSize = useAppearanceStore((s) => s.setFontSize);
  const navigate = useNavigate();
  const { isThai, t } = useAppI18n();
  const [helpOpen, setHelpOpen] = useState(false);

  const handleLogout = () => {
    logout();
    navigate("/explore", { replace: true });
  };

  return (
    <div className="container max-w-lg px-4 pb-12 pt-6">
      <h1 className="font-serif text-2xl font-bold">{t("Settings", "ตั้งค่า")}</h1>
      <p className="mt-1 text-sm text-muted-foreground">
        {t("Manage your account and preferences", "จัดการบัญชีและการตั้งค่าของคุณ")}
      </p>

      <div className="mt-6 space-y-6">
        {/* Appearance: theme + font size */}
        <div>
          <h2 className="mb-2 text-xs font-semibold uppercase tracking-wider text-muted-foreground">
            {t("Appearance", "หน้าตาแอป")}
          </h2>
          <div className="overflow-hidden rounded-lg border border-border bg-card">
            <div className="flex items-center justify-between px-4 py-3">
              <div>
                <p className="text-sm font-medium text-foreground">
                  {t("Theme", "ธีม")}
                </p>
                <p className="text-xs text-muted-foreground">
                  {t("Current mode", "โหมดปัจจุบัน")}:{" "}
                  {theme === "dark" ? t("Dark", "มืด") : t("Light", "สว่าง")}
                </p>
              </div>
              <ThemeToggle />
            </div>

            <div className="border-t border-border px-4 py-3">
              <div className="flex items-center gap-3">
                <Type className="h-4 w-4 shrink-0 text-muted-foreground" />
                <div>
                  <p className="text-sm font-medium text-foreground">
                    {t("Font size", "ขนาดตัวอักษร")}
                  </p>
                  <p className="text-xs text-muted-foreground">
                    {t("Applies to the whole app", "มีผลกับตัวหนังสือทั้งแอป")}
                  </p>
                </div>
              </div>
              <div className="mt-3 inline-flex flex-wrap gap-1 rounded-md border border-border p-1">
                {FONT_SIZE_OPTIONS.map((option) => (
                  <button
                    key={option.id}
                    type="button"
                    onClick={() => setFontSize(option.id)}
                    className={`rounded px-2.5 py-1 text-xs font-medium transition-colors ${
                      fontSize === option.id
                        ? "bg-primary text-primary-foreground"
                        : "text-muted-foreground hover:bg-accent"
                    }`}
                  >
                    {isThai ? option.labelTh : option.labelEn}
                  </button>
                ))}
              </div>
            </div>
          </div>
        </div>

        {/* Language */}
        <div>
          <h2 className="mb-2 text-xs font-semibold uppercase tracking-wider text-muted-foreground">
            {t("Language", "ภาษา")}
          </h2>
          <div className="flex items-center justify-between rounded-lg border border-border bg-card px-4 py-3">
            <div className="flex items-center gap-3">
              <Globe className="h-4 w-4 text-muted-foreground" />
              <div>
                <p className="text-sm font-medium text-foreground">
                  {t("Language", "ภาษา")}
                </p>
                <p className="text-xs text-muted-foreground">
                  {t("App display language", "ภาษาที่ใช้แสดงผลในแอป")}
                </p>
              </div>
            </div>
            <div className="inline-flex rounded-md border border-border p-1">
              <button
                type="button"
                onClick={() => setLanguage("en")}
                className={`rounded px-2.5 py-1 text-xs font-medium transition-colors ${
                  language === "en"
                    ? "bg-primary text-primary-foreground"
                    : "text-muted-foreground hover:bg-accent"
                }`}
              >
                {t("English", "อังกฤษ")}
              </button>
              <button
                type="button"
                onClick={() => setLanguage("th")}
                className={`rounded px-2.5 py-1 text-xs font-medium transition-colors ${
                  language === "th"
                    ? "bg-primary text-primary-foreground"
                    : "text-muted-foreground hover:bg-accent"
                }`}
              >
                {t("Thai", "ไทย")}
              </button>
            </div>
          </div>
        </div>

        {/* AI Tutor personality */}
        <div>
          <h2 className="mb-2 text-xs font-semibold uppercase tracking-wider text-muted-foreground">
            {t("AI Tutor", "AI ติวเตอร์")}
          </h2>
          <div className="rounded-lg border border-border bg-card px-4 py-3">
            <p className="text-sm font-medium text-foreground">
              {t("Choose how the tutor talks to you", "เลือกสไตล์การคุยของติวเตอร์")}
            </p>
            <p className="mb-3 text-xs text-muted-foreground">
              {t(
                "Applies to every AI chat in lessons.",
                "มีผลกับการคุย AI ทุกจุดในบทเรียน",
              )}
            </p>
            <PersonalityPicker />
          </div>
        </div>

        {/* Account */}
        <div>
          <h2 className="mb-2 text-xs font-semibold uppercase tracking-wider text-muted-foreground">
            {t("Account", "บัญชี")}
          </h2>
          <div className="overflow-hidden rounded-lg border border-border bg-card">
            {token && (
              <button
                type="button"
                onClick={() => navigate("/change-password")}
                className="flex w-full items-center gap-3 px-4 py-3 text-left transition-colors hover:bg-accent"
              >
                <Shield className="h-4 w-4 shrink-0 text-muted-foreground" />
                <div className="min-w-0 flex-1">
                  <p className="text-sm font-medium text-foreground">
                    {t("Password", "รหัสผ่าน")}
                  </p>
                  <p className="text-xs text-muted-foreground">
                    {t("Change your password", "เปลี่ยนรหัสผ่านของคุณ")}
                  </p>
                </div>
                <ChevronRight className="h-4 w-4 shrink-0 text-muted-foreground" />
              </button>
            )}
            <button
              type="button"
              onClick={() => setHelpOpen(true)}
              className={`flex w-full items-center gap-3 px-4 py-3 text-left transition-colors hover:bg-accent ${
                token ? "border-t border-border" : ""
              }`}
            >
              <HelpCircle className="h-4 w-4 shrink-0 text-muted-foreground" />
              <div className="min-w-0 flex-1">
                <p className="text-sm font-medium text-foreground">
                  {t("Help & Support", "ช่วยเหลือและสนับสนุน")}
                </p>
                <p className="text-xs text-muted-foreground">
                  {t("Contact & status", "ติดต่อเราและสถานะระบบ")}
                </p>
              </div>
              <ChevronRight className="h-4 w-4 shrink-0 text-muted-foreground" />
            </button>
          </div>
        </div>

        {/* Danger zone — logged-in only (Delete account returns with Tier 6.A) */}
        {token && (
          <div>
            <h2 className="mb-2 text-xs font-semibold uppercase tracking-wider text-muted-foreground">
              {t("Danger zone", "โซนอันตราย")}
            </h2>
            <div className="overflow-hidden rounded-lg border border-border bg-card">
              <button
                type="button"
                onClick={handleLogout}
                className="flex w-full items-center gap-3 px-4 py-3 text-left transition-colors hover:bg-accent"
              >
                <LogOut className="h-4 w-4 shrink-0 text-destructive" />
                <div className="min-w-0 flex-1">
                  <p className="text-sm font-medium text-destructive">
                    {t("Log out", "ออกจากระบบ")}
                  </p>
                  <p className="text-xs text-muted-foreground">
                    {t("Sign out of your account", "ออกจากบัญชีของคุณ")}
                  </p>
                </div>
                <ChevronRight className="h-4 w-4 shrink-0 text-muted-foreground" />
              </button>
            </div>
          </div>
        )}
      </div>

      {/* Help & Support popup */}
      <AlertDialog open={helpOpen} onOpenChange={setHelpOpen}>
        <AlertDialogContent>
          <AlertDialogHeader>
            <AlertDialogTitle>
              {t("Help & Support", "ช่วยเหลือและสนับสนุน")}
            </AlertDialogTitle>
            <AlertDialogDescription>
              {t(
                "Hot Potato is built and run for free by one person 😄 Found a bug or have an idea? Message me directly:",
                "เว็บนี้ทำฟรีโดยคนคนเดียว 😄 เจอปัญหาหรือมีไอเดียอะไร ทักมาคุยกันได้เลย:",
              )}
            </AlertDialogDescription>
          </AlertDialogHeader>
          <a
            href={OWNER_FACEBOOK_URL}
            target="_blank"
            rel="noreferrer"
            className="text-sm font-medium text-primary underline underline-offset-4"
          >
            {t("Message me on Facebook", "ทักแชทผ่าน Facebook")}
          </a>
          <p className="text-xs text-muted-foreground">
            {t("Think the site is down?", "สงสัยว่าเว็บล่มไหม?")}{" "}
            <Link
              to="/status"
              onClick={() => setHelpOpen(false)}
              className="text-primary underline underline-offset-4"
            >
              {t("Check the status page", "เช็กหน้าสถานะระบบ")}
            </Link>
          </p>
          <AlertDialogFooter>
            <AlertDialogCancel>{t("Close", "ปิด")}</AlertDialogCancel>
          </AlertDialogFooter>
        </AlertDialogContent>
      </AlertDialog>
    </div>
  );
}
```

What this rewrite deliberately changed, so you can verify rather than rediscover:
- The `sections`/`SettingRow` array machinery and the fragile `item.label === t("Log out", …)` dispatch are **gone** — every row has an explicit `onClick`.
- Cut rows: *Notifications*, duplicate *Appearance*, *Delete account* (and the icons `Bell`, `Moon`, `Trash2`).
- The Password row is wired to `/change-password` **and shown only when logged in**. (Deliberate refinement of the roadmap's one-liner: `/change-password` is `RequireLogin`, so for an anonymous visitor the row would be a dead end into a sign-in prompt — and the acceptance says anonymous users see *no dead rows*. Password management is account-scoped anyway.)
- Local `t` helper replaced with `useAppI18n` (same behavior, one source of truth).
- Both dialog and page use the **controlled** `open` state so tests can drive them without Radix trigger mechanics.

### Step A3 — Tests

**New file `src/stores/__tests__/appearance.store.test.ts`** (complete content). The store applies state at import time, so persisted-value tests need `vi.resetModules()` + dynamic import:

```ts
// @vitest-environment happy-dom
import { describe, it, expect, beforeEach, vi } from "vitest";

beforeEach(() => {
  vi.resetModules();
  window.localStorage.clear();
  document.documentElement.style.fontSize = "";
});

describe("appearance.store", () => {
  it("defaults to normal with no inline font-size", async () => {
    const { useAppearanceStore } = await import("../appearance.store");
    expect(useAppearanceStore.getState().fontSize).toBe("normal");
    expect(document.documentElement.style.fontSize).toBe("");
  });

  it("setFontSize applies the css size and persists", async () => {
    const { useAppearanceStore } = await import("../appearance.store");
    useAppearanceStore.getState().setFontSize("xlarge");
    expect(document.documentElement.style.fontSize).toBe("125%");
    expect(window.localStorage.getItem("app-font-size")).toBe("xlarge");
  });

  it("re-applies a persisted size at module load", async () => {
    window.localStorage.setItem("app-font-size", "large");
    const { useAppearanceStore } = await import("../appearance.store");
    expect(useAppearanceStore.getState().fontSize).toBe("large");
    expect(document.documentElement.style.fontSize).toBe("112.5%");
  });

  it("falls back to normal on a garbage persisted value", async () => {
    window.localStorage.setItem("app-font-size", "gigantic");
    const { useAppearanceStore } = await import("../appearance.store");
    expect(useAppearanceStore.getState().fontSize).toBe("normal");
    expect(document.documentElement.style.fontSize).toBe("");
  });

  it("returning to normal clears the inline style", async () => {
    const { useAppearanceStore } = await import("../appearance.store");
    useAppearanceStore.getState().setFontSize("small");
    expect(document.documentElement.style.fontSize).toBe("87.5%");
    useAppearanceStore.getState().setFontSize("normal");
    expect(document.documentElement.style.fontSize).toBe("");
    expect(window.localStorage.getItem("app-font-size")).toBe("normal");
  });
});
```

**New file `src/pages/__tests__/Setting.test.tsx`.** Harness: copy the `createRoot`+`act` render helper from `Profile.test.tsx`, but wrap renders in `MemoryRouter` (the page uses `Link`). Mocks:

```tsx
// @vitest-environment happy-dom
import { MemoryRouter } from "react-router-dom";
// mutable auth state:
let authToken: string | null = null;
const mockLogout = vi.fn();
vi.mock("@/stores/auth.store", () => ({
  useAuthStore: (selector: (s: { token: string | null; logout: () => void }) => unknown) =>
    selector({ token: authToken, logout: mockLogout }),
}));
// mutable language (drives useAppI18n through the real lib):
let language: "en" | "th" = "en";
vi.mock("@/stores/language.store", () => ({
  useLanguageStore: (selector: (s: { language: "en" | "th"; setLanguage: (l: string) => void }) => unknown) =>
    selector({ language, setLanguage: vi.fn() }),
}));
vi.mock("@/stores/theme.store", () => ({
  useThemeStore: () => ({ theme: "light" }),
}));
vi.mock("@/components/ThemeToggle", () => ({
  ThemeToggle: () => <div data-testid="theme-toggle" />,
}));
vi.mock("@/components/editor/extensions/PersonalityPicker", () => ({
  default: () => <div data-testid="personality-picker" />,
}));
const mockNavigate = vi.fn();
vi.mock("react-router-dom", async () => {
  const actual = await vi.importActual("react-router-dom");
  return { ...actual, useNavigate: () => mockNavigate };
});
```

Do **not** mock `appearance.store` — use the real one (it has no network) and assert its observable effects. Reset in `beforeEach`: `language = "en"; authToken = null; mockNavigate.mockReset(); mockLogout.mockReset(); window.localStorage.clear(); document.documentElement.style.fontSize = "";` and in `afterEach` unmount + also remove any Radix portal leftovers (`document.body.innerHTML = ""` after unmount is acceptable here).

Test cases (each an `it`; render `<MemoryRouter><Settings /></MemoryRouter>`):

1. **Dead rows are gone:** anonymous render → `textContent` contains none of: `Notifications`, `Push & email alerts`, `Theme, font size, display`, `Privacy & Security`, `Delete account`.
2. **Anonymous sees only working rows (Golden Rule 2):** no `Log out`, no `Password` row; **has** `Help & Support`, `Font size`, the theme-toggle stub, the personality-picker stub, and the language buttons.
3. **Logged-in rows:** `authToken = "tok"` → has `Password` and `Log out`; clicking the Password row calls `mockNavigate` with `/change-password`.
4. **Font size applies + persists:** click the button labeled `Large` → `document.documentElement.style.fontSize === "112.5%"` and `localStorage.getItem("app-font-size") === "large"`; click `Normal` → inline style back to `""`.
5. **Help popup opens with the contact link:** click the `Help & Support` row → `document.body.textContent` contains `Message me on Facebook` and `built and run for free by one person`; the anchor's `href` equals `OWNER_FACEBOOK_URL` (import it from `@/lib/contact`); a link to `/status` exists. (Radix portals render into `document.body` — query there, not in your container.)
6. **Thai copy:** `language = "th"` → contains `ตั้งค่า`, `ขนาดตัวอักษร`, `ช่วยเหลือและสนับสนุน`.
7. **Logout flow:** `authToken = "tok"`, click `Log out` → `mockLogout` called and `mockNavigate` called with `/explore`, `{ replace: true }`.

`npm test` → green.

### Step A4 — Manual pass (server on :5000, client on :5173)

1. **Incognito `/settings`** (Golden Rule 2): every visible row does something — theme flips, language flips, personality selects, font-size buttons work, Help & Support opens the popup (Facebook link opens a new tab; status link navigates); **no** Password / Log out / dead rows.
2. **Logged in `/settings`:** Password row → `/change-password`; Log out works and lands on `/explore`.
3. **Font size end-to-end:** set ใหญ่มาก → reload → still ใหญ่มาก (persisted). Then, still at ใหญ่มาก, walk the test lesson `/view/69e39d0b60d467bd515a4945`: lesson text, question cards, and the tutor chat all scale; run one tutor exchange and confirm the **chat input row** and suggestion chips are usable.
4. **~390 px phone pass at ใหญ่มาก** (the acceptance's hard case): `/settings` itself, TopNav (content may truncate, must not wrap the bar), `/view/:id` question cards + chat input row — no horizontal scroll anywhere.
5. **Viewer zoom sanity:** in the lesson viewer, the existing zoom control still works independently of font size (two separate mechanisms — see §0).

### Step A5 — Docs + commit
1. Update `client/CLAUDE.md` (exact edits in §4).
2. `cd client && git add -A && git commit -m "feat(tier3.A): settings cleanup — cut dead rows, help popup, app-wide font size"`. **Do not push.**

**Phase 3.A acceptance (from ROADMAP):** every visible `/settings` row does something ✓ (tests 1–3, manual 1–2); anonymous users see no broken rows ✓ (test 2); font size persists across reload ✓ (test 4 + manual 3); largest size usable at 390 px ✓ (manual 4); tests cover the store + cut/wired rows ✓.

---

## 3. Phase 3.B — `/profile` completion (tutor-memory UI)

**Goal:** logged-in students can *see* what น้องมันฝรั่ง remembers about them and wipe it, from Profile, with confirm; plus a personality shortcut and one duplicate row removed. Size **S–M**, repo: `client/` only. Independent of 3.A (but commit 3.A first if you did both).

### Step B1 — New file `src/stores/tutorMemory.store.ts` (complete content)

```ts
import { create } from "zustand";
import api from "@/lib/axios";

export interface RecentTopic {
  content_id: string;
  summary: string;
  updatedAt: string;
}

export interface TutorMemory {
  interests: string[];
  strengths: string[];
  growth_areas: string[];
  preferences: string[];
  recent_topics: RecentTopic[];
}

const EMPTY_MEMORY: TutorMemory = {
  interests: [],
  strengths: [],
  growth_areas: [],
  preferences: [],
  recent_topics: [],
};

export const isMemoryEmpty = (memory: TutorMemory): boolean =>
  memory.interests.length === 0 &&
  memory.strengths.length === 0 &&
  memory.growth_areas.length === 0 &&
  memory.preferences.length === 0 &&
  memory.recent_topics.length === 0;

const toStringArray = (value: unknown): string[] =>
  Array.isArray(value)
    ? value.filter((item): item is string => typeof item === "string")
    : [];

// GET /chat/memory returns the raw StudentMemory doc (or {} when none exists).
// Normalize to the five public groups and DROP internal fields — _id, user_id,
// tutor_personality, timestamps must never reach the UI.
const normalizeMemory = (data: Record<string, unknown>): TutorMemory => ({
  interests: toStringArray(data.interests),
  strengths: toStringArray(data.strengths),
  growth_areas: toStringArray(data.growth_areas),
  preferences: toStringArray(data.preferences),
  recent_topics: Array.isArray(data.recent_topics)
    ? (data.recent_topics as RecentTopic[]).filter(
        (topic) =>
          typeof topic?.summary === "string" && topic.summary.trim() !== "",
      )
    : [],
});

interface TutorMemoryState {
  memory: TutorMemory | null; // null = not fetched yet
  isLoading: boolean;
  isClearing: boolean;
  error: "load_failed" | "clear_failed" | null;
  fetchMemory: () => Promise<void>;
  clearMemory: () => Promise<void>;
}

export const useTutorMemoryStore = create<TutorMemoryState>((set) => ({
  memory: null,
  isLoading: false,
  isClearing: false,
  error: null,

  fetchMemory: async () => {
    set({ isLoading: true, error: null });
    try {
      const res = await api.get<Record<string, unknown>>("/chat/memory");
      set({ memory: normalizeMemory(res.data ?? {}), isLoading: false });
    } catch {
      set({ error: "load_failed", isLoading: false });
    }
  },

  clearMemory: async () => {
    set({ isClearing: true, error: null });
    try {
      await api.delete("/chat/memory");
      set({ memory: { ...EMPTY_MEMORY }, isClearing: false });
    } catch {
      set({ error: "clear_failed", isClearing: false });
      throw new Error("clear_failed");
    }
  },
}));
```

(Error state is a **code**, not a display string — the card translates it at render, matching the bilingual convention.)

### Step B2 — New file `src/components/TutorMemoryCard.tsx` (complete content)

```tsx
import { useEffect, useState } from "react";
import { Brain, Loader2, Trash2 } from "lucide-react";
import { Button } from "@/components/ui/button";
import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
} from "@/components/ui/alert-dialog";
import { useTutorMemoryStore, isMemoryEmpty } from "@/stores/tutorMemory.store";
import { useAppI18n } from "@/lib/i18n";

function ChipGroup({ label, items }: { label: string; items: string[] }) {
  if (items.length === 0) return null;
  return (
    <div>
      <p className="mb-1.5 text-xs font-medium text-muted-foreground">{label}</p>
      <div className="flex flex-wrap gap-1.5">
        {items.map((item, i) => (
          <span
            key={`${item}-${i}`}
            className="rounded-full border border-border bg-muted/50 px-2.5 py-1 text-xs text-foreground"
          >
            {item}
          </span>
        ))}
      </div>
    </div>
  );
}

export function TutorMemoryCard() {
  const { t } = useAppI18n();
  const { memory, isLoading, isClearing, error, fetchMemory, clearMemory } =
    useTutorMemoryStore();
  const [confirmOpen, setConfirmOpen] = useState(false);
  const [clearedFlash, setClearedFlash] = useState(false);

  useEffect(() => {
    void fetchMemory();
  }, [fetchMemory]);

  const handleClear = async () => {
    try {
      await clearMemory();
      setClearedFlash(true);
      setTimeout(() => setClearedFlash(false), 2500);
    } catch {
      // clear_failed rendered from the store state below
    }
  };

  const empty = memory !== null && isMemoryEmpty(memory);

  return (
    <div className="rounded-lg border border-border bg-card px-4 py-4">
      <div className="flex items-start justify-between gap-3">
        <div className="flex items-center gap-2">
          <Brain className="h-4 w-4 shrink-0 text-muted-foreground" />
          <p className="text-sm font-medium text-foreground">
            {t("What your tutor remembers", "ความจำของติวเตอร์")}
          </p>
        </div>
        {memory !== null && !empty && (
          <Button
            variant="ghost"
            size="sm"
            className="gap-1.5 text-destructive hover:text-destructive"
            disabled={isClearing}
            onClick={() => setConfirmOpen(true)}
          >
            {isClearing ? (
              <Loader2 className="h-3.5 w-3.5 animate-spin" />
            ) : (
              <Trash2 className="h-3.5 w-3.5" />
            )}
            {t("Forget", "ลบความจำ")}
          </Button>
        )}
      </div>

      <div className="mt-3">
        {isLoading && memory === null ? (
          <div className="flex justify-center py-6">
            <Loader2 className="h-5 w-5 animate-spin text-muted-foreground" />
          </div>
        ) : error === "load_failed" ? (
          <div className="py-2 text-center">
            <p className="text-sm text-destructive" role="alert">
              {t("Couldn't load tutor memory.", "โหลดความจำของติวเตอร์ไม่สำเร็จ")}
            </p>
            <Button
              variant="outline"
              size="sm"
              className="mt-2"
              onClick={() => void fetchMemory()}
            >
              {t("Retry", "ลองอีกครั้ง")}
            </Button>
          </div>
        ) : empty ? (
          <p className="py-4 text-center text-sm text-muted-foreground">
            {clearedFlash
              ? t("All forgotten 🥔", "ลบความจำแล้ว 🥔")
              : t(
                  "Chat with your tutor first — they'll start remembering you. 🥔",
                  "คุยกับติวเตอร์ไปก่อน เดี๋ยวเขาจะจำเธอได้เอง 🥔",
                )}
          </p>
        ) : memory ? (
          <div className="space-y-3">
            <ChipGroup
              label={t("Interests", "สิ่งที่สนใจ")}
              items={memory.interests}
            />
            <ChipGroup
              label={t("Strengths", "จุดแข็ง")}
              items={memory.strengths}
            />
            <ChipGroup
              label={t("Growing in", "กำลังพัฒนา")}
              items={memory.growth_areas}
            />
            <ChipGroup
              label={t("Preferences", "สไตล์ที่ชอบ")}
              items={memory.preferences}
            />
            <ChipGroup
              label={t("Recent topics", "เรื่องที่คุยกันล่าสุด")}
              items={memory.recent_topics.map((topic) => topic.summary)}
            />
            {error === "clear_failed" && (
              <p className="text-xs text-destructive" role="alert">
                {t("Couldn't clear memory. Try again.", "ลบความจำไม่สำเร็จ ลองอีกครั้งนะ")}
              </p>
            )}
          </div>
        ) : null}
      </div>

      <AlertDialog open={confirmOpen} onOpenChange={setConfirmOpen}>
        <AlertDialogContent>
          <AlertDialogHeader>
            <AlertDialogTitle>
              {t("Clear tutor memory?", "ลบความจำของติวเตอร์?")}
            </AlertDialogTitle>
            <AlertDialogDescription>
              {t(
                "น้องมันฝรั่ง will forget everything it remembers about you — interests, strengths, recent topics. This can't be undone.",
                "น้องมันฝรั่งจะลืมทุกอย่างที่จำเกี่ยวกับเธอไว้ ทั้งสิ่งที่สนใจ จุดแข็ง และเรื่องที่เคยคุยกัน ลบแล้วกู้คืนไม่ได้นะ",
              )}
            </AlertDialogDescription>
          </AlertDialogHeader>
          <AlertDialogFooter>
            <AlertDialogCancel>{t("Keep it", "เก็บไว้ก่อน")}</AlertDialogCancel>
            <AlertDialogAction
              className="bg-destructive/10 text-destructive hover:bg-destructive/20"
              onClick={() => void handleClear()}
            >
              {t("Clear memory", "ลบเลย")}
            </AlertDialogAction>
          </AlertDialogFooter>
        </AlertDialogContent>
      </AlertDialog>
    </div>
  );
}
```

Copy notes (product principle 2 — dopamine, never negative): `growth_areas` is labeled **"กำลังพัฒนา / Growing in"**, never "weaknesses/จุดอ่อน". The `tutor_personality` field must never render here (the store already strips it — that's what the normalize test guards).

### Step B3 — Edit `src/pages/Profile.tsx` (surgical — read the file first; four edits)

1. **Remove the Theme row** — the first block inside the form `div` (`{t("Theme", "ธีม")}` … `<ThemeToggle />`, currently lines ~197–209). Then remove the now-unused `import { ThemeToggle } …`, `import { useThemeStore } …`, and `const { theme } = useThemeStore();`.
2. **Extend the imports:**
   ```tsx
   import { Link, useNavigate } from "react-router-dom"; // Link added
   import { TutorMemoryCard } from "@/components/TutorMemoryCard";
   import {
     TUTOR_PERSONALITY_CATALOG,
     useTutorPersonalityStore,
   } from "@/stores/tutorPersonality.store";
   ```
   and change `const { t } = useAppI18n();` → `const { isThai, t } = useAppI18n();`.
3. **Resolve the current personality** (next to the other hooks at the top):
   ```tsx
   const personalityId = useTutorPersonalityStore((s) => s.personality);
   const currentPersonality =
     TUTOR_PERSONALITY_CATALOG.find((p) => p.id === personalityId) ??
     TUTOR_PERSONALITY_CATALOG[0];
   ```
4. **Add the tutor section** at the end of the page, after the form's closing `</div>` (i.e. after the save-button/error block, still inside the `container` div):
   ```tsx
   {/* Tutor section (Tier 3.B) */}
   <div className="mt-10 space-y-4">
     <h2 className="text-xs font-semibold uppercase tracking-wider text-muted-foreground">
       {t("Your AI tutor", "ติวเตอร์ของคุณ")}
     </h2>

     <div className="flex items-center justify-between rounded-lg border border-border bg-card px-4 py-3">
       <div>
         <p className="text-sm font-medium text-foreground">
           {t("Tutor personality", "บุคลิกติวเตอร์")}
         </p>
         <p className="text-xs text-muted-foreground">
           {currentPersonality.emoji}{" "}
           {isThai ? currentPersonality.labelTh : currentPersonality.labelEn}
         </p>
       </div>
       <Link
         to="/settings"
         className="text-sm font-medium text-primary hover:underline"
       >
         {t("Change", "เปลี่ยน")}
       </Link>
     </div>

     <TutorMemoryCard />
   </div>
   ```
   (A shortcut showing the current preset with a link — the picker itself is NOT duplicated here; it lives in Settings only.)

**OPTIONAL — light learning stats (roadmap marks it optional; skip if anything above ran long).** One more row in the tutor section, above `<TutorMemoryCard />`: lessons-visited count from the existing history store. Import `useLearningHistoryStore` from `@/stores/learningHistory.store`, then `const entries = useLearningHistoryStore((s) => s.entries); const fetchHistory = useLearningHistoryStore((s) => s.fetchHistory);` + `useEffect(() => { void fetchHistory(50); }, [fetchHistory]);`, rendering `{t("Lessons visited", "บทเรียนที่เคยเปิด")}` with `{entries.length >= 50 ? "50+" : entries.length}`. **No scores, no comparisons — ever** (product principle 2). If you take it, also mock `@/stores/learningHistory.store` in `Profile.test.tsx` and note it in §7.

### Step B4 — Tests

**New file `src/stores/__tests__/tutorMemory.store.test.ts`.** Mock axios (`vi.mock("@/lib/axios", () => ({ default: { get: mockGet, delete: mockDelete } }))` with hoisted `vi.fn()`s). The store is module-level zustand — reset state in `beforeEach` with `useTutorMemoryStore.setState({ memory: null, isLoading: false, isClearing: false, error: null })`. Cases:

1. **Normalizes the raw doc and strips internal fields:** `mockGet` resolves with a realistic doc — the five public groups populated **plus** `_id`, `user_id: "u1"`, `tutor_personality: "gentle"`, `createdAt`, `updatedAt` — after `fetchMemory()`, `memory` contains the groups and `JSON.stringify(memory)` contains neither `"gentle"` nor `"user_id"`.
2. **Empty server response (`{}`):** `memory` equals the all-empty shape and `isMemoryEmpty(memory)` is `true`.
3. **Recent topics with blank summaries are filtered** (send one real summary + one `""`).
4. **Fetch failure:** `mockGet` rejects → `error === "load_failed"`, `memory` stays `null`.
5. **clearMemory:** calls `DELETE` on `/chat/memory`, resets `memory` to empty; **failure case:** rejects → `error === "clear_failed"` and the promise rejects (assert with `await expect(...).rejects.toThrow()`), previous `memory` untouched.

**New file `src/components/__tests__/TutorMemoryCard.test.tsx`.** Same harness as `Profile.test.tsx` (no router needed — the card has no links). Mock **only** the store, with a hoisted mutable `memoryState` (pattern from `Profile.test.tsx`); re-implement `isMemoryEmpty` inside the mock factory (it's four lines — the component imports it from the same module you're mocking):

```tsx
vi.mock("@/stores/tutorMemory.store", () => ({
  isMemoryEmpty: (m: {
    interests: string[]; strengths: string[]; growth_areas: string[];
    preferences: string[]; recent_topics: unknown[];
  }) =>
    m.interests.length === 0 && m.strengths.length === 0 &&
    m.growth_areas.length === 0 && m.preferences.length === 0 &&
    m.recent_topics.length === 0,
  useTutorMemoryStore: (selector?: (s: typeof memoryState) => unknown) =>
    selector ? selector(memoryState) : memoryState,
}));
```

Also mock `@/stores/language.store` (mutable `language`, as in `Profile.test.tsx`). Reset `memoryState` fields + `document.body.innerHTML` cleanup in `beforeEach`/`afterEach`. Cases:

1. **Fetches on mount:** rendering calls `mockFetchMemory` once.
2. **Empty state (th):** `memory` = all-empty, `language = "th"` → contains `คุยกับติวเตอร์ไปก่อน เดี๋ยวเขาจะจำเธอได้เอง 🥔`; no "Forget" button.
3. **Loaded chips:** memory with items in all five groups (recent_topics as `{ content_id, summary, updatedAt }`) → each item's text renders; the `Growing in` label renders; "Forget" button present.
4. **Delete flow:** loaded memory → click the Forget button → confirm dialog appears (**assert on `document.body.textContent`** — Radix portal) with `ลบความจำของติวเตอร์?`/`Clear tutor memory?` → click the action button (`Clear memory`) → `mockClearMemory` called once.
5. **load_failed:** shows the error text and a Retry button; clicking Retry calls `mockFetchMemory` again.
6. **clear_failed:** loaded memory + `error: "clear_failed"` → the inline error text renders.

**Extend `src/pages/__tests__/Profile.test.tsx`** (keep every existing test passing):
- Profile now renders `Link` → **wrap the existing renders in `MemoryRouter`** (adjust the `render(<Profile />)` calls to `render(<MemoryRouter><Profile /></MemoryRouter>)`; the existing `useNavigate` mock via `importActual` already keeps `MemoryRouter` real).
- Add mocks: stub the card — `vi.mock("@/components/TutorMemoryCard", () => ({ TutorMemoryCard: () => <div data-testid="tutor-memory-card" /> }))` — and the personality store with an inline two-preset catalog:
  ```tsx
  vi.mock("@/stores/tutorPersonality.store", () => ({
    TUTOR_PERSONALITY_CATALOG: [
      { id: "default", labelTh: "น้องมันฝรั่งคลาสสิก", labelEn: "Classic", emoji: "🥔" },
      { id: "gentle", labelTh: "สุภาพ อ่อนโยน", labelEn: "Gentle", emoji: "🌷" },
    ],
    useTutorPersonalityStore: (selector: (s: { personality: string }) => unknown) =>
      selector({ personality: "gentle" }),
  }));
  ```
- New tests: **(a)** the Theme row is gone — `textContent` contains neither `Theme` nor `ธีม`, and no `theme-toggle` testid; **(b)** the personality shortcut shows `Gentle` (en) with the 🌷 emoji and a link whose `href` ends with `/settings`; **(c)** the memory-card stub renders.

`npm test` → all green.

### Step B5 — Manual pass

1. **Logged in**, `/profile`: if the account has chatted before, the memory card shows chips; otherwise chat with the tutor on the test lesson for 2–3 real exchanges (memory digest is an async background write — give it a moment), then refresh Profile. Delete flow: ลบความจำ → confirm → Network shows `DELETE /api/chat/memory` 200 → card flips to the empty state with the "ลบความจำแล้ว 🥔" flash → refresh → still empty (server-verified).
2. **Personality survives the wipe** (§1.4 interaction): after clearing memory, the personality shortcut still shows your preset; one more tutor exchange re-syncs it server-side. Nothing to fix — just confirm no visible breakage.
3. **Incognito `/profile`:** the `RequireLogin` sign-in prompt renders; Network shows **no** call to `/chat/memory` (the card never mounted).
4. **Both languages** on the tutor section; **390 px pass**: chips wrap, no horizontal scroll, confirm dialog fits (`max-w-sm` — fine at 390 px, eyeball it).

### Step B6 — Docs + commit
1. Update `client/CLAUDE.md` (§4).
2. `cd client && git add -A && git commit -m "feat(tier3.B): profile completion — tutor memory card + personality shortcut"`. **Do not push.**

**Phase 3.B acceptance (from ROADMAP):** logged-in students view + wipe tutor memory from Profile with confirm ✓ (card tests 3–4, manual 1); anonymous users never see the memory card ✓ (route guard + manual 3); tests cover loaded / empty / delete states ✓.

---

## 4. Documentation updates (required — part of done)

1. **`client/CLAUDE.md`:**
   - Stores table, two new rows:
     - `appearance.store` — App-wide font size (`small…xlarge`; persisted as `app-font-size`); applies `%` to `document.documentElement.style.fontSize` at module load — imported in `main.tsx` so it runs on every route. Separate mechanism from the viewer's CSS `zoom`; never merge them.
     - `tutorMemory.store` — Student's tutor memory (`GET/DELETE /chat/memory`); normalizes the raw StudentMemory doc and strips internal fields (`tutor_personality`, ids, timestamps).
   - Routing table: `/settings` note → "Settings (public; cleaned in Tier 3.A — theme, language, font size, personality, help popup; account rows only when logged in)"; `/profile` note → append "+ tutor-memory card & personality shortcut (Tier 3.B)".
   - Gotchas, one bullet: "**Font size vs viewer zoom.** The app-wide font-size control (`appearance.store`, inline `%` on `<html>`) and the lesson viewer's CSS `zoom` are deliberately separate mechanisms — don't merge them, and never use `transform: scale` (double-scrollbar bug). `--app-nav-height` stays px on purpose."
2. **`../ROADMAP.md`** (workspace root, not in either git repo):
   - Under the Tier 3 heading add: `> ✅ Shipped <date> — 3.A + 3.B (client commits <hashes>).`
   - In the Tier 5 ordered checklist, append a new item: `7. **Real Facebook contact URL:** replace the placeholder in client/src/lib/contact.ts (OWNER_FACEBOOK_URL) with the owner's page and re-run the settings help-popup smoke.`
3. If you deviated anywhere, record it in §7 of this file.

---

## 5. Final done-definition checklist

- [ ] `cd client && npm test` green — baseline (recorded in §7) + `appearance.store.test.ts` + `Setting.test.tsx` + `tutorMemory.store.test.ts` + `TutorMemoryCard.test.tsx` + extended `Profile.test.tsx`, none skipped.
- [ ] `cd client && npm run build` green; `npm run lint` introduces **no new** errors.
- [ ] `/settings` incognito: zero dead rows; theme/language/font-size/personality/help-popup all work logged-out (Golden Rule 2).
- [ ] Font size: persists across reload; ใหญ่มาก at 390 px is usable on `/settings`, TopNav, and the test lesson's question cards + chat input row.
- [ ] `/profile` logged-in: memory chips render; delete = confirm → `DELETE /api/chat/memory` → empty state; personality shortcut links to `/settings`.
- [ ] `/profile` incognito: sign-in prompt, no `/chat/memory` request.
- [ ] Theme row exists in exactly one place (Settings); Delete-account button exists nowhere (until Tier 6).
- [ ] `client/CLAUDE.md` + `../ROADMAP.md` updated; §7 below filled in.
- [ ] Two commits in `client/`, **nothing pushed**; zero changes in `server/`.
- [ ] Untouched: viewer zoom, `tutorPersonality.store`, `--app-nav-height`, prompts, rate limits, route guards.

## 6. Failure playbook

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Dialog content not found in a test | You queried your render container | Radix portals render into `document.body` — assert on `document.body.textContent` / query `document.body` |
| Radix `AlertDialog` won't open/close via `.click()` in happy-dom | happy-dom pointer-event gaps | The dialogs are **controlled** — click the plain page button that sets state (already the design). If `AlertDialogAction`'s click misbehaves, dispatch `new MouseEvent("click", { bubbles: true })`; last resort: `vi.mock("@/components/ui/alert-dialog")` with pass-through divs and note it in §7 |
| `Link`/`useNavigate` throws "outside a `<Router>`" in tests | Bare render of a page that now uses `Link` | Wrap the render in `MemoryRouter` (§3 Step B4 first bullet) |
| Existing `Profile.test.tsx` tests break after B3 | New store imports unmocked | Add the `TutorMemoryCard` stub + `tutorPersonality.store` mock exactly as in Step B4 |
| Lint error `react-refresh/only-export-components` on `Setting.tsx` | You exported a constant from the page | The Facebook URL lives in `src/lib/contact.ts` — import it, never export non-components from pages |
| Font size "doesn't work" on a fresh route in dev | `appearance.store` never imported on that route's graph | Confirm the `import "./stores/appearance.store";` line in `main.tsx` (Step A1) |
| TopNav content cramped at ใหญ่มาก | Px nav height + larger rem text | Expected within reason — let text truncate; do NOT grow `--app-nav-height` (editor grid depends on it) |
| Personality picker "reset" after clearing memory | It didn't — server-side sync field was wiped with the doc | Expected (§1.4); client keeps the local choice and re-syncs on the next tutor call. Do nothing |
| `store.test.ts` state bleeding between cases | Module-level zustand store | `useTutorMemoryStore.setState({ … })` reset in `beforeEach` (Step B4); for `appearance.store` use `vi.resetModules()` + dynamic import (Step A3) |

## 7. Phase notes (implementing agent fills this in)

### Baseline (preflight A0)
_86 tests green; build green. Tiers 0, 1, 2.A, 2.B already landed before this session._

### After 3.A
_98 tests (+5 appearance store, +7 Setting page). Build green._

### After 3.B
_113 tests (+6 tutorMemory store, +6 TutorMemoryCard, +3 Profile extensions). Optional learning-stats row skipped. Build green._

### Deviations from this plan
_tutorMemory.store.test.ts: preferences mock uses `"warm tone"` instead of `"gentle tone"` because the spec's `JSON.stringify` guard falsely matched the substring `"gentle"` inside a legitimate preference chip._
