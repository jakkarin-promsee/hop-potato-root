# Graph Report - C:\Users\BTCOM\Desktop\0_Project\Hot-Potato  (2026-07-11)

## Corpus Check
- 90 files · ~356,617 words
- Verdict: corpus is large enough that graph structure adds value.

## Summary
- 1905 nodes · 2576 edges · 210 communities (126 shown, 84 thin omitted)
- Extraction: 98% EXTRACTED · 2% INFERRED · 0% AMBIGUOUS · INFERRED: 52 edges (avg confidence: 0.84)
- Token cost: 0 input · 0 output

## Community Hubs (Navigation)
- auth.middleware.ts
- tutor.controller.ts
- formulaReducer.ts
- cn()
- EditorLeftSidebar.tsx
- LanguageToggle.tsx
- seed-guide-demo.mjs
- devDependencies
- EditorRightSidebar.tsx
- axios.ts
- App.tsx
- RichLine
- Lesson Editor Page
- CanvasRightSidebar.tsx
- Status.tsx
- AiDraftDialog.tsx
- persona.ts
- QuestionFeedbackMode
- ShowcaseShell.tsx
- prompts.ts
- tutorMemory.store.ts
- devDependencies
- components.json
- auth.controller.ts
- validate.ts
- dependencies
- compilerOptions
- observability.service.ts
- Phase 3.B — Profile completion + tutor memory card
- lessonContext.service.ts
- useAuthStore
- tutorApi.ts
- content.model.ts
- AiWritingAssistant.tsx
- creatorApi.ts
- TopNav.tsx
- package.json
- TipTap Lesson Editor
- TiptapViewer.tsx
- app.ts
- compilerOptions
- History.tsx
- QuestionWriteView.tsx
- Creator 11 Actions
- ROADMAP.md Launch-Readiness v3
- creator.controller.ts
- BlockDeleteButton.tsx
- AiQuestionDialog.tsx
- EditorHeader.tsx
- QuestionBlankWriteView.tsx
- creator.test.ts
- Mobile lesson viewer page (TipTapViewer)
- archive/arc2/ROADMAP.md Universal
- AiToolsPanel.tsx
- writingAssist.ts
- FabricCanvasView.tsx
- QuestionChoiceView.tsx
- ResizableImage.ts
- CLAUDE.md (root)
- asking-flow.md
- numberedSectionHeadings.ts
- QuestionAgentNode.ts
- QuestionBlankChoiceView.tsx
- POST /api/creator/assist
- 🥔 Hot Potato
- scripts
- Ask AI modal dialog (centered overlay on dimmed lesson page)
- Lesson page initial state before answer submission
- dependencies
- MarkdownMessage.tsx
- PublishAutofill.test.tsx
- guards.test.tsx
- Write question: Can a red sedan parked roadside be a referen
- index.html — Site-Wide Thai SEO/OG/Twitter Meta Tags
- Hot Potato brand mascot (og-image)
- AiToolsPanel.test.tsx
- AiFormulaPanel.test.tsx
- Login.test.tsx
- theme.store.ts
- POST /api/chat/tutor
- tutor.test.ts
- archive/arc1/ROADMAP-detailed.md
- Reference point (จุดอ้างอิง) — fixed origin for measuring di
- Suggested follow-up question chips (purple outline pills)
- AiCriticButton.test.tsx
- AiQuestionDialog.test.tsx
- AiWritingAssistant.test.tsx
- questionEvaluation.ts
- PublishSettingsModal.tsx
- learning_history.model.ts
- parse.ts
- Inline contextual chat panel embedded in lesson scroll
- SearchHighlight.ts
- clipboardPasteImageCache.ts
- vite-env.d.ts
- observability.service.ts — In-Memory Error Ring Buffer + AI 
- chat_session.model.ts
- personality.ts
- bookmark.store.ts
- seo.test.ts
- playwright
- category.model.ts
- user_content.model.ts
- user_data.model.ts
- @tiptap/extension-table-cell
- AiErrorRetry.tsx
- History.test.tsx
- Tier 2 Execution Plan — Bundle Split + Self-Hosted Fonts
- check-models.ts
- promote-admin.ts
- retry.ts
- auth.test.ts
- class-variance-authority
- axios
- clsx
- fabric
- @fontsource/inter
- @fontsource/jetbrains-mono
- @fontsource/lora
- @fontsource-variable/geist
- katex
- lucide-react
- qrcode.react
- radix-ui
- react
- react-dom
- react-markdown
- @react-oauth/google
- react-router-dom
- remark-gfm
- shadcn
- sonner
- tailwind-merge
- tailwindcss
- @tailwindcss/vite
- @tanstack/react-query
- @tiptap/extension-highlight
- @tiptap/extension-placeholder
- @tiptap/extension-task-item
- @tiptap/extension-task-list
- @tiptap/extension-text-style
- @tiptap/extension-youtube
- tiptap-markdown
- @tiptap/pm
- @tiptap/react
- @tiptap/starter-kit
- tw-animate-css
- uuid
- zustand
- saveState Fabric-TipTap Sync Bridge
- vercel.json
- caretInsertPoint
- Tier Decision T2 — Preview Accept
- App Loading State Snapshot
- Phase 0.A — Static Meta + OG Tags
- ts-node
- CanvasContext
- Entry Chunk Target ≤323 kB Gzip for / and /explore
- no-cdn-fonts.test.ts — CDN Font Regression Guard
- Tier Decision T3 — No Raw TipTap JSON
- Golden Rule 1 — Never ration tokens
- App Loading State Snapshot (Duplicate)
- Ask AI Button
- Lesson Viewer with Question Blocks
- Cloudinary Image Media
- Google Gemini AI Tutor
- MongoDB Atlas Persistence
- Thai Default AI Language
- Phase 2.A — Route-Level Code Splitting
- Phase 2.B — Self-Host Fonts
- Phase 3.A — Settings Cleanup + Appearance
- Tier 1 — Observability
- Tier 2 — Performance Bundle Split
- Tier 3 — Settings & Profile
- Tier 4 — CI GitHub Actions
- Tier 6 — Deferred Backlog
- server/README.md
- Content Entity (schema sketch)
- User_content Entity (schema sketch)
- User Entity (schema sketch)
- Tier 3 planning session prompt
- AI content critic for lessons
- AI-assisted question creation
- Owner teacher AI helper feature request
- Tier 3 implementation session prompt

