# Graph Report - .  (2026-07-16)

## Corpus Check
- cluster-only mode — file stats not available

## Summary
- 1882 nodes · 3776 edges · 171 communities (103 shown, 68 thin omitted)
- Extraction: 99% EXTRACTED · 1% INFERRED · 0% AMBIGUOUS · INFERRED: 48 edges (avg confidence: 0.81)
- Token cost: 0 input · 0 output

## Graph Freshness
- Built from commit: `3480b3ee`
- Run `git rev-parse HEAD` and compare to check if the graph is stale.
- Run `graphify update .` after code changes (no API cost).

## Community Hubs (Navigation)
- cn
- creator.controller.ts
- app.ts
- EditorRightSidebar.tsx
- seed-guide-demo.mjs
- devDependencies
- observability.service.ts
- useAppI18n
- devDependencies
- Content
- answer.controller.ts
- compilerOptions
- App.tsx
- AiDraftDialog.tsx
- CanvasRightSidebar.tsx
- TiptapViewer.tsx
- components.json
- AiQuestionDialog.tsx
- CanvasLeftSidebar.tsx
- Profile.tsx
- QuestionChoiceView.tsx
- AiWritingAssistant.tsx
- creatorGradeLevel.store.ts
- Setting.tsx
- RichLine
- QuestionBlankWriteView.tsx
- tutorApi.ts
- axios.ts
- ThemeToggle.tsx
- tutorMemory.store.ts
- CanvasImagePanel.tsx
- i18n.ts
- Intuition Documentation Layer
- AiDraftDialog.test.tsx
- QuestionBlankChoiceView.tsx
- useFabric.ts
- language.store.ts
- FabricCanvasView.tsx
- editorExtensions.ts
- appearance.store.ts
- QuestionWriteView.tsx
- dependencies
- editor.i18n.ts
- MarkdownMessage.tsx
- CanvasContext.tsx
- Explore.tsx
- AiCriticButton.test.tsx
- AiToolsPanel.test.tsx
- AiFormulaPanel.test.tsx
- TopNav.tsx
- Login.test.tsx
- ROADMAP.md
- Ask AI modal dialog (centered overlay on dimmed lesson page)
- CloudinaryUpload.tsx
- Landing.tsx
- QuestionWriteGuideAnswer.test.tsx
- vite-env.d.ts
- seo.test.ts
- @tiptap/extension-table-cell
- image.controller.ts
- History.tsx
- axios
- class-variance-authority
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
- vercel.json
- tutor.controller.ts
- EditorLeftSidebar.tsx
- persona.ts
- TipTapEditor.tsx
- dependencies
- archive/arc2/ROADMAP.md Universal
- auth.middleware.ts
- creatorApi.ts
- types.ts
- compilerOptions
- memory.ts
- FormulaCanvas.tsx
- formulaReducer.ts
- package.json
- user.controller.ts
- lessonContext.service.ts
- TipTap Document as Single Source of Truth
- User
- docs/ โ€” System Intuition Layer
- tutor.test.ts
- parse.ts
- AI Tutor Call Flow
- operations โ€” Deploy and Run
- genaiClient.ts
- scripts
- memory.test.ts
- Sidebar.tsx
- PublishAutofill.test.tsx
- Golden Rule 2 โ€” Anonymous Full Access
- numberedSectionHeadings.ts
- chat.routes.ts
- Status.test.tsx
- playwright
- Golden Rule 1 โ€” Never Ration Tokens
- Render Cold Starts
- structured 401
- check-models.ts
- index.ts
- notes.md
- Authoring Plane (Teacher)
- Server Request Lifecycle
- StudentMemory Entity
- ADR-007 Viewer CSS zoom
- supertest
- Student Answers Flow
- Nine Mongoose Models
- User Entity
- access_type
- agent_settings
- block / block id
- feedbackMode
- mode (tutor mode)
- ADR-001 Two Separate Git Repos
- ADR-010 Tutor Light Markdown
- ADR-012 Deterministic Client Evaluation
- ADR-013 Personality Under agent_settings
- ADR-016 Loanword Rule
- ADR-019 Denormalized Author Names
- Environment Variable Matrix
- PublicRoute Guard
- RequireLogin Guard

