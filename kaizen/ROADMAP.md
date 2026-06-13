# E7 Strategic Roadmap & Context

## High-Level Profile: Todd (Kaizen)
- Senior Full Stack Architect & Developer.
- Experience: LAMP, Node, React, jQuery, Expo.
- History: 100+ applications built over 20 years.
- Preferences: OO languages, Design Patterns, SOLID, GRASP.
- Startup: E7 LLC (Phase 1, no revenue).
- Current Tech: Expo (SDK 52-55), Convex (Backend/API Proxy), LAMP (Website Hub).
- Goals: Create multiple revenue streams for apps solving real-world problems, foster learning/job opportunities, offer assistance.

## Long-Term Vision: E7 Central Hub
- A unified management platform with dual views:
    - Web-based LAMP Dashboard.
    - Mobile App.
- Core Functionality: State management, todo lists, feature/bug/backlog tracking via internal Kanban, feature planning.
- Backend Strategy: Hybrid LAMP (browser)/Convex.

## Operational Constraints
- Device dedicated to Hermes (Kaizen).
- Security: Repos/development performed on other secure systems.
- Authentification: Avoid Clerk (Google credential issues); modular authentication desired.
- Status: Long-term strategic development pending project initiation (target phase: upcoming weeks/post-MacBook arrival).

## E7 Central Hub — Architecture & API Design (COMPLETED 2026-05-24)

The architecture and API specification for the E7 Central Hub is complete:

- **Architecture Document:** `~/e7/kaizen/ARCHITECTURE.md` — System design, data model, security, deployment, and implementation phasing
- **API Specification:** `~/e7/kaizen/e7-central-hub-openapi.yaml` — Complete OpenAPI 3.1 spec (44 operations, 49 schemas, 4 API groups)
- **Status:** Design phase complete. Ready for Phase 1 implementation (foundation: schema + scaffolding).

### API Groups
| Group | Prefix | Operations |
|-------|--------|-----------|
| App State | `/state` | 8 |
| Todos | `/todos` | 7 |
| Backlog | `/backlog` | 11 |
| Kanban | `/boards` | 18 |

### Open Questions (HITL Needed)
1. User management endpoints not yet defined
2. Tags: JSON arrays vs normalized many-to-many
3. File attachment strategy
4. WebSocket/SSE for real-time web dashboard
5. Rate limiting strategy
6. API versioning strategy for breaking changes

## Persistence
- Contextual details stored here are cold-stored until requested.
- Related documentation is located in: ~/e7/kaizen/
