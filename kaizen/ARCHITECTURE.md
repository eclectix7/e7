# E7 Central Hub — Architecture & Design Specification

**Version:** 1.0.0-dev
**Status:** Design Complete — Ready for Implementation Planning
**Last Updated:** 2026-05-24

---

## 1. Executive Summary

The E7 Central Hub is a unified project management platform built around a **dual-view architecture**: a LAMP-based web dashboard for browser administration and an Expo (React Native) mobile app for iOS/Android. Both views consume a single, shared REST API defined in the accompanying OpenAPI 3.1 specification (`e7-central-hub-openapi.yaml`).

The backend follows a **hybrid LAMP + Convex** strategy: LAMP (PHP/MySQL/Apache) serves the web dashboard with traditional REST endpoints, while Convex powers real-time subscriptions for the mobile client and acts as the API proxy layer.

### Key Numbers

| Metric | Value |
|--------|-------|
| API Groups | 4 (App State, Todos, Backlog, Kanban) |
| Total REST Operations | 44 |
| Total API Paths | 22 |
| JSON Schemas | 49 |
| Request Schemas | 15 |
| Entity Schemas | 19 |
| Enum Schemas | 5 |
| Common/Utility Schemas | 6 |
| Security Schemes | 2 (JWT Bearer, API Key) |
| Server Environments | 4 (2 prod, 2 local dev) |

---

## 2. System Architecture

### 2.1 High-Level Topology

```
+------------------+          +------------------+
|   LAMP Web       |          |   Expo Mobile    |
|   Dashboard      |          |   App (iOS/And)  |
|   (PHP/MySQL)    |          |   (React Native) |
+--------+---------+          +--------+---------+
         |                             |
         |  HTTPS (REST)               |  HTTPS (REST)
         |  + JWT Bearer               |  + API Key
         v                             v
+--------+-----------------------------+---------+
|                                                  |
|              API Gateway / Router                |
|                                                  |
|  +------------------+   +---------------------+  |
|  |  LAMP REST Layer |   |  Convex Backend     |  |
|  |  (PHP/Apache)    |   |  (Real-time DB +    |  |
|  |                  |   |   Auth Pipeline)    |  |
|  +--------+---------+   +----------+----------+  |
|           |                        |             |
|           v                        v             |
|  +--------+-------------------------+--------+   |
|  |           MySQL Database Layer             |   |
|  +-------------------------------------------+   |
+--------------------------------------------------+
```

### 2.2 Dual-View Design

**LAMP Web Dashboard**
- Stack: PHP 8.x / MySQL 8.x / Apache 2.4
- Authentication: API Key (`X-API-Key` header)
- Role: Primary admin interface, full CRUD access
- Rendering: Server-side HTML with progressive JS enhancement
- API consumption: Direct REST calls to LAMP REST layer

**Expo Mobile App**
- Stack: React Native (Expo SDK 52-55) / TypeScript
- Authentication: JWT Bearer tokens via Convex auth pipeline
- Role: On-the-go task management, triage submission, kanban viewing
- Rendering: Native UI components via React Native
- API consumption: REST via Convex proxy + real-time subscriptions

### 2.3 Hybrid Backend Strategy

The backend is intentionally split to leverage the strengths of each platform:

**LAMP Layer (Traditional REST)**
- Serves the web dashboard synchronously
- Uses prepared statements for all SQL operations (injection prevention)
- Implements offset/limit pagination
- ETag-based HTTP caching for app state endpoints
- FULLTEXT indexing for search operations (todos, backlog)
- Soft-delete pattern with `deletedAt` timestamp columns
- Multi-statement transactions for batch operations

**Convex Layer (Real-time + Mobile Proxy)**
- Powers real-time subscriptions for the mobile client
- Handles JWT issuance and refresh token rotation
- Acts as API proxy: mobile clients hit Convex, which forwards to LAMP
- Uses Convex mutations for writes, queries for reads
- Compound indexes for filtered list queries
- Batched mutations for atomic batch operations
- Built-in optimistic concurrency for conflict resolution

