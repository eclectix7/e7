# Cursor Prompt — Phase 5: Error Handling

## Context

The system needs to be resilient. Every external dependency (OpenRouter API, filesystem, network) can fail. Build comprehensive error handling and recovery.

## Requirements

### 1. Error Taxonomy (`src/errors/types.ts`)

```typescript
enum ErrorSeverity {
  TRANSIENT,    // retry will likely succeed (network timeout, rate limit)
  RECOVERABLE,  // can fall back to degraded mode (embedding API down)
  CRITICAL,     // requires user intervention (database corruption, no storage)
}

enum ErrorCategory {
  NETWORK,          // no connection, timeout, DNS failure
  API_RATE_LIMIT,   // 429 from OpenRouter
  API_AUTH,         // 401/403 — bad key
  API_MODEL,        // model not available, overloaded
  STORAGE_FULL,     // device storage exhausted
  STORAGE_CORRUPT,  // SQLite integrity check fails
  EMBEDDING_FAIL,   // embedding API failed after retries
  VECTOR_SEARCH,    // vector dimension mismatch, empty index
  FILESYSTEM,       // read/write failure
}
```

### 2. Retry Queue (`src/errors/retry-queue.ts`)

- Persistent retry queue using a new SQLite table `retry_queue`:
  ```sql
  CREATE TABLE retry_queue (
    id TEXT PRIMARY KEY,
    operation_type TEXT NOT NULL,  -- 'embed' | 'chat' | 'markdown_write'
    payload TEXT NOT NULL,         -- JSON: the operation parameters
    attempt_count INTEGER DEFAULT 0,
    max_attempts INTEGER DEFAULT 3,
    next_retry_at TEXT,            -- ISO timestamp
    created_at TEXT NOT NULL
  );
  ```
- On failure, enqueue the operation with exponential backoff schedule
- Background task processes the queue every 60 seconds
- After max attempts, log to markdown and notify user (don't silently drop)

### 3. Circuit Breaker (`src/errors/circuit-breaker.ts`)

- Implement circuit breaker pattern for OpenRouter API:
  - **Closed**: normal operation, requests pass through
  - **Open**: after 5 consecutive failures, stop making requests for 60 seconds
  - **Half-open**: after 60s, allow 1 test request. If success → close. If fail → open again.
- Separate circuit breaker for embeddings vs chat (one can be down while the other works)
- Expose circuit breaker state to the UI (show "AI features temporarily unavailable" banner)

### 4. Graceful Degradation Matrix

| Component Failure | Degraded Behavior |
|-------------------|-------------------|
| Embedding API down | Skip semantic search, use profile + recent context only. Queue embeddings for retry. |
| Chat API down | Show user-friendly message: "AI is temporarily unavailable. Your data is safe." |
| SQLite read failure | Fall back to markdown files only (read user-profile.md + today's log) |
| SQLite write failure | Queue writes in memory, flush when DB recovers. Show warning. |
| Filesystem full | Stop writing markdown logs. Continue with SQLite only. Notify user. |
| Vector search failure | Fall back to keyword matching on context_entries.content |

### 5. Health Check (`src/errors/health-check.ts`)

- Function `runHealthCheck(): Promise<HealthStatus>` that checks:
  - SQLite: `SELECT 1` + integrity check
  - Filesystem: can read/write to document directory
  - OpenRouter: lightweight API call (GET /models or a trivial embedding)
  - Storage space: check available disk space
- Returns: `{ overall: 'healthy' | 'degraded' | 'critical', components: { [name]: { status, lastError, lastSuccess } } }`
- Run on app startup and every 5 minutes while app is active
- Store last health check result in SQLite for debugging

### 6. User-Facing Error UI

- Non-blocking error toasts (not alerts) for transient errors
- Persistent banner for degraded mode: "Some AI features are temporarily unavailable"
- Error log screen: show recent errors with timestamps, let user copy for support
- Never crash the app — all errors must be caught and handled

### 7. Data Integrity

- Run `PRAGMA integrity_check` on SQLite weekly (during self-improvement cycle)
- If corruption detected: attempt recovery via `REINDEX`, if that fails, export data to markdown, reinitialize DB, re-import
- Keep a `data-export.md` weekly backup of critical data (preferences, context entries) in markdown format

## Verification
1. Circuit breaker opens after 5 consecutive API failures
2. Circuit breaker recovers after successful half-open test
3. Retry queue processes failed operations when service recovers
4. App continues working (degraded) when OpenRouter is down
5. App continues working (degraded) when filesystem is full
6. Health check correctly identifies each failure mode
7. No unhandled promise rejections in any error scenario
8. Data integrity check runs weekly and logs results