## God Nodes (most connected - your core abstractions)
1. `useAppI18n()` - 69 edges
2. `cn()` - 50 edges
3. `useCanvasStore` - 38 edges
4. `useAuthStore` - 29 edges
5. `runAction()` - 29 edges
6. `tutorChat()` - 28 edges
7. `QuestionFeedbackMode` - 22 edges
8. `useColdStartHint()` - 21 edges
9. `RichLine` - 20 edges
10. `AiDraftDialog()` - 19 edges

## Surprising Connections (you probably didn't know these)
- `graphify-out Knowledge Graph` --semantically_similar_to--> `Documentation Intuition Layer`  [INFERRED] [semantically similar]
  AGENT.md → archive/arc3/ROADMAP-docs.md
- `p()` --indirect_call--> `_wireRichLine()`  [INFERRED]
  scripts/guide-demo-docs.mjs → src/hooks/useFabric.ts
- `h()` --indirect_call--> `AiDraftDialog()`  [INFERRED]
  scripts/guide-demo-docs.mjs → src/components/editor/ai/AiDraftDialog.tsx
- `h()` --indirect_call--> `AiQuestionDialog()`  [INFERRED]
  scripts/guide-demo-docs.mjs → src/components/editor/ai/AiQuestionDialog.tsx
- `q()` --indirect_call--> `AiDraftDialog()`  [INFERRED]
  scripts/guide-demo-docs.mjs → src/components/editor/ai/AiDraftDialog.tsx

## Import Cycles
- None detected.

## Communities (171 total, 68 thin omitted)

### Community 0 - "cn"
Cohesion: 0.08
Nodes (32): ACCESS_TYPE_LABELS, ACCESS_TYPES, PublishSettingsModalProps, TitleImageGalleryModal, Card(), CardAction(), CardContent(), CardDescription() (+24 more)

### Community 1 - "creator.controller.ts"
Cohesion: 0.08
Nodes (61): ActionOutcome, bad(), callJson(), callPlain(), callPlainNonEmpty(), CREATOR_ACTIONS, CreatorAction, creatorAssist() (+53 more)

### Community 2 - "app.ts"
Cohesion: 0.14
Nodes (17): app, router, router, clearLessonContextCache(), { mockGenerateContent, mockVerifyIdToken }, validUser, capturedCall(), capturedUserMessage() (+9 more)

### Community 3 - "EditorRightSidebar.tsx"
Cohesion: 0.05
Nodes (42): ActiveFormats, ALIGN_OPTIONS, CODE_LANGS, CodeAttrs, CodePanel, COLORS, DEFAULT_ACTIVE_FORMATS, DEFAULT_IMAGE_ATTRS (+34 more)

### Community 4 - "seed-guide-demo.mjs"
Cohesion: 0.07
Nodes (33): args, BASE, demo, IMAGES_TS, login(), main(), ONLY, OUT_DIR (+25 more)

### Community 5 - "devDependencies"
Cohesion: 0.05
Nodes (43): eslint, @eslint/js, eslint-plugin-react-hooks, eslint-plugin-react-refresh, globals, happy-dom, devDependencies, eslint (+35 more)

### Community 6 - "observability.service.ts"
Cohesion: 0.15
Nodes (21): checkRequiredEnv(), getAllStatus(), getDatabaseStatus(), getEnvStatus(), getServerStatus(), mongoStateLabel(), REQUIRED_ENV_VARS, uptimeFormatted() (+13 more)

### Community 7 - "useAppI18n"
Cohesion: 0.11
Nodes (23): getClientEnvCheck(), REQUIRED_CLIENT_ENV_VARS, RequiredClientEnvVar, useAppI18n(), AiTutorCard(), ClientEnvCard(), DatabaseCard(), formatRelative() (+15 more)

### Community 8 - "devDependencies"
Cohesion: 0.09
Nodes (23): mongodb-memory-server, nodemon, devDependencies, mongodb-memory-server, nodemon, ts-node, @types/bcryptjs, @types/cors (+15 more)

### Community 9 - "Content"
Cohesion: 0.24
Nodes (14): main(), createBlankContent(), deleteContent(), isValidContentId(), loadContent(), sanitizeAgentSettings(), searchContent(), updateContent() (+6 more)