## God Nodes (most connected - your core abstractions)
1. `cn()` - 37 edges
2. `RichLine` - 19 edges
3. `compilerOptions` - 18 edges
4. `useAuthStore` - 17 edges
5. `tutorChat()` - 14 edges
6. `callCreator()` - 12 edges
7. `POST /api/chat/tutor` - 12 edges
8. `AiDraftDialog()` - 12 edges
9. `system()` - 12 edges
10. `QuestionFeedbackMode` - 11 edges

## Surprising Connections (you probably didn't know these)
- `POST /api/chat/tutor (Arc1 Phase 1)` --semantically_similar_to--> `POST /api/chat/tutor`  [INFERRED] [semantically similar]
  archive/arc1/ROADMAP-detailed.md → server/CLAUDE.md
- `Question View Feedback Refactor` --semantically_similar_to--> `TipTap Lesson Editor`  [INFERRED] [semantically similar]
  notes.md → client/CLAUDE.md
- `Half-AI formula block for low-tech teachers` --semantically_similar_to--> `FormulaBlock latex attr`  [INFERRED] [semantically similar]
  temp/prompt/next.md → plan/tier3.5.md
- `Teacher AI Copilot` --references--> `callCreator()`  [INFERRED]
  client/CLAUDE.md → client/src/lib/creatorApi.ts
- `content-answer.store ContentId Scoping` --semantically_similar_to--> `tutorApi Single Bridge`  [INFERRED] [semantically similar]
  notes.md → client/CLAUDE.md

## Import Cycles
- None detected.