**Shared MySQL Database**
- Both LAMP and Convex read/write to the same MySQL instance
- UUID v4 primary keys throughout (no auto-increment)
- ISO 8601 UTC timestamps (`TIMESTAMP` columns)
- Floating-point `sort_order` columns for reorderable lists
- Foreign key constraints where relational integrity matters
- JSON columns for flexible metadata (tags, labels)

---

## 3. API Design

### 3.1 API Specification

The complete API is defined in OpenAPI 3.1 format:
**File:** `e7-central-hub-openapi.yaml` (2995 lines, 84KB)

The spec is fully self-contained — all `$ref` pointers resolve intra-document.
No external schema dependencies.

### 3.2 API Groups

#### App State (`/state`, `/state/{key}`) — 8 operations
Key-value state management with delta-sync for mobile offline support.

| Operation | Method | Path | Description |
|-----------|--------|------|-------------|
| listAppState | GET | /state | List / delta-sync with `?since=` |
| batchFetchAppState | POST | /state/batch | Batch fetch by key list |
| batchUpsertAppState | PUT | /state/batch | Atomic batch upsert |
| batchDeleteAppState | DELETE | /state/batch | Atomic batch delete |
| getAppStateByKey | GET | /state/{key} | Single entry + ETag |
| upsertAppStateByKey | PUT | /state/{key} | Idempotent upsert |
| patchAppStateByKey | PATCH | /state/{key} | Shallow merge |
| deleteAppStateByKey | DELETE | /state/{key} | Idempotent delete |

**Key design decisions:**
- PUT is idempotent upsert (safe for mobile retry)
- PATCH is partial merge (404 if key missing)
- DELETE is idempotent (200 even if absent)
- Delta-sync via `?since=` timestamp for mobile reconciliation
- ETag + `If-None-Match` for HTTP caching
- Batch operations use all-or-nothing semantics

#### Todos (`/todos`) — 7 operations
Task management with rich filtering, soft-delete, and batch operations.

| Operation | Method | Path | Description |
|-----------|--------|------|-------------|
| listTodos | GET | /todos | Filter + paginate + sort |
| createTodo | POST | /todos | Create new todo |
| getTodo | GET | /todos/{todo_id} | Get single todo |
| updateTodo | PATCH | /todos/{todo_id} | Partial update |
| deleteTodo | DELETE | /todos/{todo_id} | Soft-delete |
| toggleTodoCompletion | POST | /todos/{todo_id}/toggle | Flip completion |
| batchTodoOperations | POST | /todos/batch | Atomic batch CRUD |

**Key design decisions:**
- Status flow: `pending -> in_progress -> completed`; `cancelled` is terminal
- Toggle: `in_progress`/`cancelled` -> `completed`; `completed` -> `pending`
- Soft-delete sets `deletedAt`; children become orphans by default
- `?cascade=true` cascades delete to children
- Parent-child hierarchy via `parentId` (arbitrary depth)
- FULLTEXT search on title + description
- Batch supports create/update/delete/toggle sub-operations

#### Backlog (`/backlog`) — 11 operations
Feature/bug/task tracking with hierarchical linking and triage.

| Operation | Method | Path | Description |
|-----------|--------|------|-------------|
| listBacklogItems | GET | /backlog | Rich filtering + pagination |
| createBacklogItem | POST | /backlog | Create feature/bug/task/idea |
| submitTriageIdea | POST | /backlog/triage | Lightweight triage (forces type=idea) |
| getBacklogItem | GET | /backlog/{id} | Get + optional expand |
| updateBacklogItem | PATCH | /backlog/{id} | Partial update |
| deleteBacklogItem | DELETE | /backlog/{id} | Delete (409 if has children) |
| listBacklogChildren | GET | /backlog/{id}/children | Direct children |
| linkChildToParent | POST | /backlog/{id}/children | Link with cycle detection |
| unlinkChild | DELETE | /backlog/{id}/children/{childId} | Unlink (set parentId=null) |
| listBacklogAncestors | GET | /backlog/{id}/ancestors | Full ancestor chain |
| batchBacklogOperations | POST | /backlog/batch | Bulk create/status/assign |