### Community 10 - "answer.controller.ts"
Cohesion: 0.19
Nodes (15): bulkSaveAnswers(), getAnswers(), isValidContentId(), saveAnswer(), getHistory(), recordVisit(), ILearningHistory, LearningHistory (+7 more)

### Community 11 - "compilerOptions"
Cohesion: 0.08
Nodes (24): DOM, DOM.Iterable, ESNext, ./src/*, compilerOptions, allowImportingTsExtensions, isolatedModules, jsx (+16 more)

### Community 12 - "App.tsx"
Cohesion: 0.08
Nodes (26): App(), ChangePassword, CloudinaryUpload, Create, CreatingShowcase, Dashboard, Explore, History (+18 more)

### Community 13 - "AiDraftDialog.tsx"
Cohesion: 0.21
Nodes (21): AiDraftDialog(), DraftTab, AiDraftLauncher(), caretInsertPoint(), docEndPos(), formatHeadingBelowOptionLabel(), formatHeadingOptionLabel(), formatLessonMarkdownPreview() (+13 more)

### Community 14 - "CanvasRightSidebar.tsx"
Cohesion: 0.08
Nodes (23): ARROW_TYPES, ColorRow, FontDropdown, FONTS, IconBtn, LayerSection, LINE_STYLES, MixedAttrs (+15 more)

### Community 15 - "TiptapViewer.tsx"
Cohesion: 0.23
Nodes (10): AiErrorRetry(), AiErrorRetryProps, getQuestionAgentViewportContext(), getInitialZoom(), isAtTopOfVerticalScrollChain(), LessonAiAnswer, LessonAiMessage, parseTiptapContent() (+2 more)

### Community 16 - "components.json"
Cohesion: 0.09
Nodes (21): aliases, components, hooks, lib, ui, utils, iconLibrary, menuAccent (+13 more)

### Community 17 - "AiQuestionDialog.tsx"
Cohesion: 0.20
Nodes (13): AiQuestionDialog(), ALL_TYPES, Difficulty, captureQuestionInsertPos(), generatedQuestionToNode(), insertGeneratedQuestions(), QuestionNodeJson, resolveInsertPos() (+5 more)

### Community 18 - "CanvasLeftSidebar.tsx"
Cohesion: 0.11
Nodes (18): BRUSH_COLORS, CanvasLeftSidebar(), CanvasMediaPanel, CATEGORIES, CategoryBtn, CONNECTOR_PRESETS, ConnectorPreset, ConnectorPreview (+10 more)

### Community 19 - "Profile.tsx"
Cohesion: 0.18
Nodes (12): PersonalityPicker(), Button(), buttonVariants, uploadImage(), Profile(), { profileState, mockFetchProfile, mockSaveProfile }, TiptapView(), useProfileStore (+4 more)

### Community 20 - "QuestionChoiceView.tsx"
Cohesion: 0.20
Nodes (10): QuestionChoiceAttrs, BlockAnswer, ChoiceInputProps, CreatorViewProps, QuestionChoiceAttrs, ViewerViewProps, Choice, ChoiceEvaluation (+2 more)

### Community 21 - "AiWritingAssistant.tsx"
Cohesion: 0.22
Nodes (13): AiWritingAssistant(), AiWritingToolCard(), PendingJob, selectionPreview(), useHasSelection(), WritingPreviewDialog(), getSelectionSnapshot(), GRADE_LEVELS (+5 more)

### Community 22 - "creatorGradeLevel.store.ts"
Cohesion: 0.14
Nodes (7): GENERATED, { mockCallCreator }, makeEditorWithSelection(), { mockCallCreator }, selectAllText(), CreatorGradeLevelState, useCreatorGradeLevelStore

### Community 23 - "Setting.tsx"
Cohesion: 0.13
Nodes (16): AlertDialog(), AlertDialogAction(), AlertDialogCancel(), AlertDialogContent(), AlertDialogDescription(), AlertDialogFooter(), AlertDialogHeader(), AlertDialogOverlay() (+8 more)

### Community 25 - "QuestionBlankWriteView.tsx"
Cohesion: 0.27
Nodes (10): buildAnswers(), CreatorView(), CreatorViewProps, getBlankIndices(), InlineBlankTextarea(), InlineBlankTextareaProps, renderTemplatePieces(), ViewerView() (+2 more)

### Community 26 - "tutorApi.ts"
Cohesion: 0.19
Nodes (15): baseRequest, { mockPost }, AiUnavailableError, attachPersonality(), callTutor(), callTutorStream(), normalizeSuggestions(), parseSseBuffer() (+7 more)

### Community 27 - "axios.ts"
Cohesion: 0.21
Nodes (7): api, buildForcedLoginUrl(), isProtectedPath(), PROTECTED_PATH_PREFIXES, ProfileData, ProfileState, { mockGet, mockPut, authSetState }

### Community 28 - "ThemeToggle.tsx"
Cohesion: 0.17
Nodes (9): ThemeToggle(), ThemeToggleProps, Login(), resolveRedirectTarget(), SESSION_BANNER_COPY, initialTheme, ThemeMode, ThemeState (+1 more)

### Community 29 - "tutorMemory.store.ts"
Cohesion: 0.17
Nodes (11): { mockFetchMemory, mockClearMemory, memoryState }, TutorMemoryCard(), { mockGet, mockDelete }, EMPTY_MEMORY, isMemoryEmpty(), normalizeMemory(), RecentTopic, toStringArray() (+3 more)

### Community 30 - "CanvasImagePanel.tsx"
Cohesion: 0.14
Nodes (13): ASPECT_RATIOS, AspectRatio, CanvasImagePanel, CropOverlay, CropOverlayProps, FILTER_PRESETS, FilterPreset, FilterThumb (+5 more)

### Community 31 - "i18n.ts"
Cohesion: 0.32
Nodes (7): SceneImage(), SceneSection(), ShowcaseShellProps, GUIDE_IMAGES, BilingualText, GuideScene, SceneImageRef

### Community 34 - "QuestionBlankChoiceView.tsx"
Cohesion: 0.17
Nodes (14): FeedbackDiscussionPanelProps, FeedbackThreadMessage, BlockAnswer, CreatorView(), getBlankIndices(), remapCorrectByBlank(), renderTemplatePieces(), ViewerView() (+6 more)

### Community 35 - "useFabric.ts"
Cohesion: 0.24
Nodes (11): addAnimatedGif(), addVideo(), _angleDeg(), _buildArrowhead(), createStarPoints(), _dashArray(), isAnimatedMedia(), isVideoMedia() (+3 more)

### Community 36 - "language.store.ts"
Cohesion: 0.11
Nodes (9): LanguageToggle(), LanguageToggleProps, options, setLanguage, CREATING_SCENES, LEARNING_SCENES, AppLanguage, LanguageState (+1 more)

### Community 37 - "FabricCanvasView.tsx"
Cohesion: 0.36
Nodes (7): CanvasRightSidebar(), FabricCanvasEditable(), FabricCanvasReadOnly(), useCanvasContext(), wireRichLinesOnCanvas(), useFabricSetup(), useFabricSetupOptions

### Community 38 - "editorExtensions.ts"
Cohesion: 0.09
Nodes (26): FabricCanvasNode, FabricCanvasView(), OutlineDraftParagraph, Commands, QuestionBlankChoiceAttrs, QuestionBlankChoiceNode, @tiptap/core, CreatorViewProps (+18 more)

### Community 39 - "appearance.store.ts"
Cohesion: 0.31
Nodes (7): AppearanceState, applyFontSize(), detectInitialFontSize(), FONT_SIZE_OPTIONS, FontSize, initialFontSize, isFontSize()

### Community 40 - "QuestionWriteView.tsx"
Cohesion: 0.16
Nodes (17): AiCriticDialog(), AREA_LABELS, QuestionAgentView(), CreatorView(), ViewerView(), CreatorView(), CreatorViewProps, QuestionWriteAttrs (+9 more)

### Community 41 - "dependencies"
Cohesion: 0.43
Nodes (8): dependencies, @tiptap/extension-color, @tiptap/extension-image, @tiptap/extension-link, @tiptap/extension-table, @tiptap/extension-table-row, @tiptap/extension-text-align, @tiptap/extension-link

### Community 42 - "editor.i18n.ts"
Cohesion: 0.11
Nodes (21): AI_THINKING_MESSAGES, AiThinkingMessage(), AiThinkingMessageProps, COLD_START_MESSAGE, nextIndex(), BlockDeleteButton(), BlockDeleteButtonProps, deleteBlock() (+13 more)

### Community 43 - "MarkdownMessage.tsx"
Cohesion: 0.25
Nodes (3): ALLOWED_ELEMENTS, components, MarkdownMessageProps

### Community 44 - "CanvasContext.tsx"
Cohesion: 0.32
Nodes (6): CanvasContext, CanvasContextType, CanvasProvider(), SelectedCategory, DragHandlers, useCanvasDrag()

### Community 45 - "Explore.tsx"
Cohesion: 0.13
Nodes (12): ContentCard(), ContentCardProps, formatAuthorLine(), CreatorDashboard(), Explore(), ExploreTab, TABS, BookmarkState (+4 more)

### Community 46 - "AiCriticButton.test.tsx"
Cohesion: 0.29
Nodes (3): fakeEditor, { mockCallCreator, mockSaveContent }, REPORT

### Community 48 - "AiFormulaPanel.test.tsx"
Cohesion: 0.43
Nodes (6): expandAndFill(), findButton(), { mockCallCreator }, render(), renderCanvas(), setInputValue()

### Community 49 - "TopNav.tsx"
Cohesion: 0.14
Nodes (15): NavLink, NavLinkCompatProps, authOnlyNavItems, publicNavItems, TopNav(), DropdownMenu(), DropdownMenuContent(), DropdownMenuTrigger() (+7 more)

### Community 50 - "Login.test.tsx"
Cohesion: 0.29
Nodes (4): mockLogin, mockLoginWithGoogle, mockNavigate, mockRegister

### Community 51 - "ROADMAP.md"
Cohesion: 0.08
Nodes (45): appearance.store (Font Size), plan/guide.md, plan/tier0.md, Tier 0: SEO and Link Previews, plan/tier1.md, Tier 1: Observability, plan/tier2.md, Tier 2: Bundle Split and Fonts (+37 more)

### Community 52 - "Ask AI modal dialog (centered overlay on dimmed lesson page)"
Cohesion: 0.05
Nodes (46): AI DEEP EVALUATION feedback panel (light-purple card), App header bar (Intuita logo, Log in, dark mode, hamburger menu), Electric pole reference-point example (5 meters to the right), Light-purple bordered example callout in lesson body, Lesson title: การระบุตำแหน่งของวัตถุ (Identifying Object Position), Mobile lesson viewer page (TipTapViewer), AI nuance: reference point should be permanent/reliable, car may drive away, 390px phone-first viewport audit (Tier 0 Phase 0.C C3) (+38 more)

### Community 53 - "CloudinaryUpload.tsx"
Cohesion: 0.12
Nodes (19): BADGE_STYLES, CloudinaryUpload(), CloudinaryUploadProps, resolveUploadCategoryId(), applyTransform(), CLOUDINARY_CONFIG, QUICK_TRANSFORMS, Transform (+11 more)

### Community 54 - "Landing.tsx"
Cohesion: 0.50
Nodes (3): Landing(), ROLE_CARDS, RoleCard

### Community 55 - "QuestionWriteGuideAnswer.test.tsx"
Cohesion: 0.50
Nodes (3): { mockCallCreator }, render(), renderView()

### Community 56 - "vite-env.d.ts"
Cohesion: 0.40
Nodes (4): axios, AxiosRequestConfig, ImportMeta, ImportMetaEnv

### Community 57 - "seo.test.ts"
Cohesion: 0.67
Nodes (3): html, p(), read()

### Community 58 - "@tiptap/extension-table-cell"
Cohesion: 0.67
Nodes (3): @tiptap/extension-table-cell, @tiptap/extension-table-header, @tiptap/extension-table-cell

### Community 59 - "image.controller.ts"
Cohesion: 0.18
Nodes (21): createCategory(), deleteCategory(), getCategories(), getImagesByCategory(), parseCategoryParam(), updateCategory(), assignCategory(), clearImages() (+13 more)

### Community 60 - "History.tsx"
Cohesion: 0.19
Nodes (11): authorWithRelativeTime(), formatRelative(), groupByDate(), History(), Translate, sampleEntries, TipTapCanvas(), HistoryContentPreview (+3 more)

### Community 107 - "tutor.controller.ts"
Cohesion: 0.13
Nodes (25): checkContentAccess(), ClientThreadEntry, coerceAccuracyPercent(), coerceBlockId(), coerceEvaluationLevel(), coerceFeedbackMode(), coerceMode(), coerceOptionalText() (+17 more)

### Community 108 - "EditorLeftSidebar.tsx"
Cohesion: 0.09
Nodes (19): ALIGN_OPTIONS, CATEGORIES, CategoryBtn, CategoryKey, COLORS, DEFAULT_ACTIVE, FormulaBtn, FormulaGroup (+11 more)

### Community 109 - "persona.ts"
Cohesion: 0.16
Nodes (17): main(), tok(), cacheSet(), getLessonContext(), agentSettingsDifferFromDefaults(), buildFeedbackUserTurn(), buildSystemInstruction(), buildTeacherSettingsSection() (+9 more)

### Community 110 - "TipTapEditor.tsx"
Cohesion: 0.18
Nodes (14): createEditorExtensions(), EditorHeader, EditorHeaderProps, IconBtn, isSaveShortcut(), saveLessonNow(), { mockSetTiptapJson, mockSaveContent }, TipTapEditor() (+6 more)

### Community 111 - "dependencies"
Cohesion: 0.10
Nodes (21): axios, bcryptjs, cors, dotenv, express, express-rate-limit, google-auth-library, @google/genai (+13 more)

### Community 112 - "archive/arc2/ROADMAP.md Universal"
Cohesion: 0.10
Nodes (20): Phase -1 Test Harness, POST /api/chat/tutor (Arc1 Phase 1), archive/arc1/ROADMAP-detailed.md, archive/arc1/ROADMAP.md, Arc1 AI Tutor Rework Roadmap, Tier 0.A Client Rewire, archive/arc2/plan/tier0.md, archive/arc2/plan/tier1.md (+12 more)

### Community 113 - "auth.middleware.ts"
Cohesion: 0.19
Nodes (13): extractBearerToken(), JWT_VERIFY_OPTIONS, loadUserFromToken(), optionalAuth(), protect(), sendReloginResponse(), TokenErrorCode, createAuthRateLimiter() (+5 more)

### Community 114 - "creatorApi.ts"
Cohesion: 0.12
Nodes (10): OpenDialog, AgentSettingsSuggestion, CreatorAction, CreatorActionMap, CreatorAiError, CreatorErrorCode, CriticIssue, CriticReport (+2 more)

### Community 115 - "types.ts"
Cohesion: 0.14
Nodes (14): FormulaNode, FormulaNodeProps, FormulaSlot, SlotProps, Commands, FormulaBlockAttrs, FormulaBlockNode, @tiptap/core (+6 more)

### Community 116 - "compilerOptions"
Cohesion: 0.11
Nodes (17): dist, ES2020, node_modules, src/**/*, compilerOptions, esModuleInterop, forceConsistentCasingInFileNames, lib (+9 more)