## Hyperedges (group relationships)
- **Critical Thinking Question System** — client_claude_tiptap_editor, client_claude_tutor_api_bridge, client_claude_tiptap_node_truth [EXTRACTED 1.00]
- **Teacher AI Lesson Authoring Flow** — _playwright_mcp_page_2026_07_11t03_32_19_442z_lesson_editor_page, _playwright_mcp_page_2026_07_11t03_32_19_442z_ai_start_lesson_modal, _playwright_mcp_page_2026_07_11t03_32_19_442z_ai_outline_tab, _playwright_mcp_page_2026_07_11t03_33_23_897z_friction_lesson_outline, _playwright_mcp_page_2026_07_11t03_34_04_475z_ai_fill_content_tab, _playwright_mcp_page_2026_07_11t03_36_06_066z_ai_generate_questions_panel, _playwright_mcp_page_2026_07_11t03_37_11_699z_formula_sidebar, _playwright_mcp_page_2026_07_11t03_38_27_246z_formula_block, _playwright_mcp_page_2026_07_11t03_39_59_214z_text_polish_panel, _playwright_mcp_page_2026_07_11t03_40_41_755z_lesson_review_panel, _playwright_mcp_page_2026_07_11t03_41_42_205z_publish_settings_modal [INFERRED 0.95]
- **Global App Navigation** — _playwright_mcp_page_2026_07_11t03_31_37_109z_global_navigation, _playwright_mcp_page_2026_07_11t03_31_37_109z_route_home, _playwright_mcp_page_2026_07_11t03_31_37_109z_route_guide, _playwright_mcp_page_2026_07_11t03_31_37_109z_explore_page, _playwright_mcp_page_2026_07_11t03_31_37_109z_route_history, _playwright_mcp_page_2026_07_11t03_31_37_109z_route_create, _playwright_mcp_page_2026_07_11t03_31_37_109z_route_profile, _playwright_mcp_page_2026_07_11t03_31_37_109z_route_settings [EXTRACTED 1.00]
- **Lesson Publish Pipeline** — _playwright_mcp_page_2026_07_11t03_41_42_205z_publish_settings_modal, _playwright_mcp_page_2026_07_11t03_43_44_780z_ai_publish_metadata_fill, _playwright_mcp_page_2026_07_11t03_41_42_205z_ai_tutor_settings, _playwright_mcp_page_2026_07_11t03_43_45_055z_ai_tutor_recommendation, _playwright_mcp_page_2026_07_11t03_43_07_510z_route_view, _playwright_mcp_page_2026_07_11t03_43_07_510z_access_visibility_controls [INFERRED 0.85]
- **Launch-Readiness Tier Sequence** — roadmap_tier_0, roadmap_tier_1, roadmap_tier_2, roadmap_tier_3, roadmap_tier_3_5, roadmap_tier_4, roadmap_tier_5, roadmap_tier_6 [EXTRACTED 1.00]
- **Tier 3.5 Shippable Phases** — roadmap_phase_3_5_a, roadmap_phase_3_5_g, plan_tier3_5_eleven_actions [EXTRACTED 1.00]
- **Non-Negotiable Golden Rules** — roadmap_golden_rule_1, roadmap_golden_rule_2 [EXTRACTED 1.00]
- **Dual AI Surfaces (Student vs Teacher)** — server_claude_post_api_chat_tutor, server_claude_post_api_creator_assist, client_claude_tutor_api, client_claude_creator_api [EXTRACTED 1.00]
- **Prompt Change Governance** — server_claude_persona_ts, server_claude_creator_prompts, server_prompt_notes [EXTRACTED 1.00]
- **Tier 3.A settings cleanup and font-size flow** — plan_tier3_settings_page, plan_tier3_appearance_store, plan_tier3_golden_rule_2, plan_tier3_decision_d4 [EXTRACTED 1.00]
- **Tier 3.B tutor memory view-and-wipe flow** — plan_tier3_profile_page, plan_tier3_tutor_memory_card, plan_tier3_tutor_memory_store, plan_tier3_memory_api_contract [EXTRACTED 1.00]
- **Golden Rules enforce anonymous full AI access** — claude_golden_rule_1, claude_golden_rule_2, server_claude_optional_auth, server_claude_post_api_chat_tutor [EXTRACTED 1.00]
- **Unified tutor request flow end-to-end** — client_claude_tutor_api, server_claude_post_api_chat_tutor, asking_flow_lesson_context, server_claude_tutor_persona, asking_flow_suggestions_contract [EXTRACTED 1.00]
- **Write-question AI feedback flow at 390px** — archive_arc2_plan_tier0_audit_audit_390_feedback_red_sedan_question, archive_arc2_plan_tier0_audit_audit_390_feedback_student_answer_text, archive_arc2_plan_tier0_audit_audit_390_feedback_ai_deep_evaluation_panel, archive_arc2_plan_tier0_audit_audit_390_feedback_stationary_praise, archive_arc2_plan_tier0_audit_audit_390_feedback_permanence_nuance [EXTRACTED 1.00]
- **Ask AI modal vertical layout stack** — archive_arc2_plan_tier0_audit_audit_390_modal_modal_header, archive_arc2_plan_tier0_audit_audit_390_modal_chat_history_area, archive_arc2_plan_tier0_audit_audit_390_modal_suggestion_chips_row, archive_arc2_plan_tier0_audit_audit_390_modal_textarea_input, archive_arc2_plan_tier0_audit_audit_390_modal_ask_send_button [EXTRACTED 1.00]
- **Lesson content hierarchy: title → section → example → interactive questions** — archive_arc2_plan_tier0_audit_audit_390_top_lesson_initial_state, archive_arc2_plan_tier0_audit_audit_390_feedback_lesson_title, archive_arc2_plan_tier0_audit_audit_390_feedback_reference_point_section, archive_arc2_plan_tier0_audit_audit_390_feedback_example_callout_box, archive_arc2_plan_tier0_audit_audit_390_top_write_question_empty [INFERRED 0.85]
- **TipTap-Fabric Editor Data Architecture** — client_src_components_readme_tiptap_fabric_editor, client_src_components_readme_canvas_context, client_src_components_readme_save_state, client_claude_tiptap_node_source_of_truth [EXTRACTED 1.00]

## Communities (210 total, 84 thin omitted)

### Community 0 - "auth.middleware.ts"
Cohesion: 0.05
Nodes (56): User, bulkSaveAnswers(), getAnswers(), isValidContentId(), saveAnswer(), createCategory(), deleteCategory(), getCategories() (+48 more)

### Community 1 - "tutor.controller.ts"
Cohesion: 0.06
Nodes (45): checkContentAccess(), ClientThreadEntry, coerceAccuracyPercent(), coerceBlockId(), coerceEvaluationLevel(), coerceFeedbackMode(), coerceMode(), coerceOptionalText() (+37 more)

### Community 2 - "formulaReducer.ts"
Cohesion: 0.06
Nodes (40): FormulaNode, FormulaNodeProps, FormulaSlot, SlotProps, createFormulaNode(), createFormulaRow(), createId(), createInitialFormulaState() (+32 more)

### Community 3 - "cn()"
Cohesion: 0.07
Nodes (32): NavLink, NavLinkCompatProps, Button(), buttonVariants, Card(), CardAction(), CardContent(), CardDescription() (+24 more)

### Community 4 - "EditorLeftSidebar.tsx"
Cohesion: 0.04
Nodes (38): BADGE_STYLES, CloudinaryUpload(), CloudinaryUploadProps, resolveUploadCategoryId(), BRUSH_COLORS, CanvasMediaPanel, CATEGORIES, CategoryBtn (+30 more)

### Community 5 - "LanguageToggle.tsx"
Cohesion: 0.07
Nodes (27): BlockMoveControls(), BlockMoveControlsProps, canMove(), getMoveContext(), moveBlock(), MoveContext, MoveDirection, FeedbackDiscussionPanelProps (+19 more)

### Community 6 - "seed-guide-demo.mjs"
Cohesion: 0.07
Nodes (33): args, BASE, demo, IMAGES_TS, login(), main(), ONLY, OUT_DIR (+25 more)

### Community 7 - "devDependencies"
Cohesion: 0.05
Nodes (43): devDependencies, eslint, @eslint/js, eslint-plugin-react-hooks, eslint-plugin-react-refresh, globals, happy-dom, playwright (+35 more)

### Community 8 - "EditorRightSidebar.tsx"
Cohesion: 0.05
Nodes (38): ActiveFormats, ALIGN_OPTIONS, CODE_LANGS, CodeAttrs, CodePanel, COLORS, DEFAULT_ACTIVE_FORMATS, DEFAULT_IMAGE_ATTRS (+30 more)