**Key design decisions:**
- Types: `feature`, `bug`, `task`, `idea`
- Statuses: `triage`, `open`, `in_progress`, `done`, `wont_fix`, `duplicate`
- Arbitrary-depth hierarchy via `parentId`
- Cycle detection on link (full ancestor chain check)
- Triage endpoint forces `type=idea` + `status=triage` regardless of input
- `source` field on triage captures origin channel for analytics
- Batch: bulk_create, bulk_status_change, bulk_assign (all-or-nothing)

#### Kanban Boards (`/boards`) — 18 operations
Full kanban workflow with boards, columns, cards, and drag-and-drop.

| Operation | Method | Path | Description |
|-----------|--------|------|-------------|
| createBoard | POST | /boards | Create board |
| listBoards | GET | /boards | List + expand columns/cards |
| getBoard | GET | /boards/{boardId} | Get board + expand |
| updateBoard | PATCH | /boards/{boardId} | Patch metadata |
| deleteBoard | DELETE | /boards/{boardId} | Soft-delete + cascade |
| createColumn | POST | /boards/{boardId}/columns | Add column |
| listColumns | GET | /boards/{boardId}/columns | List columns |
| updateColumn | PATCH | /boards/{boardId}/columns/{columnId} | Patch column |
| deleteColumn | DELETE | /boards/{boardId}/columns/{columnId} | Delete + reassign cards |
| reorderColumns | PUT | /boards/{boardId}/columns/reorder | Atomic column reorder |
| createCard | POST | /boards/{boardId}/cards | Add card |
| listCards | GET | /boards/{boardId}/cards | List + expand backlogItem |
| getCard | GET | /boards/{boardId}/cards/{cardId} | Get card |
| updateCard | PATCH | /boards/{boardId}/cards/{cardId} | Patch card |
| deleteCard | DELETE | /boards/{boardId}/cards/{cardId} | Soft-delete |
| moveCard | POST | /boards/{boardId}/cards/{cardId}/move | Move to column |
| reorderCards | PUT | /boards/{boardId}/cards/reorder | Reorder within column |
| getBoardStats | GET | /boards/{boardId}/stats | Dashboard statistics |

**Key design decisions:**
- Default columns: Todo, In Progress, Done (created on board creation)
- Column reorder: atomic `PUT` with ordered UUID array
- Card reorder: scoped per-column via board-scoped URI
- Card move: dedicated `POST /move` with `targetColumnId` + optional `sortOrder`
- WIP limits on columns (optional, 0 = unlimited)
- Card links to BacklogItem via `backlogItemId` for traceability
- Column deletion reassigns cards to first remaining column
- Stats endpoint for dashboard summary bar

### 3.3 Cross-Cutting Conventions

**Authentication**
- JWT Bearer tokens for mobile (Convex auth pipeline, short-lived, rotated refresh)
- API Keys for LAMP dashboard (long-lived, stored hashed in MySQL, scoped)

**Identifiers**
- UUID v4 for all primary keys
- Pattern: `^[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}$`

**Timestamps**
- ISO 8601 UTC everywhere: `2025-01-01T00:00:00.000Z`
- `createdAt`, `updatedAt`, `deletedAt` (soft-delete), `closedAt` (backlog)

**Pagination**
- Offset + limit query parameters
- Default limit: 20, max limit: 100
- Response envelope: `{ items: [...], total: N, offset: N, limit: N }`

**Partial Updates**
- PATCH is always partial
- Use explicit `null` to clear optional fields
- Nullable fields use `type: [string, "null"]` in schemas

**Error Responses**
- Standard error envelope: `{ code: N, message: "...", details: [...] }`
- HTTP status codes: 400 (bad request), 401 (unauthorized), 404 (not found), 409 (conflict), 500 (server error)

