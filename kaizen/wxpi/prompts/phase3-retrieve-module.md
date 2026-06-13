# Cursor Prompt — Phase 3: Retrieve Module

## Context

Wxpi has stored user signals, preferences, context entries with embeddings, and markdown logs. Now we need the Retrieve Module — the system that assembles context envelopes for LLM calls.

## Requirements

Create a Retrieve Module that:

1. **Profile Pull** (`src/retrieve/profile.ts`):
   - `getActivePreferences(userId: string): Promise<Preference[]>` — returns active preferences from SQLite
   - `getProfileSummary(prefs: Preference[]): string` — formats preferences into a compact text block
   - Caches results for 5 minutes (in-memory, per session)

2. **Semantic Search** (`src/retrieve/semantic.ts`):
   - `searchContext(userId: string, queryText: string, topK: number = 5): Promise<ContextResult[]>`
   - Embeds the query text using the embedding client (with cache)
   - Calls `semanticSearch` from the embeddings module
   - Filters by distance threshold (<0.4)
   - Returns results sorted by relevance

3. **Recent Context** (`src/retrieve/recent.ts`):
   - `getRecentContext(userId: string, maxChars: number = 500): Promise<string>`
   - Reads today's and yesterday's daily markdown logs
   - Returns the last `maxChars` characters (most recent entries)
   - Falls back to yesterday if today's log doesn't exist

4. **Context Envelope Assembly** (`src/retrieve/envelope.ts`):
   - `assembleEnvelope(userId: string, userQuery: string, tokenBudget: number = 2000): Promise<ContextEnvelope>`
   - Assembles in order: profile summary → semantic results → recent context
   - Respects token budget (use `estimateTokens(text: string): number` — approximate as `text.length / 4`)
   - Budget allocation: profile 15%, semantic 40%, recent 15%, reserve 30% for query+response
   - Truncates in order: recent first, then low-importance semantic, then low-confidence semantic
   - Never truncates below: profile summary + top 2 semantic results
   - Returns: `{ systemContext: string; tokenBudget: number; entries: ContextResult[]; truncated: boolean }`

## File Structure

```
src/retrieve/
├── profile.ts       # Profile pull + caching
├── semantic.ts      # Semantic search wrapper
├── recent.ts        # Recent markdown context
├── envelope.ts      # Context envelope assembly
├── token-estimator.ts  # Token counting utility
├── types.ts         # ContextEnvelope, ContextResult, Preference
└── index.ts         # Public API
```

## Constraints
- TypeScript strict mode
- All functions must be async
- Graceful degradation: if any phase fails, continue with available data
- Profile cache: 5-minute TTL, in-memory only
- Token estimation can be approximate (char count / 4)

## Verification
1. Profile pull returns active preferences formatted as text
2. Semantic search returns relevant context entries for a test query
3. Recent context reads from markdown files correctly
4. Envelope assembly produces a well-formatted system prompt
5. Token budget is respected (envelope doesn't exceed budget)
6. Truncation works correctly when context is too large
7. System works even if semantic search returns no results
