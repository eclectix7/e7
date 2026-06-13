---
title: Wxpi — Architecture
type: architecture-doc
status: design
tags:
  - wxpi
  - architecture
  - system-design
  - diagrams
---

# Wxpi — System Architecture

## 1. High-Level Topology

The Wxpi app uses a **local-first, self-improving architecture**: all user data stays on-device, an embedded AI assistant learns from interaction patterns, and the combined system progressively optimizes its own context quality over time.

```
┌─────────────────────────────────────────────────────────────┐
│                      Wxpi Expo App                         │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  UI Layer                           │   │
│  │   React Native  ·  Expo SDK  ·  User Interactions  │   │
│  └──────────────────────┬──────────────────────────────┘   │
│                         │                                   │
│  ┌──────────────────────▼──────────────────────────────┐   │
│  │              Context Orchestrator                    │   │
│  │                                                     │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │   │
│  │  │ Capture   │  │ Retrieve │  │ Self-Improvement │  │   │
│  │  │ Module    │──│ Module   │──│ Engine           │  │   │
│  │  └──────────┘  └──────────┘  └──────────────────┘  │   │
│  └──────────────────────┬──────────────────────────────┘   │
│                         │                                   │
│  ┌──────────────────────▼──────────────────────────────┐   │
│  │                Storage Layer                         │   │
│  │                                                     │   │
│  │  ┌──────────────┐  ┌────────────┐  ┌────────────┐  │   │
│  │  │ SQLite        │  │ Vector DB  │  │ Markdown   │  │   │
│  │  │ (structured   │  │ (semantic  │  │ (context   │  │   │
│  │  │  user data)   │  │  search)   │  │  notes)    │  │   │
│  │  └──────────────┘  └────────────┘  └────────────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                          │
                    ┌─────▼─────┐
                    │ OpenRouter │
                    │  (API)     │
                    │            │
                    │ · Embed    │
                    │ · Summarize│
                    │ · Reason   │
                    └───────────┘
```

## 2. Component Responsibilities

### 2.1 UI Layer (React Native / Expo)

- Renders user interface and captures **explicit** user signals (form inputs, preferences, corrections, feedback)
- Passes all user interaction events to the Context Orchestrator via a typed event bus
- Receives enriched context from the Orchestrator to personalize UI (e.g., suggested actions, remembered preferences)

### 2.2 Context Orchestrator

The brain of the system. Three sub-modules:

**Capture Module** — Receives raw user signals (explicit and implicit), classifies them, and routes them for storage.
- Explicit: user fills a form, sets a preference, corrects the assistant, gives feedback
- Implicit: dwell time, feature usage frequency, navigation patterns, time-of-day patterns, session length

**Retrieve Module** — When the assistant needs context about the user, this module:
1. Pulls structured data from SQLite
2. Runs semantic search against the vector store
3. Loads relevant markdown context notes
4. Assembles a compact "context envelope" injected into the LLM prompt

**Self-Improvement Engine** — Periodically (on a schedule or after N new signals), analyzes stored context to:
- Detect patterns in user behavior
- Identify redundant or conflicting context entries
- Generate summary/consolidation updates
- Suggest new context categories to track
- **Key: all self-improvements are logged to Markdown for human auditability**

### 2.3 Storage Layer

Three complementary stores, each serving a different purpose:

| Store | Technology | Purpose | Query Type |
|-------|-----------|---------|------------|
| **Structured** | SQLite (relational) | User profiles, preferences, settings, timestamps, metadata | SQL WHERE, JOIN, ORDER BY |
| **Semantic** | SQLite + vector extension (sqlite-vec or sqlite-vss) | Embedding vectors for all context entries | Cosine similarity, ANN search |
| **Narrative** | Markdown files on filesystem | Human-readable session summaries, self-improvement logs, context audit trail | File read, grep, Obsidian |

### 2.4 OpenRouter (External API)

Only two types of calls leave the device:

1. **Embeddings** — `text-embedding-3-small` (or similar) to convert user signals into vectors
2. **Optional LLM calls** — For context summarization, pattern detection, or generating self-improvement suggestions

All user data stays local. Only **derived representations** (embeddings, summaries) are generated via API.

## 3. Data Flow: User Interaction → Enriched Context

```
User Action (UI Event)
    │
    ▼
Capture Module
    │
    ├─► Normalize & classify signal
    │   (preference / behavior / feedback / correction / explicit_input)
    │
    ├─► Write structured data ──────────────────► SQLite (user_signals, preferences)
    │
    ├─► Generate embedding via OpenRouter ──────► SQLite vector store
    │
    └─► Append context note ───────────────────► Markdown (daily log)
              │
              ▼
Self-Improvement Engine (periodic)
    │
    ├─► Analyze recent signals for patterns
    │
    ├─► Update/consolidate preference records
    │
    ├─► Log improvements ──────────────────────► Markdown (improvement log)
    │
    └─► Re-embed updated context ──────────────► SQLite vector store
```

## 4. Data Flow: LLM Request with Enriched Context