### Community 117 - "memory.ts"
Cohesion: 0.20
Nodes (12): IRecentTopic, IStudentMemory, studentMemorySchema, buildMemoryDigest(), buildMemoryPrompt(), clampStringList(), isMemoryEmpty(), memoryToJson() (+4 more)

### Community 118 - "FormulaCanvas.tsx"
Cohesion: 0.19
Nodes (14): createTemplateFromAction(), FormulaAttrs, FormulaCanvas(), inferInitialLatex(), TemplateInsert, formulaToLatex(), latexOf(), ActiveBlockEventDetail (+6 more)

### Community 119 - "formulaReducer.ts"
Cohesion: 0.30
Nodes (14): createFormulaNode(), createFormulaRow(), createId(), createInitialFormulaState(), deleteNodeById(), ensureRow(), formulaReducer(), insertIntoRow() (+6 more)

### Community 120 - "package.json"
Cohesion: 0.13
Nodes (14): author, bugs, url, description, homepage, keywords, license, main (+6 more)

### Community 121 - "user.controller.ts"
Cohesion: 0.23
Nodes (11): formatProfileResponse(), getMyProfile(), getUsers(), updateMyProfile(), restrictTo(), IUserData, UserData, userDataSchema (+3 more)

### Community 122 - "lessonContext.service.ts"
Cohesion: 0.21
Nodes (14): buildLessonContext(), cache, cacheInsertionOrder, DEFAULT_AGENT_SETTINGS, extractGuideAnswer(), getNodeTextContent(), LessonQuestion, normalizeAgentSettings() (+6 more)