**Expand Pattern**
- `?query` parameter for embedding related data inline
- Reduces round-trips for mobile clients
- Examples: `?expand=columns`, `?expand=cards`, `?expand=backlogItem`, `?expand=children,ancestors`

---

## 4. Data Model

### 4.1 Entity Relationship Diagram (Logical)

```
AppState (1)  -- standalone key-value store
  appId: UUID
  key: string (dot-separated, e.g. "settings.theme")
  value: JSON
  version: integer (optimistic concurrency)
  etag: string

TodoItem (N)  -- hierarchical task list
  id: UUID
  title: string
  description: string | null
  status: TodoStatus (pending | in_progress | completed | cancelled)
  priority: TodoPriority (low | medium | high | urgent)
  dueDate: timestamp | null
  parentId: UUID | null (self-referential hierarchy)
  assignedToUserId: UUID | null
  tags: string[]

BacklogItem (N)  -- feature/bug/task tracking
  id: UUID
  type: BacklogItemType (feature | bug | task | idea)
  status: BacklogItemStatus (triage | open | in_progress | done | wont_fix | duplicate)
  priority: BacklogPriority (low | medium | high | critical)
  title: string
  description: string | null
  reporterId: UUID | null
  assignedToUserId: UUID | null
  parentId: UUID | null (self-referential hierarchy)
  tags: string[]
  estimatedEffort: integer | null
  actualEffort: integer | null

KanbanBoard (N)  -- board container
  id: UUID
  title: string
  description: string | null
  isArchived: boolean

KanbanColumn (N)  -- belongs to board
  id: UUID
  boardId: UUID (FK -> KanbanBoard)
  title: string
  sortOrder: float
  colorHex: string | null
  wipLimit: integer | null

KanbanCard (N)  -- belongs to column, optionally linked to backlog
  id: UUID
  columnId: UUID (FK -> KanbanColumn)
  boardId: UUID (FK -> KanbanBoard)
  title: string
  description: string | null
  sortOrder: float
  backlogItemId: UUID | null (FK -> BacklogItem, traceability)
  assigneeUserId: UUID | null
  labels: string[]
```

### 4.2 Schema Inventory

| Category | Count | Schemas |
|----------|-------|---------|
| Common | 6 | UUID, Timestamp, SortOrder, PaginationParams, PaginatedResponse, Error |
| Enum | 5 | TodoStatus, TodoPriority, BacklogItemType, BacklogItemStatus, BacklogPriority |
| Entity | 12 | AppState, TodoItem, BacklogItem, BacklogItemExpanded, KanbanBoard, KanbanColumn, KanbanCard, KanbanBoardExpanded, KanbanColumnExpanded, KanbanCardExpanded, KanbanBoardStats, BatchBacklogResult |
| Request | 15 | AppStateUpsertRequest, AppStateBatchRequest, TodoCreateRequest, TodoUpdateRequest, BacklogCreateRequest, BacklogUpdateRequest, KanbanBoardCreateRequest, KanbanBoardUpdateRequest, KanbanColumnCreateRequest, KanbanColumnUpdateRequest, KanbanCardCreateRequest, KanbanCardUpdateRequest, KanbanColumnReorderRequest, KanbanCardReorderRequest, KanbanCardMoveRequest |
| Response | 5 | AppStateBatchResponse, AppStateCollectionFilter, AppStateSyncMeta, TodoBatchResultSuccess, TodoBatchResultFailure |
| **Total** | **49** | |

---

## 5. Security Architecture

### 5.1 Authentication Flow (Mobile / Expo)

```
User Login -> Convex Auth -> JWT Access Token (short-lived)
                           -> Refresh Token (server-side rotation)

API Request -> Authorization: Bearer <jwt>
           -> Convex validates JWT
           -> Proxies to LAMP REST layer
           -> Returns response
```

### 5.2 Authentication Flow (Web Dashboard)

