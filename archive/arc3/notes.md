# notes.md — forward-looking notes from the full audit pass (2026-07-11)

Aggregated from the 11-agent maintenance audit across `client/` and `server/`.
Routine fixes already applied are **not** listed here — only things to build,
delete, or refactor later. Both test suites are green (client 183, server 272),
lint is clean, `tsc` passes, and the production build succeeds.

---

## 1. Suggested new features (noticed gaps — NOT built)

### Client

- **Error toast on lesson-create failure** — `Create.tsx` and `Dashboard.tsx` now recover from a failed `POST /content/create`, but the user gets no visible message.
- **Per-lesson `document.title` on `/view/:id`** — every lesson tab is currently titled the same; cheap SEO/UX win (real per-lesson OG tags stay parked as ROADMAP Tier 6.D).
- **i18n gaps** — `ChangePassword.tsx`, `NotFound.tsx`, `Landing.tsx`, and the mobile editor gate in `TipTapCanvas.tsx` are English-only while sibling pages use `useAppI18n`; `AiErrorRetry` button text is hardcoded Thai.
- **Surface missing Cloudinary config** — `cloudinary.ts` exports `isConfigured()` but the upload UI never uses it; missing env vars today fail with generic errors.
- **ImagePanel link field** — doesn't initialize from an image's existing `data-href` when reopening.
- **Persist tutor personality server-side on change** — logged-in users' preset only syncs _from_ the server when local is `"default"`; changing it never writes back.
- **Landing showcase cards are static placeholders** — could pull live public lessons from the API.

### Server

- **Abort Gemini stream on client disconnect** — when an SSE client drops, the server still reads the full model output; wire `req.on("close")` to abort consumption (token cost, not correctness).
- **Suggestions gate final flush** — `createSuggestionsGate` has no `flush()` at stream end; an inline `[SUGGESTIONS]` marker without a preceding newline would leak into the reply (persona forbids it, but defense-in-depth).

### Tests worth adding

- Client: unit tests for `callTutorStream` (SSE parsing, mid-stream failure, JSON fallback) and for `caretInsertPoint`'s empty-paragraph replace path.
- Server: API tests for `/api/images` and `/api/categories` (none exist); `test/api/history.test.ts` covering the `recordVisit` access matrix (public / link-only / private); integration test for case-insensitive login; a `lesson_meta` model-routing assertion in `creator.test.ts`.

---

## 2. Recommended for deletion