### Community 9 - "axios.ts"
Cohesion: 0.07
Nodes (29): PublicRoute(), api, buildForcedLoginUrl(), isProtectedPath(), isSafeRedirectTarget(), PROTECTED_PATH_PREFIXES, CLOUDINARY_CONFIG, QUICK_TRANSFORMS (+21 more)

### Community 10 - "App.tsx"
Cohesion: 0.06
Nodes (29): App(), ChangePassword, CloudinaryUpload, Create, CreatingShowcase, Dashboard, Explore, History (+21 more)

### Community 11 - "RichLine"
Cohesion: 0.08
Nodes (23): CanvasContext, CanvasContextType, CanvasProvider(), SelectedCategory, useCanvasContext(), DragHandlers, useCanvasDrag(), addAnimatedGif() (+15 more)

### Community 12 - "Lesson Editor Page"
Cohesion: 0.06
Nodes (39): Continue Learning Section (เรียนต่อ), Explore Page (/explore), Global Navigation Bar, Lesson Filter Tabs (ทั้งหมด/บุ๊กมาร์ก/ล่าสุด), Public Lesson Search, Route /create (Lesson Editor), Route /guide, Route /history (+31 more)

### Community 13 - "CanvasRightSidebar.tsx"
Cohesion: 0.05
Nodes (36): ASPECT_RATIOS, AspectRatio, CanvasImagePanel, CropOverlay, CropOverlayProps, FILTER_PRESETS, FilterPreset, FilterThumb (+28 more)

### Community 14 - "Status.tsx"
Cohesion: 0.09
Nodes (19): getClientEnvCheck(), REQUIRED_CLIENT_ENV_VARS, RequiredClientEnvVar, AiTutorCard(), ClientEnvCard(), formatRelative(), Status(), Translate (+11 more)

### Community 15 - "AiDraftDialog.tsx"
Cohesion: 0.13
Nodes (18): AiDraftDialog(), DraftTab, caretInsertPoint(), docEndPos(), formatHeadingOptionLabel(), formatLessonMarkdownPreview(), HeadingEntry, insertMarkdownAt() (+10 more)

### Community 16 - "persona.ts"
Cohesion: 0.12
Nodes (21): main(), tok(), cleanReply(), createSuggestionsGate(), INTERNAL_TAG_REGEX, parseTutorReply(), REPORT_LABEL_REGEX, REPORT_LABELS (+13 more)

### Community 17 - "QuestionFeedbackMode"
Cohesion: 0.11
Nodes (18): Commands, QuestionBlankChoiceAttrs, QuestionBlankChoiceNode, @tiptap/core, Commands, QuestionBlankWriteAttrs, QuestionBlankWriteNode, @tiptap/core (+10 more)

### Community 18 - "ShowcaseShell.tsx"
Cohesion: 0.16
Nodes (10): SceneImage(), SceneSection(), ShowcaseShell(), ShowcaseShellProps, CREATING_SCENES, GUIDE_IMAGES, LEARNING_SCENES, BilingualText (+2 more)

### Community 19 - "prompts.ts"
Cohesion: 0.18
Nodes (22): buildAgentSettingsPrompt(), buildCriticPrompt(), buildDistractorsPrompt(), buildDraftSectionPrompt(), buildFormulaLatexPrompt(), buildGenerateQuestionsPrompt(), buildGuideAnswerPrompt(), buildImportStructurePrompt() (+14 more)

### Community 20 - "tutorMemory.store.ts"
Cohesion: 0.12
Nodes (12): { mockFetchMemory, mockClearMemory, memoryState }, TutorMemoryCard(), { profileState, mockFetchProfile, mockSaveProfile }, { mockGet, mockDelete }, EMPTY_MEMORY, isMemoryEmpty(), normalizeMemory(), RecentTopic (+4 more)

### Community 21 - "devDependencies"
Cohesion: 0.09
Nodes (23): mongodb-memory-server, nodemon, devDependencies, mongodb-memory-server, nodemon, supertest, @types/bcryptjs, @types/cors (+15 more)

### Community 22 - "components.json"
Cohesion: 0.09
Nodes (21): aliases, components, hooks, lib, ui, utils, iconLibrary, menuAccent (+13 more)

### Community 23 - "auth.controller.ts"
Cohesion: 0.17
Nodes (16): changePassword(), generateToken(), googleLogin(), login(), normalizeEmail(), recheckToken(), register(), clearMemory() (+8 more)

### Community 24 - "validate.ts"
Cohesion: 0.19
Nodes (20): AgentSettingsSuggestion, clampText(), cleanText(), CRITIC_AREAS, CriticIssue, CriticResult, GeneratedQuestion, normalizeTemplate() (+12 more)

### Community 25 - "dependencies"
Cohesion: 0.10
Nodes (21): bcryptjs, cors, dotenv, express, express-rate-limit, google-auth-library, @google/genai, jsonwebtoken (+13 more)

### Community 26 - "compilerOptions"
Cohesion: 0.10
Nodes (20): compilerOptions, allowImportingTsExtensions, isolatedModules, jsx, lib, module, moduleResolution, noEmit (+12 more)

### Community 27 - "observability.service.ts"
Cohesion: 0.15
Nodes (15): errorHandler(), AiHealth, AiStatus, bootTime, getAiHealth(), getErrorSummary(), getErrorSummaryDetailed(), PublicRecordedError (+7 more)