```
Admin Login -> LAMP Session -> API Key Generation (one-time display)

API Request -> X-API-Key: <key>
           -> LAMP validates against MySQL (hashed comparison)
           -> Processes request
           -> Returns response
```

### 5.3 Security Rules

- All endpoints require authentication (no public read-only routes)
- API keys stored bcrypt-hashed in MySQL; plain value shown only at issuance
- JWT tokens are short-lived; refresh tokens rotated server-side
- Prepared statements for all SQL queries (no string interpolation)
- Input validation via OpenAPI schema constraints (maxLength, pattern, enum)
- CORS configured per server environment

---

## 6. Implementation Notes

### 6.1 LAMP Implementation Patterns

**Prepared Statements (all endpoints):**
```php
$stmt = $pdo->prepare("SELECT * FROM todos WHERE status = ? AND deleted_at IS NULL ORDER BY sort_order LIMIT ? OFFSET ?");
$stmt->execute([$status, $limit, $offset]);
```

**Soft-Delete Pattern:**
```sql
-- Instead of DELETE, set deleted_at
UPDATE todos SET deleted_at = NOW() WHERE id = ?;
-- All list queries include: WHERE deleted_at IS NULL
```

**FULLTEXT Search:**
```sql
ALTER TABLE todos ADD FULLTEXT idx_todos_search (title, description);
SELECT * FROM todos WHERE MATCH(title, description) AGAINST(? IN BOOLEAN MODE);
```

**Atomic Reorder (single query):**
```sql
UPDATE kanban_columns
  SET sort_order = CASE id
    WHEN ? THEN 0
    WHEN ? THEN 1
    WHEN ? THEN 2
  END
  WHERE board_id = ? AND id IN (?, ?, ?);
```

**Batch Transaction:**
```php
$pdo->beginTransaction();
try {
  foreach ($operations as $op) { /* execute each */ }
  $pdo->commit();
} catch (Exception $e) {
  $pdo->rollBack();
  throw $e;
}
```

### 6.2 Convex Implementation Patterns

**Mutation Example (createTodo):**
```typescript
export const createTodo = mutation({
  args: { title: v.string(), priority: v.string(), ... },
  handler: async (ctx, args) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error("Unauthorized");
    return await ctx.db.insert("todos", {
      id: crypto.randomUUID(),
      title: args.title,
      priority: args.priority,
      status: args.status ?? "pending",
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString(),
    });
  },
});
```

**Compound Index (filtered list):**
```typescript
// convex/schema.ts
todos: defineTable({...})
  .index("by_status_priority", ["status", "priority"])
  .index("by_parent", ["parentId"])
  .index("by_assigned", ["assignedToUserId"]),
```

**Delta-Sync Query:**
```typescript
export const getStateSince = query({
  args: since: v.string() },
  handler: async (ctx, { since }) => {
    return await ctx.db
      .query("app_state")
      .filter(q => q.gt(q.field("updatedAt"), since))
      .collect();
  },
});
```

### 6.3 Database Schema (MySQL)