### Community 123 - "TipTap Document as Single Source of Truth"
Cohesion: 0.15
Nodes (14): Five Load-Bearing Ideas, AI Writes via Preview then Accept, Server Owns Lesson Context, TipTap Document as Single Source of Truth, Auth and Session Flow, Lesson Lifecycle, Teacher Copilot Flow, Content Entity (+6 more)

### Community 124 - "User"
Cohesion: 0.34
Nodes (11): main(), VALID_ROLES, changePassword(), generateToken(), googleLogin(), login(), normalizeEmail(), recheckToken() (+3 more)

### Community 125 - "docs/ โ€” System Intuition Layer"
Cohesion: 0.37
Nodes (13): ARCHITECTURE โ€” System Mental Model, data-flow โ€” End-to-End Flows, data-model โ€” Nine Entities, GLOSSARY โ€” Codebase Vocabulary, AppLayout, creator (copilot) / creatorApi, ideas โ€” Architecture Decision Records, onboarding โ€” Clone to Working AI (+5 more)

### Community 126 - "tutor.test.ts"
Cohesion: 0.23
Nodes (10): ChatSession, chatSessionSchema, IChatMessage, IChatSession, capturedCall(), capturedContents(), capturedModel(), capturedSystemInstruction() (+2 more)

### Community 127 - "parse.ts"
Cohesion: 0.33
Nodes (9): cleanReply(), createSuggestionsGate(), INTERNAL_TAG_REGEX, parseTutorReply(), REPORT_LABEL_REGEX, REPORT_LABELS, stripInternalTags(), stripReportLabels() (+1 more)