### Community 28 - "Phase 3.B — Profile completion + tutor memory card"
Cohesion: 0.14
Nodes (20): appearance.store — app-wide font size, Client-only scope fence, Owner decision D3 — No PDPA page; delete account deferred, Owner decision D4 — Settings keeps only working rows, plan/tier3.md, Golden Rule 2 — Anonymous full access on /settings, Memory API contract (GET/DELETE /api/chat/memory), น้องมันฝรั่ง tutor persona name (+12 more)

### Community 29 - "lessonContext.service.ts"
Cohesion: 0.16
Nodes (17): buildLessonContext(), cache, cacheInsertionOrder, cacheSet(), DEFAULT_AGENT_SETTINGS, extractGuideAnswer(), getLessonContext(), getNodeTextContent() (+9 more)

### Community 30 - "useAuthStore"
Cohesion: 0.15
Nodes (11): Dashboard(), formatDate(), Explore(), ExploreTab, TABS, Login(), resolveRedirectTarget(), SESSION_BANNER_COPY (+3 more)

### Community 31 - "tutorApi.ts"
Cohesion: 0.19
Nodes (15): baseRequest, { mockPost }, AiUnavailableError, attachPersonality(), callTutor(), callTutorStream(), feedbackThreadToClientThread(), normalizeSuggestions() (+7 more)

### Community 32 - "content.model.ts"
Cohesion: 0.18
Nodes (12): main(), agentSettingsSchema, Content, contentSchema, IAgentSettings, IContent, AuthorSnapshot, buildAuthorSnapshot() (+4 more)

### Community 33 - "AiWritingAssistant.tsx"
Cohesion: 0.20
Nodes (11): AiWritingAssistant(), AiWritingToolCard(), PendingJob, selectionPreview(), useHasSelection(), WritingPreviewDialog(), TipTapCanvas(), AgentSettings (+3 more)

### Community 34 - "creatorApi.ts"
Cohesion: 0.18
Nodes (10): AiFormulaPanel(), AgentSettingsSuggestion, callCreator(), CreatorAction, CreatorActionMap, CreatorAiError, CreatorErrorCode, CriticIssue (+2 more)

### Community 35 - "TopNav.tsx"
Cohesion: 0.23
Nodes (8): authOnlyNavItems, publicNavItems, TopNav(), isGuideShowcasePath(), RevealOnScrollUpStore, useRevealOnScrollUp(), useRevealOnScrollUpListener(), useRevealOnScrollUpStore

### Community 36 - "package.json"
Cohesion: 0.13
Nodes (14): author, bugs, url, description, homepage, keywords, license, main (+6 more)

### Community 37 - "TipTap Lesson Editor"
Cohesion: 0.15
Nodes (14): Teacher AI Copilot, React.lazy Code Splitting, Hot Potato Client, SPA Generic OG Limitation, TipTap Lesson Editor, TipTap Node Single Source of Truth, tutorApi Single Bridge, Hot Potato Client README (+6 more)

### Community 38 - "TiptapViewer.tsx"
Cohesion: 0.24
Nodes (11): qaHistoryToClientThread(), getInitialZoom(), isAtTopOfVerticalScrollChain(), LessonAiAnswer, LessonAiMessage, parseTiptapContent(), TiptapViewer(), TiptapViewerProps (+3 more)

### Community 39 - "app.ts"
Cohesion: 0.22
Nodes (4): app, router, clearLessonContextCache(), { mockGenerateContentStream }

### Community 40 - "compilerOptions"
Cohesion: 0.14
Nodes (13): compilerOptions, esModuleInterop, forceConsistentCasingInFileNames, lib, module, outDir, resolveJsonModule, rootDir (+5 more)

### Community 41 - "History.tsx"
Cohesion: 0.22
Nodes (7): ContentCard(), ContentCardProps, authorWithRelativeTime(), formatRelative(), groupByDate(), History(), Translate

### Community 42 - "QuestionWriteView.tsx"
Cohesion: 0.17
Nodes (7): BlockAnswer, CreatorViewProps, QuestionWriteAttrs, ViewerViewProps, { mockCallCreator }, render(), renderView()

### Community 43 - "Creator 11 Actions"
Cohesion: 0.15
Nodes (13): Action agent_settings_suggest, Action critic, Creator 11 Actions, FormulaBlock latex attr, Action formula_latex, Action generate_questions, Product Principles (Owner), Content Model (+5 more)

### Community 44 - "ROADMAP.md Launch-Readiness v3"
Cohesion: 0.15
Nodes (13): ROADMAP.md Launch-Readiness v3, Golden Rule 1 — Never Ration Tokens, Owner Decision D1 — Deploy Last, Owner Decision D8 — Teacher AI Helper, Phase 1.A — Error Ring Buffer + AI Health, Phase 1.B — Status Page v2, Tier 0 — SEO & Link Previews, Tier 3.5 — Teacher AI Helper (+5 more)

### Community 45 - "creator.controller.ts"
Cohesion: 0.27
Nodes (12): ActionOutcome, bad(), callJson(), callPlain(), callPlainNonEmpty(), CREATOR_ACTIONS, CreatorAction, creatorAssist() (+4 more)

### Community 46 - "BlockDeleteButton.tsx"
Cohesion: 0.24
Nodes (8): BlockDeleteButton(), BlockDeleteButtonProps, deleteBlock(), createTemplateFromAction(), FormulaAttrs, FormulaCanvas(), inferInitialLatex(), TemplateInsert