```sql
-- Core tables
CREATE TABLE app_state (
  app_id    CHAR(36) NOT NULL,
  key       VARCHAR(255) NOT NULL,
  value     JSON NOT NULL,
  version   INT NOT NULL DEFAULT 1,
  etag      CHAR(32) NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (app_id, key)
);

CREATE TABLE todos (
  id                CHAR(36) NOT NULL,
  title             VARCHAR(500) NOT NULL,
  description       TEXT,
  status            ENUM('pending','in_progress','completed','cancelled') NOT NULL DEFAULT 'pending',
  priority          ENUM('low','medium','high','urgent') NOT NULL,
  due_date          TIMESTAMP NULL,
  parent_id         CHAR(36) NULL,
  assigned_to_user_id CHAR(36) NULL,
  sort_order        DOUBLE NOT NULL DEFAULT 0,
  created_at        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at        TIMESTAMP NULL,
  PRIMARY KEY (id),
  INDEX idx_todos_status (status),
  INDEX idx_todos_parent (parent_id),
  INDEX idx_todos_assigned (assigned_to_user_id),
  FULLTEXT INDEX idx_todos_search (title, description)
);

CREATE TABLE backlog_items (
  id                CHAR(36) NOT NULL,
  type              ENUM('feature','bug','task','idea') NOT NULL,
  status            ENUM('triage','open','in_progress','done','wont_fix','duplicate') NOT NULL DEFAULT 'triage',
  priority          ENUM('low','medium','high','critical') NOT NULL,
  title             VARCHAR(500) NOT NULL,
  description       TEXT,
  reporter_id       CHAR(36) NULL,
  assigned_to_user_id CHAR(36) NULL,
  parent_id         CHAR(36) NULL,
  estimated_effort  INT NULL,
  actual_effort     INT NULL,
  sort_order        DOUBLE NOT NULL DEFAULT 0,
  created_at        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  closed_at         TIMESTAMP NULL,
  PRIMARY KEY (id),
  INDEX idx_backlog_type (type),
  INDEX idx_backlog_status (status),
  INDEX idx_backlog_parent (parent_id),
  INDEX idx_backlog_assigned (assigned_to_user_id),
  FULLTEXT INDEX idx_backlog_search (title, description)
);

CREATE TABLE kanban_boards (
  id          CHAR(36) NOT NULL,
  title       VARCHAR(255) NOT NULL,
  description TEXT,
  is_archived BOOLEAN NOT NULL DEFAULT FALSE,
  created_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at  TIMESTAMP NULL,
  PRIMARY KEY (id)
);

CREATE TABLE kanban_columns (
  id          CHAR(36) NOT NULL,
  board_id    CHAR(36) NOT NULL,
  title       VARCHAR(255) NOT NULL,
  sort_order  DOUBLE NOT NULL DEFAULT 0,
  color_hex   CHAR(7) NULL,
  wip_limit   INT NULL,
  created_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at  TIMESTAMP NULL,
  PRIMARY KEY (id),
  INDEX idx_columns_board (board_id),
  FOREIGN KEY (board_id) REFERENCES kanban_boards(id)
);

CREATE TABLE kanban_cards (
  id              CHAR(36) NOT NULL,
  column_id       CHAR(36) NOT NULL,
  board_id        CHAR(36) NOT NULL,
  title           VARCHAR(500) NOT NULL,
  description     TEXT,
  sort_order      DOUBLE NOT NULL DEFAULT 0,
  backlog_item_id CHAR(36) NULL,
  assignee_user_id CHAR(36) NULL,
  created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at      TIMESTAMP NULL,
  PRIMARY KEY (id),
  INDEX idx_cards_column (column_id),
  INDEX idx_cards_board (board_id),
  INDEX idx_cards_backlog (backlog_item_id),
  FOREIGN KEY (column_id) REFERENCES kanban_columns(id),
  FOREIGN KEY (board_id) REFERENCES kanban_boards(id)
);

-- Tags (many-to-many via JSON column or separate table)
-- For simplicity, tags are stored as JSON arrays in the parent table.
-- If tag-based querying becomes critical, normalize into:
--   tags (id, name) + todo_tags (todo_id, tag_id) + backlog_tags (backlog_id, tag_id)
```

---

## 7. Deployment Architecture

### 7.1 Server Environments

| Environment | URL | Description |
|-------------|-----|-------------|
| Production (Convex) | `https://api.e7-hub.com/v1` | Convex backend proxy |
| Production (LAMP) | `https://e7-hub.com/api/v1` | LAMP REST layer |
| Local Dev (Convex) | `http://localhost:3000/v1` | Convex dev server |
| Local Dev (LAMP) | `http://localhost:8080/api/v1` | Apache/PHP local |

### 7.2 Deployment Targets

- **LAMP**: Traditional VPS or shared hosting (Apache + PHP-FPM + MySQL)
- **Convex**: Managed Convex Cloud (serverless, auto-scaling)
- **Expo**: EAS Build -> App Store + Google Play Store
- **Web Dashboard**: Served by Apache from the same VPS as the LAMP API

