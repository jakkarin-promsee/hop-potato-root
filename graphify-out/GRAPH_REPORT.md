# Graph Report - c:\Users\BTCOM\Desktop\0_Project\Hot-Potato  (2026-07-12)

## Corpus Check
- 31 files · ~111,137 words
- Verdict: corpus is large enough that graph structure adds value.

## Summary
- 217 nodes · 280 edges · 35 communities (12 shown, 23 thin omitted)
- Extraction: 89% EXTRACTED · 11% INFERRED · 0% AMBIGUOUS · INFERRED: 32 edges (avg confidence: 0.85)
- Token cost: 0 input · 0 output

## Community Hubs (Navigation)
- Launch Roadmap Tiers
- 390px Lesson Feedback
- Ask AI Modal UI
- Docs Hub Overview
- Archive Roadmaps
- Architecture Data Flows
- Platform Core Concepts
- Golden Rules Security
- Guide Tutorial Track
- Playwright MCP
- Render Cold Starts
- Axios 401 Contract
- Tech Debt Notes
- Authoring Learning Planes
- Request Lifecycle
- Student Memory
- ADR Memory Personas
- Student Answers Flow
- Mongoose Models
- User Entity
- Access Type Glossary
- Agent Settings
- Block ID Glossary
- Feedback Mode
- Tutor Mode Glossary
- ADR-001
- ADR-010
- ADR-012
- ADR-013
- ADR-016
- ADR-019
- Env Matrix
- Intuition Layer
- Public Route
- Require Login Route

## God Nodes (most connected - your core abstractions)
1. `docs/ โ€” System Intuition Layer` - 12 edges
2. `Golden Rule 2: Anonymous Full Access` - 11 edges
3. `archive/arc2/ROADMAP.md Universal` - 9 edges
4. `ARCHITECTURE โ€” System Mental Model` - 9 edges
5. `plan/guide.md` - 8 edges
6. `Ask AI modal dialog (centered overlay on dimmed lesson page)` - 7 edges
7. `Inline contextual chat panel embedded in lesson scroll` - 7 edges
8. `plan/tier3.md` - 7 edges
9. `data-flow โ€” End-to-End Flows` - 7 edges
10. `ideas โ€” Architecture Decision Records` - 7 edges

## Surprising Connections (you probably didn't know these)
- `graphify-out Knowledge Graph` --semantically_similar_to--> `Documentation Intuition Layer`  [INFERRED] [semantically similar]
  AGENT.md → archive/arc3/ROADMAP-docs.md
- `Golden Rule 2: Anonymous Full Access` --rationale_for--> `optionalAuth Middleware`  [INFERRED]
  AGENT.md → archive/arc3/ROADMAP.md
- `Ask AI button on question card (robot icon, bottom-right)` --semantically_similar_to--> `Ask AI modal dialog (centered overlay on dimmed lesson page)`  [INFERRED] [semantically similar]
  archive/arc2/plan/tier0-audit/audit-390-thread.jpeg → archive/arc2/plan/tier0-audit/audit-390-modal.jpeg
- `plan/guide.md` --references--> `Golden Rule 2: Anonymous Full Access`  [EXTRACTED]
  archive/arc3/plan/guide.md → AGENT.md
- `Hot Potato` --conceptually_related_to--> `Client (React/Vite)`  [INFERRED]
  README.md → CLAUDE.md

## Import Cycles
- None detected.

