---
title: Wxpi — Implementation Roadmap
type: roadmap
status: design
tags:
  - wxpi
  - roadmap
  - implementation
  - phases
---

# Wxpi — Implementation Roadmap

Phased build order for the context memory system. Each phase delivers a working increment.

---

## Phase 1: Foundation (Week 1-2)

**Goal:** SQLite schema + basic signal capture working.

### Tasks
1. Set up `expo-sqlite` and create all tables (user_profile, user_signals, preferences, context_entries, self_improvement_log)
2. Set up `expo-file-system` and markdown directory structure
3. Build typed event bus for signal capture
4. Implement Capture Module (Stage 1-3: capture, normalize, classify)
5. Implement basic SQLite storage (Stage 4)
6. Implement markdown daily log append (Stage 6)
7. Write unit tests for signal normalization and classification

### Milestone
- ✅ User interactions are captured and stored in SQLite
- ✅ Daily markdown log is being written
- ✅ No external API calls yet

### Cursor Prompts
- [[prompts/phase1-sqlite-schema|Phase 1: SQLite Schema]]
- [[prompts/phase1-capture-module|Phase 1: Capture Module]]
- [[prompts/phase1-markdown-logging|Phase 1: Markdown Logging]]

---

## Phase 2: Embeddings (Week 3-4)

**Goal:** Vector storage + semantic search working.

### Tasks
1. Set up `expo-secure-store` for OpenRouter API key
2. Implement OpenRouter embedding API client
3. Implement embedding storage (choose: sqlite-vec or JS brute force)
4. Implement semantic search function
5. Update Capture Module to embed explicit_input and correction signals
6. Add embedding cache
7. Write tests for embedding pipeline

### Milestone
- ✅ Explicit inputs are embedded and stored as vectors
- ✅ Semantic search returns relevant context entries
- ✅ Embedding cache reduces redundant API calls

### Cursor Prompts
- [[prompts/phase2-embedding-client|Phase 2: Embedding Client]]
- [[prompts/phase2-vector-search|Phase 2: Vector Search]]
- [[prompts/phase2-embed-pipeline|Phase 2: Embedding Pipeline Integration]]

---

## Phase 3: Retrieval (Week 5-6)

**Goal:** Context envelope assembly + LLM integration working.

### Tasks
1. Implement Retrieve Module (profile pull, semantic search, recent context)
2. Implement context envelope assembly with token budget management
3. Implement OpenRouter chat completions client
4. Build context-aware chat function
5. Add profile cache and embedding cache
6. Implement fallback behavior for each retrieval phase
7. Write tests for envelope assembly

### Milestone
- ✅ LLM responses are personalized with user context
- ✅ Context envelope respects token budget
- ✅ System degrades gracefully when components fail

### Cursor Prompts
- [[prompts/phase3-retrieve-module|Phase 3: Retrieve Module]]
- [[prompts/phase3-context-envelope|Phase 3: Context Envelope]]
- [[prompts/phase3-chat-integration|Phase 3: Chat Integration]]

---

## Phase 4: Self-Improvement Engine (Week 7-9)

**Goal:** Automated pattern detection + HITL approval flow working.

### Tasks
1. Implement trigger scheduler (signal threshold + time-based)
2. Implement Analyzer Module (pattern detection, conflict detection, gap detection)
3. Implement Action Planner (action types, confidence scoring, auto-apply logic)
4. Implement HITL Gate (in-app proposal UI, user approval flow)
5. Implement Executor Module (apply changes to SQLite, re-embed, update markdown)
6. Implement Verifier Module (measure improvement impact)
7. Implement audit logging (SQLite + markdown)
8. Add HITL configuration (always_ask / high_confidence_auto / full_auto)
9. Write tests for each engine module

### Milestone
- ✅ System detects patterns from user behavior
- ✅ Improvement proposals are shown to user for approval
- ✅ Applied changes are logged and auditable
- ✅ System measures whether improvements actually helped

### Cursor Prompts
- [[prompts/phase4-analyzer|Phase 4: Analyzer Module]]
- [[prompts/phase4-planner|Phase 4: Action Planner]]
- [[prompts/phase4-hitl-gate|Phase 4: HITL Gate]]
- [[prompts/phase4-executor|Phase 4: Executor & Verifier]]

---

## Phase 5: Polish & Optimization (Week 10-11)

**Goal:** Performance, edge cases, and UX refinement.

### Tasks
1. Performance optimization (query indexing, caching, batch operations)
2. Error handling hardening (retry logic, offline queue, graceful degradation)
3. Background task registration (expo-background-fetch for self-improvement)
4. Cost management (rate limiting, budget caps, usage dashboard)
5. User-facing context dashboard (view/edit preferences, see improvement log)
6. Markdown summary viewer (weekly summaries, improvement history)
7. End-to-end testing

### Milestone
- ✅ System runs reliably in production
- ✅ User can view and manage their context
- ✅ Costs are controlled and predictable
- ✅ Background self-improvement works

### Cursor Prompts
- [[prompts/phase5-performance|Phase 5: Performance Optimization]]
- [[prompts/phase5-error-handling|Phase 5: Error Handling]]
- [[prompts/phase5-user-dashboard|Phase 5: User Context Dashboard]]

---

## Phase 6: Advanced Features (Week 12+)

**Goal:** Stretch features for enhanced self-improvement.

### Tasks
1. LLM-powered pattern analysis (replace rule-based analyzer with LLM calls)
2. Weekly summary generation via LLM
3. Context conflict auto-resolution (with HITL override)
4. Multi-signal batch embedding for efficiency
5. Context decay (reduce importance of old, unaccessed entries)
6. Export/import context data (for user data portability)

### Cursor Prompts
- [[prompts/phase6-llm-analysis|Phase 6: LLM-Powered Analysis]]
- [[prompts/phase6-advanced-features|Phase 6: Advanced Features]]

---

## Dependency Graph

```
Phase 1 (Foundation)
    │
    ├──▶ Phase 2 (Embeddings) ──┐
    │                            │
    ├──▶ Phase 3 (Retrieval) ───┤
    │         │                  │
    │         ▼                  │
    │   Phase 4 (Self-Improvement)
    │         │
    ▼         ▼
Phase 5 (Polish) ──▶ Phase 6 (Advanced)
```

Phases 2 and 3 can be worked in parallel after Phase 1. Phase 4 depends on both. Phase 5 depends on Phase 4. Phase 6 is optional.

---

## Related

- [[architecture|Architecture]] — system design
- [[storage-schema|Storage Schema]] — table definitions
- [[context-pipeline|Context Pipeline]] — capture pipeline
- [[retrieval-strategy|Retrieval Strategy]] — retrieval patterns
- [[self-improvement-engine|Self-Improvement Engine]] — engine design
- [[openrouter-integration|OpenRouter Integration]] — API details
- [[prompts/|Cursor Prompts]] — implementation prompts
