---
title: Wxpi — Context Memory System
type: project-index
status: design
stakeholders:
  - Kaizen (reviewer)
tags:
  - wxpi
  - expo
  - sqlite
  - embeddings
  - memory-system
  - ai-context
---

# Wxpi — User Context Memory System

## Purpose

Wxpi needs to **automatically capture, store, and progressively enrich user context** over time so that AI interactions become increasingly personalized and relevant — without requiring users to repeatedly state their preferences, history, or intent.

This is a design-level specification. Implementation should be driven by the Cursor prompts in `prompts/`, which reference these architecture docs.

## Architecture Overview

```
┌──────────────────────────────────────────────────┐
│                  Wxpi Expo App                    │
│                                                   │
│  ┌─────────────┐    ┌──────────────────────────┐ │
│  │  UI Layer    │    │  Context Pipeline        │ │
│  │  (React      │───▶│                          │ │
│  │   Native)    │    │  1. Capture (implicit +  │ │
│  └─────────────┘    │     explicit signals)     │ │
│         │            │  2. Normalize & classify │ │
│         │            │  3. Embed (OpenRouter    │ │
│         │            │     embedding API)        │ │
│         │            │  4. Store (SQLite +       │ │
│         │            │     vector extension)     │ │
│         │            │  5. Retrieve (semantic   │ │
│         │            │     search on context)    │ │
│         │            └──────────┬───────────────-┘ │
│         │                       │                  │
│  ┌──────▼───────────────────────▼───────────────┐ │
│  │           Storage Layer                       │ │
│  │                                               │ │
│  │  ┌───────────────┐  ┌────────────────────┐   │ │
│  │  │  SQLite         │  │  Markdown Files     │   │ │
│  │  │  (structured    │  │  (human-readable    │   │ │
│  │  │   user data,    │  │   context notes,    │   │ │
│  │  │   embeddings,   │  │   session logs,     │   │ │
│  │  │   metadata)     │  │   summaries)        │   │ │
│  │  └───────────────┘  └────────────────────┘   │ │
│  └───────────────────────────────────────────────┘ │
│                                                    │
│  ┌───────────────────────────────────────────────┐ │
│  │  OpenRouter API                                │ │
│  │  - Embedding model (text-embedding-3-small)    │ │
│  │  - LLM for context summarization (optional)    │ │
│  └───────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────┘
```

## Design Principles

1. **Progressive enrichment** — Context improves over time, never degrades. New data augments; it doesn't replace unless explicitly corrected.
2. **Implicit-first capture** — Most context is gathered passively from behavior, not by asking users questions.
3. **Dual storage** — SQLite for structured/queryable data, Markdown for human-readable context and audit trails.
4. **Local-first** — All data stays on-device. OpenRouter is called only for embeddings and optional summarization.
5. **Separation of concerns** — Capture, storage, and retrieval are independent modules. Each can be tested and iterated on separately.

## Document Map

| Document | Purpose |
|----------|---------|
| [[architecture|Architecture]] | Full system design, data flow, component specs |
| [[storage-schema|Storage Schema]] | SQLite tables, vector config, markdown file conventions |
| [[context-pipeline|Context Pipeline]] | How user signals are captured, classified, embedded, stored |
| [[retrieval-strategy|Retrieval Strategy]] | How context is fetched and injected into LLM prompts |
| [[openrouter-integration|OpenRouter Integration]] | API patterns, model selection, cost management |
| [[implementation-roadmap|Implementation Roadmap]] | Phased build order with milestones |

## Key Open Questions

- Should context summarization run on-device (small local model) or via OpenRouter?
- What's the right embedding refresh strategy — re-embed on every change, or batch?
- How to handle context conflicts when user behavior contradicts stored preferences?
- Retention policy — does context expire or decay over time?

## Related

- [[E7 Central Hub — Architecture & Design Specification|E7 Central Hub Architecture]] — similar dual-storage pattern
- [[prompts/|Cursor Prompts]] — implementation prompts for each module