---

## 8. Implementation Phasing

### Phase 1: Foundation (Weeks 1-2)
- MySQL schema creation (all 5 core tables + indexes)
- LAMP REST layer scaffolding (routing, auth middleware, error handling)
- Convex project setup + schema definition
- Base CRUD for Todos (simplest entity, validates the pattern)

### Phase 2: Core Features (Weeks 3-4)
- Backlog CRUD + hierarchy (parent-child linking, cycle detection)
- Kanban board CRUD (boards, columns, cards)
- Card move + reorder operations
- App state management (delta-sync, batch operations)

### Phase 3: Advanced Features (Weeks 5-6)
- Batch operations (todos, backlog)
- Board statistics endpoint
- Triage submission flow
- Expand pattern implementation across all endpoints

### Phase 4: Mobile + Polish (Weeks 7-8)
- Expo app API client generation (from OpenAPI spec)
- JWT auth integration with Convex
- Real-time subscriptions for kanban board updates
- ETag caching for app state on mobile

### Phase 5: Dashboard + Launch (Weeks 9-10)
- LAMP web dashboard UI
- API key management interface
- End-to-end testing
- Production deployment

---

## 9. OpenAPI Spec Validation Status

The complete API specification has been assembled and validated:

- **File:** `e7-central-hub-openapi.yaml`
- **Format:** OpenAPI 3.1.0 (YAML)
- **Lines:** 2,995
- **Size:** 84 KB
- **$ref Resolution:** 360/360 targets resolve (0 unresolved)
- **Operation IDs:** 44/44 unique
- **YAML Parsing:** Clean (no syntax errors)
- **Tags:** 4 groups defined at top-level
- **Security Schemes:** 2 (BearerAuth, ApiKeyAuth)
- **Servers:** 4 environments

---

## 10. Related Documents

| Document | Path | Description |
|----------|------|-------------|
| API Specification | `e7-central-hub-openapi.yaml` | Complete OpenAPI 3.1 spec (44 ops, 49 schemas) |
| Strategic Roadmap | `ROADMAP.md` | E7 LLC goals, constraints, and vision |
| Schemas (source) | `../workspaces/t_e6d3bc71/e7-central-hub-schemas.yaml` | 34 JSON Schemas (source fragment) |
| App State API | `../workspaces/t_ea7bc462/e7-app-state-api.yaml` | 8 app state endpoints (source fragment) |
| Todo API | `../workspaces/t_eb195782/e7-hub-todo-paths.yaml` | 7 todo endpoints (source fragment) |
| Backlog API | `../workspaces/t_ced88df1/e7-backlog-api-paths.yaml` | 11 backlog endpoints (source fragment) |
| Kanban API | `../workspaces/t_28a098dd/e7-kanban-api-paths.yaml` | 18 kanban endpoints (source fragment) |

---

## 11. Open Questions / HITL Decisions Needed

1. **User Management**: The API references `assignedToUserId` and `reporterId` but no User CRUD endpoints are defined. A user service (or integration with an existing auth provider) needs to be specified.

2. **Tags**: Currently stored as JSON arrays. If tag-based filtering/querying becomes critical, a normalized many-to-many structure should be considered.

3. **File Attachments**: No attachment/upload endpoints are defined. If cards or backlog items need file attachments, a separate `/uploads` endpoint and storage strategy (S3, local) will be needed.

4. **WebSockets**: The spec is REST-only. Real-time updates for the web dashboard (if needed beyond mobile) would require WebSocket or SSE endpoints.

5. **Rate Limiting**: No rate-limiting strategy is defined in the spec. Should be added at the gateway level.

6. **API Versioning**: Currently using URL path versioning (`/v1`). Long-term strategy for breaking changes needs definition.

---

*This document is the architectural companion to the `e7-central-hub-openapi.yaml` API specification. It provides the system context, design rationale, and implementation guidance needed to begin Phase 1 development.*
