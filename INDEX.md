---
title: E7 Vault — Index
type: index
tags: [index, navigation]
---

# E7 Vault — Table of Contents

## Projects

### E7 Central Hub
Unified project management platform — LAMP web dashboard + Expoe mobile app. Hybrid LAMP/Convex backend.

| Doc | Description |
|-----|-------------|
| [[ROADMAP.md\|ROADMAP]] | Strategic context, open questions, project status |
| [[ARCHITECTURE.md\|ARCHITECTURE]] | System design, data model, security, implementation patterns |
| `e7-central-hub-openapi.yaml` | Full OpenAPI 3.1 spec (44 ops, 49 schemas, 4 groups) |

**API Groups:** App State `/state` (8) · Todos `/todos` (7) · Backlog `/backlog` (11) · Kanban `/boards` (18)

---

### Wxpi
Independent Expo app — local-first context memory system. SQLite + vector embeddings + markdown + OpenRouter.

| Doc | Description |
|-----|-------------|
| [[wxpi/README.md\|README]] | Project index, architecture overview, open questions |
| [[wxpi/architecture.md\|Architecture]] | System topology, data flow, module interactions |
| [[wxpi/storage-schema.md\|Storage Schema]] | SQLite tables, vector config, markdown conventions |
| [[wxpi/context-pipeline.md\|Context Pipeline]] | Capture → normalize → classify → store → embed → log |
| [[wxpi/retrieval-strategy.md\|Retrieval Strategy]] | Context envelope assembly, token budgets, caching |
| [[wxpi/self-improvement-engine.md\|Self-Improvement Engine]] | Analyzer, planner, HITL gate, executor, verifier |
| [[wxpi/openrouter-integration.md\|OpenRouter Integration]] | Embedding + chat API, cost management, fallbacks |
| [[wxpi/implementation-roadmap.md\|Implementation Roadmap]] | 6 phases, 12 weeks, dependency graph |

**Cursor Prompts:** `wxpi/prompts/` — 15 implementation prompts (Phases 1-6)

---

### MacBook & iOS Deployment
Setup guides for the new MacBook Pro (M5 Max) and iOS app deployment pipeline.

| Doc | Description |
|-----|-------------|
| [[macbook-setup/iOS-Deployment-Checklist.md\|iOS Checklist]] | Phased setup: prerequisites → MacBook → Xcode → TestFlight → App Review |
| [[macbook-setup/Hermes-Secure-Install-Proposal.md\|Hermes Security]] | Agent security architecture, account separation, credential rules |

---

## Meta

| Doc | Description |
|-----|-------------|
| [[meta/Hermes About Me, 1st Chat.md\|First Chat]] | Initial context: who Kaizen is, E7 goals, working style |
| [[meta/README.md\|Meta README]] | Meta directory description |
| [[Prompts.md\|Prompts]] | Original task prompts and TODOs |

---

## Quick Reference

**Tech Stack:** LAMP · Convex · Expo SDK 52-55 · React Native · TypeScript · SQLite · OpenRouter

**Security:** JWT Bearer (mobile) · API Key (web) · No Clerk · Modular auth preferred

**Active Skills:** `e7-api-designer` · `e7-strategic-roadmap` · `hermes-cli` · `context-session-digest` · `skill-curator-pro`

**Open Questions:** User management API · Tags strategy · File attachments · WebSocket/SSE · Rate limiting · API versioning