### Community 48 - "AiQuestionDialog.tsx"
Cohesion: 0.27
Nodes (8): AiQuestionDialog(), ALL_TYPES, Difficulty, PreviewCardStatus, QuestionPreviewCard(), questionTypeLabel(), GeneratedQuestion, GeneratedQuestionType

### Community 49 - "EditorHeader.tsx"
Cohesion: 0.33
Nodes (7): EditorHeader, EditorHeaderProps, IconBtn, isSaveShortcut(), saveLessonNow(), { mockSetTiptapJson, mockSaveContent }, TipTapEditor()

### Community 50 - "QuestionBlankWriteView.tsx"
Cohesion: 0.29
Nodes (8): BlockAnswer, buildAnswers(), CreatorView(), CreatorViewProps, getBlankIndices(), InlineBlankTextareaProps, renderTemplatePieces(), ViewerView()

### Community 51 - "creator.test.ts"
Cohesion: 0.22
Nodes (7): callTutorModel(), TutorTurn, capturedCall(), capturedUserMessage(), GenerateContentArg, { mockGenerateContent }, VALID_CHOICE_JSON

### Community 52 - "Mobile lesson viewer page (TipTapViewer)"
Cohesion: 0.20
Nodes (10): App header bar (Intuita logo, Log in, dark mode, hamburger menu), Mobile lesson viewer page (TipTapViewer), 390px phone-first viewport audit (Tier 0 Phase 0.C C3), Tier 0 audit screenshot — AI feedback state at 390px, Potato tutor persona (มันฝรั่ง) warm greeting response, User greeting message: สวัสดี, Primary visual identity for Hot Potato project, Hot Potato pixel-art mascot character (+2 more)

### Community 53 - "archive/arc2/ROADMAP.md Universal"
Cohesion: 0.20
Nodes (10): Tier 0.A Client Rewire, archive/arc2/plan/tier0.md, archive/arc2/plan/tier2.md, CORS_ORIGINS Allowlist, archive/arc2/plan/tier3.md, archive/arc2/ROADMAP.md Universal, Arc2 Tier 0 — Conversation Core, Arc2 Tier 1 — Tutor Experience (+2 more)

### Community 54 - "AiToolsPanel.tsx"
Cohesion: 0.24
Nodes (4): AiCriticDialog(), AREA_LABELS, OpenDialog, CriticReport

### Community 55 - "writingAssist.ts"
Cohesion: 0.27
Nodes (7): getSelectionSnapshot(), GRADE_LEVELS, replaceRangeWithMarkdown(), SelectionSnapshot, WRITING_ACTIONS, WritingAction, ProofreadPreset

### Community 56 - "FabricCanvasView.tsx"
Cohesion: 0.27
Nodes (5): FabricCanvasNode, FabricCanvasEditable(), FabricCanvasReadOnly(), useFabricSetup(), useFabricSetupOptions

### Community 57 - "QuestionChoiceView.tsx"
Cohesion: 0.20
Nodes (5): BlockAnswer, ChoiceInputProps, CreatorViewProps, QuestionChoiceAttrs, ViewerViewProps

### Community 58 - "ResizableImage.ts"
Cohesion: 0.24
Nodes (8): clampSize(), createResizableImage(), getImageNodePos(), IMAGE_RESIZE_PLUGIN_KEY, startImageResize(), IImage, Image, imageSchema

### Community 59 - "CLAUDE.md (root)"
Cohesion: 0.28
Nodes (8): graphify-out Knowledge Graph, Hot Potato Platform, Playwright MCP Browser Automation, CLAUDE.md (root), Golden Rule 1 — Never Ration Tokens, Golden Rule 2 — Anonymous Full Access, Separate Git Repositories, Client/Server Two Halves

### Community 60 - "asking-flow.md"
Cohesion: 0.22
Nodes (8): archive/arc2/plan/tier1.md, Student Tutor Personalities, Tier 1.A SSE Streaming, Content.agent_settings, ChatSession Model, lessonContext.service.ts, StudentMemory, [SUGGESTIONS] Output Contract

### Community 61 - "numberedSectionHeadings.ts"
Cohesion: 0.33
Nodes (5): collectSectionHeadingFixes(), HeadingFix, NumberedSectionHeadingsExtension, numberedSectionHeadingsKey, stripLeadingSectionNumber()

### Community 62 - "QuestionAgentNode.ts"
Cohesion: 0.28
Nodes (6): Commands, QuestionAgentAttrs, QuestionAgentNode, @tiptap/core, BlockAnswer, ChatMessage

### Community 63 - "QuestionBlankChoiceView.tsx"
Cohesion: 0.39
Nodes (7): BlockAnswer, CreatorView(), CreatorViewProps, getBlankIndices(), remapCorrectByBlank(), renderTemplatePieces(), ViewerView()

### Community 64 - "POST /api/creator/assist"
Cohesion: 0.25
Nodes (9): Tier 3.5 Execution Plan, Phase 3.5.A — Creator Assist Endpoint, creator.controller.ts, services/creator/prompts.ts, services/tutor/genaiClient.ts, POST /api/creator/assist, Prompt Notes Log, Tier 3.5.A Creator Prompts (+1 more)

### Community 65 - "🥔 Hot Potato"
Cohesion: 0.22
Nodes (8): Architecture at a glance, 🥔 Hot Potato, License, Quick start (local), Repository layout, Roadmap (phase 2), Tech stack, What it does

### Community 66 - "scripts"
Cohesion: 0.22
Nodes (9): scripts, build, check-models, dev, promote, start, test, test:ai (+1 more)