### Community 128 - "AI Tutor Call Flow"
Cohesion: 0.20
Nodes (10): SSE Streaming Bypasses Axios, AI Tutor Call Flow, ChatSession Entity, LearningHistory Entity, UserContent Entity, (user, content) Join, ChatSession, clientThread (+2 more)

### Community 129 - "operations โ€” Deploy and Run"
Cohesion: 0.20
Nodes (10): ADR-014 Route Code Splitting, ADR-015 Deploy Last One Shot, ADR-017 app.ts / index.ts Boot Split, operations โ€” Deploy and Run, performance โ€” Speed Budget, Entry Bundle Size Budget, Self-Hosted Fonts, App/Entry Split Enables Testing (+2 more)

### Community 130 - "genaiClient.ts"
Cohesion: 0.42
Nodes (7): recordAiFailure(), recordAiSuccess(), callTutorModel(), callTutorModelStream(), TutorTurn, generateWithRetry(), isTransientAIError()

### Community 131 - "scripts"
Cohesion: 0.22
Nodes (9): scripts, build, check-models, dev, promote, start, test, test:ai (+1 more)

### Community 132 - "memory.test.ts"
Cohesion: 0.33
Nodes (7): capturedSystemInstruction(), GenerateContentArg, isMemoryCall(), memoryCalls(), { mockGenerateContent }, mockMemoryJson(), tutorCalls()

