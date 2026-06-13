# Cursor Prompt — Phase 2: Vector Search

## Context

Wxpi stores embeddings in SQLite. Now we need the vector search module for semantic retrieval. Since we're in Expo managed workflow, we'll use JS-side brute-force cosine similarity (viable for <10K vectors).

## Requirements

Create a Vector Search module that:

1. **Similarity Functions** (`src/embeddings/search.ts`):
   - `cosineSimilarity(a: number[], b: number[]): number` — returns cosine similarity (-1 to 1, higher = more similar)
   - `cosineDistance(a: number[], b: number[]): number` — returns 1 - similarity (0 = identical, 2 = opposite)

2. **Search Function**:
   - `semanticSearch(userId: string, queryEmbedding: number[], topK: number = 5, threshold: number = 0.4): Promise<SearchResult[]>`
   - Loads all embeddings for the user from SQLite
   - Computes cosine distance between query and each entry
   - Filters by threshold (distance < 0.4)
   - Sorts by distance ascending
   - Returns top-K results with: `{ entryId: string; content: string; distance: number; importance: number }`

3. **Performance Note**:
   - For <10K vectors, brute force is acceptable (<200ms on modern devices)
   - If performance becomes an issue, consider: dimensionality reduction (PCA to 256 dims), or switching to sqlite-vec with a custom Expo dev client

## File Structure

Add to `src/embeddings/search.ts`, export from `src/embeddings/index.ts`

## Verification
1. `cosineSimilarity([1,0], [1,0])` returns 1.0
2. `cosineSimilarity([1,0], [0,1])` returns 0.0
3. `semanticSearch` returns relevant results for a test query
4. Threshold filtering works (irrelevant results excluded)
5. Performance: 5000 vectors searched in <200ms