### Community 67 - "Ask AI modal dialog (centered overlay on dimmed lesson page)"
Cohesion: 0.25
Nodes (8): AI message bubble (white, left-aligned), Ask AI modal dialog (centered overlay on dimmed lesson page), Ask send button (paper plane icon, light purple), Chat history scroll area with alternating user/AI bubbles, Modal header: Ask AI title, Clear chat, close X, Tier 0 audit screenshot — Ask AI modal overlay at 390px, Modal text input placeholder: Ask AI about this section..., User message bubble (light purple, right-aligned)

### Community 68 - "Lesson page initial state before answer submission"
Cohesion: 0.25
Nodes (8): Ask AI button on question card (robot icon, bottom-right), Tier 0 audit screenshot — inline chat thread and quiz at 390px, True/false question: distance equals displacement on A→B→A round trip, Ask AI floating action button on lesson page, Lesson page initial state before answer submission, Partial third question card (students visiting school scenario), Tier 0 audit screenshot — lesson top/unsubmitted state at 390px, True/false distance vs displacement question (unanswered)

### Community 69 - "dependencies"
Cohesion: 0.43
Nodes (8): dependencies, @tiptap/extension-color, @tiptap/extension-image, @tiptap/extension-link, @tiptap/extension-table, @tiptap/extension-table-row, @tiptap/extension-text-align, @tiptap/extension-link

### Community 70 - "MarkdownMessage.tsx"
Cohesion: 0.25
Nodes (3): ALLOWED_ELEMENTS, components, MarkdownMessageProps

### Community 71 - "PublishAutofill.test.tsx"
Cohesion: 0.29
Nodes (6): DEFAULT_AGENT, META, { mockCallCreator }, render(), renderModal(), SUGGESTION

### Community 73 - "Write question: Can a red sedan parked roadside be a referen"
Cohesion: 0.33
Nodes (7): AI DEEP EVALUATION feedback panel (light-purple card), Write question: Can a red sedan parked roadside be a reference point?, AI praise for identifying stationary requirement, Student answer: ได้ เพราะรถจอดอยู่กับที่ (Yes, because the car is stationary), Try again link below write-question input, Write question: red sedan roadside reference point (unanswered), Write question card with empty placeholder Write your answer...

### Community 74 - "index.html — Site-Wide Thai SEO/OG/Twitter Meta Tags"
Cohesion: 0.33
Nodes (7): Document Language lang=th (Crawler-Facing Default), og:image — /og-image.png (584×584 Square Logo), index.html — Site-Wide Thai SEO/OG/Twitter Meta Tags, robots.txt — Allow All Crawlers, public/ Stable Asset URLs (favicon.png, og-image.png), seo.test.ts — SEO Contract Regression Tests, Tier 0 Execution Plan — SEO & Link Previews Phase 0.A

### Community 75 - "Hot Potato brand mascot (og-image)"
Cohesion: 0.29
Nodes (7): Browser tab favicon purpose, client/public/favicon.png, Pixel-art Hot Potato mascot (favicon), 8-bit retro visual identity, Hot Potato brand mascot (og-image), client/public/og-image.png, Open Graph social preview image

### Community 77 - "AiFormulaPanel.test.tsx"
Cohesion: 0.43
Nodes (6): expandAndFill(), findButton(), { mockCallCreator }, render(), renderCanvas(), setInputValue()

### Community 78 - "Login.test.tsx"
Cohesion: 0.29
Nodes (4): mockLogin, mockLoginWithGoogle, mockNavigate, mockRegister

### Community 79 - "theme.store.ts"
Cohesion: 0.29
Nodes (4): initialTheme, ThemeMode, ThemeState, useThemeStore

### Community 80 - "POST /api/chat/tutor"
Cohesion: 0.33
Nodes (7): Tier 3.5 Golden Rules Scope, Careful-Not-To-Break: Anonymous Full Access, Golden Rule 2 — Anonymous Full Access, auth.middleware.ts, POST /api/chat/tutor, tutor.controller.ts, Tutor Modes (4)

### Community 81 - "tutor.test.ts"
Cohesion: 0.43
Nodes (6): capturedCall(), capturedContents(), capturedModel(), capturedSystemInstruction(), GenerateContentArg, { mockGenerateContent }

### Community 82 - "archive/arc1/ROADMAP-detailed.md"
Cohesion: 0.33
Nodes (6): Phase -1 Test Harness, POST /api/chat/tutor (Arc1 Phase 1), archive/arc1/ROADMAP-detailed.md, archive/arc1/ROADMAP.md, Arc1 AI Tutor Rework Roadmap, Arc2 Universal Roadmap

### Community 83 - "Reference point (จุดอ้างอิง) — fixed origin for measuring di"
Cohesion: 0.40
Nodes (6): Electric pole reference-point example (5 meters to the right), Light-purple bordered example callout in lesson body, Lesson title: การระบุตำแหน่งของวัตถุ (Identifying Object Position), Reference point (จุดอ้างอิง) — fixed origin for measuring distance and direction, Section 1: การกำหนดจุดอ้างอิง (Defining a Reference Point), Reference point Q&A thread (what is it, why needed)

### Community 84 - "Suggested follow-up question chips (purple outline pills)"
Cohesion: 0.33
Nodes (6): AI nuance: reference point should be permanent/reliable, car may drive away, Chip: Common reference points in daily life, Chip: What if the reference point can move?, Chip: Must a reference point be a real object?, Suggested follow-up question chips (purple outline pills), Lesson follow-up text on red car reference point durability