### Community 133 - "Sidebar.tsx"
Cohesion: 0.25
Nodes (7): FormulaSidebar, FormulaSidebarProps, InsertButton, InsertButtonProps, SidebarSection, SidebarSectionProps, PowerPosition

### Community 134 - "PublishAutofill.test.tsx"
Cohesion: 0.29
Nodes (6): DEFAULT_AGENT, META, { mockCallCreator }, render(), renderModal(), SUGGESTION

### Community 135 - "Golden Rule 2 โ€” Anonymous Full Access"
Cohesion: 0.32
Nodes (8): Golden Rule 2 โ€” Anonymous Full Access, optionalAuth, protect, ADR-004 optionalAuth on AI Routes, Onboarding Smoke Test (Both Auth States), Guard Matrix, Persistence Boundaries, ProtectedRoute Guard

### Community 136 - "numberedSectionHeadings.ts"
Cohesion: 0.43
Nodes (5): collectSectionHeadingFixes(), HeadingFix, NumberedSectionHeadingsExtension, numberedSectionHeadingsKey, stripLeadingSectionNumber()

### Community 137 - "chat.routes.ts"
Cohesion: 0.48
Nodes (5): clearMemory(), getMemory(), StudentMemory, chatGuards, router

### Community 139 - "playwright"
Cohesion: 0.50
Nodes (3): playwright, npx, @playwright/mcp