## Hyperedges (group relationships)
- **Non-Negotiable Golden Rules** — golden_rule_never_ration_tokens, golden_rule_anonymous_full_access, optional_auth [EXTRACTED 1.00]
- **Three-Repo Workspace** — three_repo_workspace, client_half, server_half [EXTRACTED 1.00]
- **Guide Screenshot Pipeline** — seed_guide_demo_mjs, capture_guide_mjs, learning_showcase, creating_showcase [EXTRACTED 1.00]
- **Five Load-Bearing Ideas** — docs_architecture_tiptap_single_source_of_truth, docs_architecture_server_owns_lesson_context, docs_architecture_golden_rule_2, docs_architecture_golden_rule_1, docs_architecture_preview_accept_copilot [EXTRACTED 1.00]
- **(user, content) Join Models** — docs_data_model_user_content, docs_data_model_learning_history, docs_data_model_chat_session [EXTRACTED 1.00]
- **Three Route Guards** — docs_ui_structure_protected_route, docs_ui_structure_require_login, docs_ui_structure_public_route [EXTRACTED 1.00]
- **Write-question AI feedback flow at 390px** — archive_arc2_plan_tier0_audit_audit_390_feedback_red_sedan_question, archive_arc2_plan_tier0_audit_audit_390_feedback_student_answer_text, archive_arc2_plan_tier0_audit_audit_390_feedback_ai_deep_evaluation_panel, archive_arc2_plan_tier0_audit_audit_390_feedback_stationary_praise, archive_arc2_plan_tier0_audit_audit_390_feedback_permanence_nuance [EXTRACTED 1.00]
- **Ask AI modal vertical layout stack** — archive_arc2_plan_tier0_audit_audit_390_modal_modal_header, archive_arc2_plan_tier0_audit_audit_390_modal_chat_history_area, archive_arc2_plan_tier0_audit_audit_390_modal_suggestion_chips_row, archive_arc2_plan_tier0_audit_audit_390_modal_textarea_input, archive_arc2_plan_tier0_audit_audit_390_modal_ask_send_button [EXTRACTED 1.00]
- **Lesson content hierarchy: title → section → example → interactive questions** — archive_arc2_plan_tier0_audit_audit_390_top_lesson_initial_state, archive_arc2_plan_tier0_audit_audit_390_feedback_lesson_title, archive_arc2_plan_tier0_audit_audit_390_feedback_reference_point_section, archive_arc2_plan_tier0_audit_audit_390_feedback_example_callout_box, archive_arc2_plan_tier0_audit_audit_390_top_write_question_empty [INFERRED 0.85]

## Communities (35 total, 23 thin omitted)

### Community 0 - "Launch Roadmap Tiers"
Cohesion: 0.15
Nodes (25): appearance.store (Font Size), plan/tier0.md, Tier 0: SEO and Link Previews, plan/tier1.md, Tier 1: Observability, plan/tier2.md, Tier 2: Bundle Split and Fonts, plan/tier3.5.md (+17 more)

### Community 1 - "390px Lesson Feedback"
Cohesion: 0.10
Nodes (23): AI DEEP EVALUATION feedback panel (light-purple card), App header bar (Intuita logo, Log in, dark mode, hamburger menu), Electric pole reference-point example (5 meters to the right), Light-purple bordered example callout in lesson body, Lesson title: การระบุตำแหน่งของวัตถุ (Identifying Object Position), Mobile lesson viewer page (TipTapViewer), 390px phone-first viewport audit (Tier 0 Phase 0.C C3), Write question: Can a red sedan parked roadside be a reference point? (+15 more)

### Community 2 - "Ask AI Modal UI"
Cohesion: 0.09
Nodes (23): AI nuance: reference point should be permanent/reliable, car may drive away, AI message bubble (white, left-aligned), Ask AI modal dialog (centered overlay on dimmed lesson page), Ask send button (paper plane icon, light purple), Chat history scroll area with alternating user/AI bubbles, Chip: Common reference points in daily life, Chip: What if the reference point can move?, Chip: Must a reference point be a real object? (+15 more)

### Community 3 - "Docs Hub Overview"
Cohesion: 0.16
Nodes (23): ARCHITECTURE โ€” System Mental Model, data-flow โ€” End-to-End Flows, data-model โ€” Nine Entities, GLOSSARY โ€” Codebase Vocabulary, AppLayout, creator (copilot) / creatorApi, ideas โ€” Architecture Decision Records, ADR-014 Route Code Splitting (+15 more)

