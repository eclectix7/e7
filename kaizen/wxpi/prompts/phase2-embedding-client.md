# Cursor Prompt — Phase 2: Embedding Client

## Context

Wxpi needs to generate embeddings via OpenRouter for semantic search. The database and capture module are working. Now we need the embedding API client.

## Requirements

Create an Embedding Client module that:

1. **API Client** (`src/embeddings/client.ts`):
   - Function `embedText(text: string): Promise<number[]>` — calls OpenRouter `/embeddings` endpoint
   - Function `embedBatch(texts: string[]): Promise<number[][]>` — batch embedding (more efficient)
   - Uses model `openai/text-embedding-3-small` (1536 dimensions)
   - Reads API key from `expo-secure-store` (key: `openrouter_api_key`)
   - Includes headers: `Authorization`, `Content-Type`, `HTTP-Referer`, `X-Title`
   - Implements retry with exponential backoff (3 attempts: 1s, 2s, 4s)
   - Throws typed `EmbeddingError` on failure after all retries

2. **Embedding Cache** (`src/embeddings/cache.ts`):
   - In-memory LRU cache for query embeddings (max 100 entries)
   - Key: hash of input text, Value: embedding array
   - Function `getCached(hash: string): number[] | undefined`
   - Function `setCached(hash: string, embedding: number[]): void`
   - Auto-evict oldest entry when cache exceeds 100 entries

3. **Storage** (`src/embeddings/storage.ts`):
   - Function `storeEmbedding(entryId: string, embedding: number[], model: string): Promise<void>` — stores in `context_embeddings` table
   - Function `getEmbedding(entryId: string): Promise<number[] | null>` — retrieves by entry ID
   - Function `getAllEmbeddings(userId: string): Promise<{ id: string; embedding: number[] }[]>` — loads all for brute-force search

## File Structure

```
src/embeddings/
├── client.ts       # OpenRouter API client
├── cache.ts        # LRU embedding cache
├── storage.ts      # SQLite embedding storage/retrieval
├── types.ts        # EmbeddingError, types
└── index.ts        # Public API
```

## Constraints
- TypeScript strict mode
- API key must never be logged or stored in plain text
- All network errors must be caught and wrapped in EmbeddingError
- Cache must be memory-only (no persistence needed — embeddings are in SQLite)

## Verification
1. `embedText("hello world")` returns a 1536-dim array
2. `embedBatch(["a", "b"])` returns 2 arrays
3. Cache returns same result for same input without API call
4. Embeddings are stored and retrieved from SQLite correctly
5. Retry logic works (test with invalid key)