### Community 140 - "Golden Rule 1 โ€” Never Ration Tokens"
Cohesion: 0.50
Nodes (4): Golden Rule 1 โ€” Never Ration Tokens, Golden Rules, ADR-005 Never Ration Tokens, Bot-Only Rate Limiting

### Community 141 - "Render Cold Starts"
Cohesion: 0.50
Nodes (4): Render Cold Starts, ADR-006 In-Memory Observability, ADR-018 Two-Model Routing and Retry, Free-Tier Physics

### Community 142 - "structured 401"
Cohesion: 0.67
Nodes (3): Transport via axios, structured 401, ADR-011 Structured 401 Contract

## Knowledge Gaps
- **604 isolated node(s):** `$schema`, `style`, `rsc`, `tsx`, `config` (+599 more)
  These have ≤1 connection - possible missing edges or undocumented components.
- **68 thin communities (<3 nodes) omitted from report** — run `graphify query` to explore isolated nodes.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **Why does `useAppI18n()` connect `useAppI18n` to `cn`, `EditorRightSidebar.tsx`, `Sidebar.tsx`, `AiDraftDialog.tsx`, `AiQuestionDialog.tsx`, `Profile.tsx`, `QuestionChoiceView.tsx`, `AiWritingAssistant.tsx`, `Setting.tsx`, `QuestionBlankWriteView.tsx`, `ThemeToggle.tsx`, `tutorMemory.store.ts`, `i18n.ts`, `QuestionBlankChoiceView.tsx`, `language.store.ts`, `FabricCanvasView.tsx`, `editorExtensions.ts`, `QuestionWriteView.tsx`, `editor.i18n.ts`, `Explore.tsx`, `TopNav.tsx`, `Landing.tsx`, `History.tsx`, `EditorLeftSidebar.tsx`, `TipTapEditor.tsx`, `creatorApi.ts`, `types.ts`, `FormulaCanvas.tsx`?**
  _High betweenness centrality (0.098) - this node is a cross-community bridge._
- **Why does `RichLine` connect `RichLine` to `useFabric.ts`, `CanvasRightSidebar.tsx`?**
  _High betweenness centrality (0.019) - this node is a cross-community bridge._
- **Why does `useCanvasStore` connect `QuestionWriteView.tsx` to `cn`, `QuestionBlankChoiceView.tsx`, `PublishAutofill.test.tsx`, `editor.i18n.ts`, `AiDraftDialog.tsx`, `TipTapEditor.tsx`, `TiptapViewer.tsx`, `AiQuestionDialog.tsx`, `Profile.tsx`, `QuestionChoiceView.tsx`, `AiWritingAssistant.tsx`, `QuestionBlankWriteView.tsx`, `History.tsx`?**
  _High betweenness centrality (0.019) - this node is a cross-community bridge._
- **What connects `$schema`, `style`, `rsc` to the rest of the system?**
  _623 weakly-connected nodes found - possible documentation gaps or missing edges._
- **Should `cn` be split into smaller, more focused modules?**
  _Cohesion score 0.07922705314009662 - nodes in this community are weakly interconnected._
- **Should `creator.controller.ts` be split into smaller, more focused modules?**
  _Cohesion score 0.07565392354124749 - nodes in this community are weakly interconnected._
- **Should `app.ts` be split into smaller, more focused modules?**
  _Cohesion score 0.1408199643493761 - nodes in this community are weakly interconnected._