```
User sends message / triggers AI action
    │
    ▼
Retrieve Module
    │
    ├─► Pull user profile & preferences ──────────────► SQLite
    │
    ├─► Semantic search for relevant context ─────────► Vector store
    │   (query embedding → top-K similar context entries)
    │
    ├─► Load recent markdown summary ────────────────► Markdown files
    │
    └─► Assemble context envelope ──────────────────► Compacted prompt
              │
              ▼
OpenRouter LLM call (with context-enriched prompt)
    │
    ▼
Response to user (personalized, context-aware)
    │
    ▼
Capture Module (response metadata stored for improvement cycle)
```

## 5. Self-Improvement Feedback Loop

This is the key differentiator — the system orchestrates its own improvement:

```
┌──────────────────────────────────────────────────┐
│              Self-Improvement Cycle               │
│                                                    │
│  1. TRIGGER                                        │
│     · Every N new signals (e.g., 50)               │
│     · Every M hours of usage (e.g., 24h)           │
│     · Explicit user request ("improve my context") │
│                    │                               │
│  2. ANALYZE                                        │
│     · Read last N signals from SQLite               │
│     · Detect frequent patterns (habit extraction)  │
│     · Identify conflicts (contradictory signals)   │
│     · Measure context coverage gaps                │
│                    │                               │
│  3. PLAN                                          │
│     · Generate improvement proposal                │
│     · Example: "User opens app most at 8am on     │
│       weekdays → add time_preference tag"          │
│                    │                               │
│  4. PROPOSE (HITL — Human In The Loop)            │
│     · Log proposal to Markdown improvement log     │
│     · Notify user: "I noticed X, should I use     │
│       this to personalize Y?"                      │
│     · Await user approval or auto-apply if config  │
│       allows                                       │
│                    │                               │
│  5. APPLY                                          │
│     · Update preference records in SQLite          │
│     · Re-affected embeddings in vector store       │
│     · Log the change to improvement audit trail    │
│                    │                               │
│  6. VERIFY                                         │
│     · Monitor subsequent interactions              │
│     · Check if personalization improved            │
│     · If not, flag for review (don't auto-revert)  │
│                                                    │
└──────────────────────────────────────────────────┘
```

## 6. Module Interaction Diagram

```
                    ┌─────────────┐
                    │   User       │
                    └──────┬──────┘
                           │ interacts
                           ▼
┌──────────────────────────────────────────────────┐
│  UI Layer (React Native)                         │
│  ┌──────────────────────────────────────┐        │
│  │ Event Bus (typed events)             │        │
│  └──────────┬───────────────────────────┘        │
└─────────────┼────────────────────────────────────┘
              │ emits signal
              ▼
┌──────────────────────────────────────────────┐
│  Capture Module                              │
│  ┌─────────────┐  ┌──────────────────────┐   │
│  │ Classifier   │  │ Signal Normalizer    │   │
│  └──────┬──────┘  └──────────┬───────────┘   │
│         └──────────┬─────────┘                │
│                    │                          │
│         ┌──────────▼──────────┐               │
│         │ Storage Router      │               │
│         └──┬─────┬─────┬──────┘               │
│            │     │     │                      │
└────────────┼─────┼─────┼──────────────────────┘
             │     │     │
     ┌───────▼──┐  │  ┌──▼─────────────┐
     │ SQLite    │  │  │ Markdown        │
     │ (struct)  │  │  │ (narrative)     │
     └───────────┘  │  └─────────────────┘
                    │
             ┌──────▼───────┐
             │ Vector Store  │◄── OpenRouter
             │ (embeddings)  │   (embed API)
             └──────────────┘
                    ▲
                    │ similarity search
             ┌──────┴───────┐
             │ Retrieve     │
             │ Module       │──► Context Envelope
             └──────────────┘    (injected into
                    ▲              LLM prompt)
                    │
             ┌──────┴───────┐
             │ Self-        │
             │ Improvement  │
             │ Engine       │
              └──────────────┘
                    │
                    ▼
             ┌──────────────┐     ┌───────────┐
             │ Improvement  │────►│ User       │
             │ Proposal     │     │ (HITL)     │
             └──────────────┘     └───────────┘
```

## 7. Expo Integration Notes

- All database access via `expo-sqlite` (built-in, no native modules needed)
- Vector extension requires `expo-sqlite` with custom native build or a JS-side brute-force cosine similarity (viable for <10K vectors)
- Markdown files stored in app's document directory via `expo-file-system`
- OpenRouter calls via standard `fetch` (no special SDK needed)
- Background self-improvement via `expo-task-manager` (periodic sync) — triggers when app is in background or on next launch if offline

## 8. Schema Relationships

```
user_profile (1) ──── (N) user_signals
       │
       ├──── (N) preferences
       │
       ├──── (N) context_entries  ──── (1) embeddings
       │
       └──── (N) self_improvement_log

markdown_files:
  /context/YYYY-MM-DD.md          (daily signal log)
  /improvements/YYYY-MM-DD.md     (improvement proposals & audit)
  /summary/weekly-summaries.md    (auto-generated user context summaries)
```

## Related

- [[storage-schema|Storage Schema]] — detailed table definitions and vector config
- [[context-pipeline|Context Pipeline]] — capture, classify, embed, store in detail
- [[retrieval-strategy|Retrieval Strategy]] — how context is fetched and assembled
- [[self-improvement-engine|Self-Improvement Engine]] — the feedback loop in depth
- [[openrouter-integration|OpenRouter Integration]] — API patterns and cost management