### Community 85 - "AiCriticButton.test.tsx"
Cohesion: 0.33
Nodes (3): fakeEditor, { mockCallCreator, mockSaveContent }, REPORT

### Community 87 - "AiWritingAssistant.test.tsx"
Cohesion: 0.40
Nodes (3): makeEditorWithSelection(), { mockCallCreator }, selectAllText()

### Community 88 - "questionEvaluation.ts"
Cohesion: 0.40
Nodes (4): Choice, ChoiceEvaluation, evaluateChoiceAnswer(), fourChoices

### Community 89 - "PublishSettingsModal.tsx"
Cohesion: 0.33
Nodes (5): ACCESS_TYPE_LABELS, ACCESS_TYPES, PublishSettingsModal(), PublishSettingsModalProps, TitleImageGalleryModal

### Community 90 - "learning_history.model.ts"
Cohesion: 0.47
Nodes (4): ILearningHistory, LearningHistory, learningHistorySchema, touchLearningHistory()

### Community 91 - "parse.ts"
Cohesion: 0.60
Nodes (3): askJson(), CreatorAiInvalidError, parseJsonLoose()

### Community 92 - "Inline contextual chat panel embedded in lesson scroll"
Cohesion: 0.40
Nodes (5): Chip: Was the red sedan a valid reference point?, Inline contextual chat panel embedded in lesson scroll, Inline chat input: Try another answer or ask..., Submit button (purple, paper plane icon), Inline suggestion chips (what to learn next, red car question, anything else)

### Community 93 - "SearchHighlight.ts"
Cohesion: 0.40
Nodes (3): SearchHighlightExtension, searchHighlightKey, SearchMatch

### Community 95 - "vite-env.d.ts"
Cohesion: 0.40
Nodes (4): axios, AxiosRequestConfig, ImportMeta, ImportMetaEnv

### Community 96 - "observability.service.ts — In-Memory Error Ring Buffer + AI "
Cohesion: 0.40
Nodes (5): Error Privacy — No Messages/Stacks/Bodies in Public Payload, GET /api/status/all Extended Payload (checks.ai + checks.errors), observability.service.ts — In-Memory Error Ring Buffer + AI Health, Passive AI Health Recording (Never Ping Gemini from Status), Tier 1 Execution Plan — Observability

### Community 97 - "chat_session.model.ts"
Cohesion: 0.40
Nodes (4): ChatSession, chatSessionSchema, IChatMessage, IChatSession

### Community 98 - "personality.ts"
Cohesion: 0.70
Nodes (3): resolvePersonality(), TUTOR_PERSONALITIES, TutorPersonality

### Community 101 - "seo.test.ts"
Cohesion: 0.67
Nodes (3): html, p(), read()

### Community 102 - "playwright"
Cohesion: 0.50
Nodes (3): playwright, npx, @playwright/mcp

### Community 103 - "category.model.ts"
Cohesion: 0.50
Nodes (3): Category, categorySchema, ICategory

### Community 104 - "user_content.model.ts"
Cohesion: 0.50
Nodes (3): IUserContent, UserContent, userContentSchema

### Community 105 - "user_data.model.ts"
Cohesion: 0.50
Nodes (3): IUserData, UserData, userDataSchema

### Community 106 - "@tiptap/extension-table-cell"
Cohesion: 0.67
Nodes (3): @tiptap/extension-table-cell, @tiptap/extension-table-header, @tiptap/extension-table-cell

### Community 109 - "Tier 2 Execution Plan — Bundle Split + Self-Hosted Fonts"
Cohesion: 0.67
Nodes (3): manualChunks — tiptap/fabric/katex Vendor Pinning, React.lazy + PageLoader Suspense Fallback, Tier 2 Execution Plan — Bundle Split + Self-Hosted Fonts

## Ambiguous Edges - Review These
- `App header bar (Intuita logo, Log in, dark mode, hamburger menu)` → `Hot Potato logo PNG brand asset`  [AMBIGUOUS]
  archive/arc2/plan/tier0-audit/audit-390-feedback.jpeg · relation: conceptually_related_to

## Knowledge Gaps
- **708 isolated node(s):** `$schema`, `style`, `rsc`, `tsx`, `config` (+703 more)
  These have ≤1 connection - possible missing edges or undocumented components.
- **84 thin communities (<3 nodes) omitted from report** — run `graphify query` to explore isolated nodes.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **What is the exact relationship between `App header bar (Intuita logo, Log in, dark mode, hamburger menu)` and `Hot Potato logo PNG brand asset`?**
  _Edge tagged AMBIGUOUS (relation: conceptually_related_to) - confidence is low._
- **Why does `User` connect `auth.middleware.ts` to `useAuthStore`?**
  _High betweenness centrality (0.023) - this node is a cross-community bridge._
- **Why does `useCanvasStore` connect `AiWritingAssistant.tsx` to `TiptapViewer.tsx`?**
  _High betweenness centrality (0.021) - this node is a cross-community bridge._
- **What connects `$schema`, `style`, `rsc` to the rest of the system?**
  _730 weakly-connected nodes found - possible documentation gaps or missing edges._
- **Should `auth.middleware.ts` be split into smaller, more focused modules?**
  _Cohesion score 0.0532724505327245 - nodes in this community are weakly interconnected._
- **Should `tutor.controller.ts` be split into smaller, more focused modules?**
  _Cohesion score 0.05764411027568922 - nodes in this community are weakly interconnected._
- **Should `formulaReducer.ts` be split into smaller, more focused modules?**
  _Cohesion score 0.06259426847662142 - nodes in this community are weakly interconnected._