- **`client/src/components/editor/FormulaBlock/FormulaNode.tsx` + `Sidebar.tsx`** — legacy visual formula builder, not imported anywhere; the block is LaTeX-first via `FormulaCanvas`. Related: `formulaReducer.ts` / `formulaToLatex.ts` are mostly legacy — only `createFormulaRow()` is still called (and it mints a new UUID per keystroke); consider dropping the tree attr entirely.
- **`auth.store` `error` field** — never set or read (`Login.tsx` uses local `serverError`); remove or wire up.
- **`learningHistory.store` / `category.store` `clearError`** — no consumers (only `cloudinary.store`'s is used).
- **`FAST_MODEL` / `TUTOR_MODEL` deprecated constants in `server/src/controllers/tutor.controller.ts`** — no in-repo importers; the `getFastModel`/`getTutorModel` re-exports could move fully to `services/tutor/models.ts`.
- **`explore=true` query param on `GET /content/search`** — now redundant (non-`mine` searches always filter public); keep for client compat, deprecate later.
- Already removed during this pass (for the record): the four dead `components/design/*` files + empty folder, the ~700-line duplicate `pages/Cloudinaryupload.tsx` body (now a thin wrapper over `components/CloudinaryUpload.tsx`).

---

## 3. Recommended to refactor later

### Client — question blocks (highest-value refactor)

- **Extract shared viewer feedback/thread boilerplate** across the five `Question*View` components (persist answer → `callTutorStream` → `FeedbackDiscussionPanel` → cold-start hints). High duplication, but risky to merge without careful regression testing — do it as its own phase with tests.
- **Shared `blankTemplateUtils.ts`** for `BLANK_TOKEN_REGEX` / `getBlankIndices` / `renderTemplatePieces` (duplicated in `QuestionBlankChoiceView` and `QuestionBlankWriteView`).
- **Consolidate the duplicate `*Attrs` interfaces** defined in both `*Node.ts` and `*View.tsx` into a shared `questionTypes.ts` (circular-import constraints today).
- **`content-answer.store`: scope answers to a `contentId` / add `clearAnswers()`** — anonymous question-block answers can still leak across lessons via the global store (the Ask-AI modal was fixed locally; the question views remain exposed).

### Client — editor & canvas

- Extract the duplicated `COLORS` / `HIGHLIGHTS` / text-structure panel logic shared by `EditorLeftSidebar` and `EditorRightSidebar`.
- `useCanvasDrag` polls every 500 ms to wire new canvases — hook into `CanvasContext.registerCanvas` instead.
- Split `CanvasRightSidebar.tsx` (~1500 lines) and the `RichLine` class inside `useFabric.ts` into smaller modules; harden `useFabric` GIF/video RAF loop cleanup on canvas `dispose()`.
- Unify drag-drop image insert (data-URL) with paste (Cloudinary upload) in `TipTapEditor` if teachers drag images often.
- DRY the zoom/layout constants shared between `TiptapViewer` and `TipTapEditor`.
- Give embedded `CloudinaryUpload` a `compact` layout variant (editor sidebars currently inherit full-page `min-h-screen` sizing).
- AI dialogs: `AiCriticDialog` / `AiQuestionDialog` / `WritingPreviewDialog` render inline — if `.editor-main` focus-steal appears there, adopt `AiDraftDialog`'s `document.body` portal + `data-editor-modal` pattern; a shared modal shell would cut duplicated markup. `AiDraftDialog` fill-tab headings memo refreshes on tab switch, not live doc edits.

### Client — misc

- `content.store` shares one `error` field between dashboard and explore flows — a failed explore search can surface on the dashboard.
- Explore's **Recent** tab sorts by lesson `updatedAt`, not the user's visit time — mislabeled; use `learningHistory` data or rename.
- Normalize import paths (`../lib/axios` vs `@/lib/axios`) and the `Create.tsx` default export name (`CreatorDashboard`).
- `Login.tsx` still manually resets `isLoading` via `setState` — redundant now that `auth.store` handles it.
- `Setting.tsx` inlines EN/TH buttons though `LanguageToggle.tsx` exists; Dashboard logout → `/login` vs Settings logout → `/explore` — align if unintentional.
- `SearchHighlight.ts` uses `any` for ProseMirror `doc`/`node` — type with `@tiptap/pm`.

### Server

- **Extract shared `isValidContentId` + `checkContentAccess`** — the same inline access logic lives in `content.controller.ts`, `history.controller.ts`, and `tutor.controller.ts`.
- **Escape/limit `q` in `searchContent`'s `$regex`** — user input flows into a regex (ReDoS / injection risk).
- Unify `sanitizeAgentSettings` (`content.controller.ts`) and `normalizeAgentSettings` (`lessonContext.service.ts`) into one normalizer.
- `refreshAuthorSnapshotsForUser` does sequential `updateOne` per lesson — switch to `bulkWrite` for renames at scale.
- Tutor: extract shared `contents` assembly from `callTutorModel` / `callTutorModelStream`; move `resolveTutorModel` into `services/tutor/models.ts`; consider atomic `findOneAndUpdate` + `$push` for session writes (must replicate the 50-message cap) and a versioned merge for concurrent `runMemoryUpdate`; clamp `AI_MEMORY_EVERY_N_TURNS` to a positive integer.
- `rateLimit.middleware.ts`: `Number(env) || default` treats `"0"` as unset — parse explicitly if zero should mean "disabled". In-memory buckets don't share state across instances (only matters if Render ever runs >1 instance — Redis then).
- Login returns unstructured **403** for blocked/suspended accounts while `protect` returns structured **401 ACCOUNT_INACTIVE** — intentional today, but consider unifying the shape.
- `connectDB` calls `process.exit(1)` on retry exhaustion, so `index.ts`'s `.catch()` is unreachable — pick one failure path.
- Consolidate duplicate `dotenv.config()` calls (`app.ts` + `index.ts`).
- Creator: `parseJsonLoose` doesn't handle top-level JSON arrays (fine today; matters if a future action returns one); a shared `resolveOwnedCategoryId`-style helper if more image-library endpoints appear; `saveImage` duplicate check could return the existing doc for same-owner duplicates (API behavior change — deliberate decision needed).