### Community 4 - "Archive Roadmaps"
Cohesion: 0.10
Nodes (20): Phase -1 Test Harness, POST /api/chat/tutor (Arc1 Phase 1), archive/arc1/ROADMAP-detailed.md, archive/arc1/ROADMAP.md, Arc1 AI Tutor Rework Roadmap, Tier 0.A Client Rewire, archive/arc2/plan/tier0.md, archive/arc2/plan/tier1.md (+12 more)

### Community 5 - "Architecture Data Flows"
Cohesion: 0.11
Nodes (20): Server Owns Lesson Context, SSE Streaming Bypasses Axios, TipTap Document as Single Source of Truth, AI Tutor Call Flow, Auth and Session Flow, Lesson Lifecycle, ChatSession Entity, Content Entity (+12 more)

### Community 6 - "Platform Core Concepts"
Cohesion: 0.21
Nodes (13): asking-flow.md Quality Bar, Client (React/Vite), Documentation Intuition Layer, FEEDBACK_TASK_TAG Evaluation Gate, graphify-out Knowledge Graph, Hot Potato, persona.ts (buildSystemInstruction), POST /api/chat/tutor (+5 more)

### Community 7 - "Golden Rules Security"
Cohesion: 0.15
Nodes (16): Five Load-Bearing Ideas, Golden Rule 1 โ€” Never Ration Tokens, Golden Rule 2 โ€” Anonymous Full Access, AI Writes via Preview then Accept, Teacher Copilot Flow, Golden Rules, optionalAuth, protect (+8 more)

### Community 8 - "Guide Tutorial Track"
Cohesion: 0.47
Nodes (7): plan/guide.md, BRAND_NAME (brand.ts), capture-guide.mjs, creating-showcase (/guide/creating), Guide Tutorial Track (G0-G6), learning-showcase (/guide/learning), seed-guide-demo.mjs

### Community 9 - "Playwright MCP"
Cohesion: 0.50
Nodes (3): playwright, npx, @playwright/mcp

### Community 10 - "Render Cold Starts"
Cohesion: 0.50
Nodes (4): Render Cold Starts, ADR-006 In-Memory Observability, ADR-018 Two-Model Routing and Retry, Free-Tier Physics

### Community 11 - "Axios 401 Contract"
Cohesion: 0.67
Nodes (3): Transport via axios, structured 401, ADR-011 Structured 401 Contract

## Knowledge Gaps
- **71 isolated node(s):** `npx`, `@playwright/mcp`, `archive/arc2/plan/tier2.md`, `Arc2 Tier 0 — Conversation Core`, `Arc2 Tier 1 — Tutor Experience` (+66 more)
  These have ≤1 connection - possible missing edges or undocumented components.
- **23 thin communities (<3 nodes) omitted from report** — run `graphify query` to explore isolated nodes.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **Why does `ARCHITECTURE โ€” System Mental Model` connect `Docs Hub Overview` to `Architecture Data Flows`, `Golden Rules Security`?**
  _High betweenness centrality (0.023) - this node is a cross-community bridge._
- **Why does `TipTap Document as Single Source of Truth` connect `Architecture Data Flows` to `Docs Hub Overview`, `Golden Rules Security`?**
  _High betweenness centrality (0.020) - this node is a cross-community bridge._
- **What connects `npx`, `@playwright/mcp`, `archive/arc2/plan/tier2.md` to the rest of the system?**
  _90 weakly-connected nodes found - possible documentation gaps or missing edges._
- **Should `Launch Roadmap Tiers` be split into smaller, more focused modules?**
  _Cohesion score 0.14814814814814814 - nodes in this community are weakly interconnected._
- **Should `390px Lesson Feedback` be split into smaller, more focused modules?**
  _Cohesion score 0.09881422924901186 - nodes in this community are weakly interconnected._
- **Should `Ask AI Modal UI` be split into smaller, more focused modules?**
  _Cohesion score 0.09486166007905138 - nodes in this community are weakly interconnected._
- **Should `Archive Roadmaps` be split into smaller, more focused modules?**
  _Cohesion score 0.1 - nodes in this community are weakly interconnected._