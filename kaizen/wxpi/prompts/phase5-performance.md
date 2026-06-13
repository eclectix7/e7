# Cursor Prompt — Phase 5: Performance Optimization

## Context

The core system is working (capture, embeddings, retrieval, self-improvement). Now we need to optimize for production performance.

## Requirements

Optimize the following areas:

### 1. Database Indexing (`src/database/optimize.ts`)
- Add composite indexes for the most common query patterns:
  - `user_signals`: (user_id, created_at DESC), (user_id, signal_type, created_at), (user_id, category, created_at)
  - `preferences`: (user_id, is_active, confidence DESC)
  - `context_entries`: (user_id, entry_type, importance DESC), (user_id, last_accessed)
  - `self_improvement_log`: (user_id, created_at DESC)
- Run `ANALYZE` after index creation to update query planner statistics
- Add a `PRAGMA optimize` call on app startup

### 2. Query Optimization
- Replace any `SELECT *` queries with specific column lists
- Add `LIMIT` to all queries that could return large result sets
- Use `EXPLAIN QUERY PLAN` to verify index usage on critical paths
- Batch SQLite writes where possible (use transactions for multi-insert operations)

### 3. Embedding Search Optimization
- Implement early termination in brute-force search: if top-K results all have distance < 0.15, stop scanning
- Add a `maxScan` parameter to limit how many vectors are checked (default: all, but configurable)
- Consider dimensionality reduction: project 1536-dim embeddings to 256 dims using random projection (pre-compute projection matrix, store reduced vectors alongside full ones)
- Benchmark: target <100ms for 5K vectors, <300ms for 10K vectors

### 4. Context Envelope Caching
- Cache assembled context envelopes keyed by (userId, queryHash) with 2-minute TTL
- Invalidate cache when new signals are captured for the user
- This avoids redundant SQLite queries + embedding API calls for similar queries in a session

### 5. Markdown File Optimization
- Keep only the last 500 lines of daily logs in the active file (rotate older entries to `context/daily/YYYY-MM-DD-archive.md`)
- Use file size check before append: if file > 10KB, create new file with `-part2` suffix
- Read only the tail of markdown files (last N lines) rather than loading entire file

### 6. Memory Management
- Implement LRU eviction for embedding cache (already at 100 entries — verify it works)
- Clear profile cache on app background, rehydrate on foreground
- Limit in-memory vector store to user's vectors only (don't load other users' data — relevant for shared device edge case)

### 7. Network Optimization
- Implement request coalescing: if multiple components request the same embedding simultaneously, deduplicate to a single API call
- Use HTTP keep-alive for OpenRouter calls (React Native fetch should handle this automatically)
- Implement offline queue: if network is down, queue embedding and chat requests, process when online

## Verification
1. All critical queries use indexes (verify with EXPLAIN QUERY PLAN)
2. Embedding search: 5K vectors in <100ms, 10K vectors in <300ms
3. Context envelope assembly: <50ms when cached, <3s when uncached (dominated by embedding API)
4. No memory leaks after 1000 signal captures (profile heap usage)
5. Offline queue processes correctly when connection returns
6. Markdown files don't grow unbounded (rotation working)
