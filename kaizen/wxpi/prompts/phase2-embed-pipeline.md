# Cursor Prompt — Phase 2: Embedding Pipeline Integration

## Context

The capture module stores signals. The embedding client can generate embeddings. Now we need to connect them — certain signal types should be automatically embedded and stored as context entries.

## Requirements

Update the Capture Module to:

1. **Embedding Router** (`src/capture/embedding-router.ts`):
   - After a signal is stored in SQLite, check if it should be embedded:
     - `explicit_input` → always embed
     - `correction` → always embed (also triggers re-embed of corrected entry)
     - `feedback` → don't embed (used to adjust importance scores instead)
     - `behavior` → don't embed individually (batched during self-improvement)
     - `system` → never embed
   - For embeddable signals:
     - Call `embedText(payload)` via the embedding client
     - Create a `context_entries` row with entry_type `'fact'`, content = payload text, source_ids = [signal.id], importance = signal.confidence
     - Store the embedding in `context_embeddings` table
     - All of this is fire-and-forget (don't block the capture pipeline)

2. **Correction Handling**:
   - When a correction signal is received, find the conflicting context entry (by matching content)
   - Mark the old entry as superseded (set importance to 0)
   - Create a new context entry with the corrected content
   - Re-embed the new content

3. **Feedback Handling**:
   - When positive feedback is received on a suggestion, increase the importance score of related context entries by +0.05 (cap at 1.0)
   - When negative feedback is received, decrease by -0.1 (floor at 0.1)

## File Structure

Add to `src/capture/embedding-router.ts`, integrate into `src/capture/capture-module.ts`

## Constraints
- Embedding must be async and non-blocking
- If embedding API fails, log the error and queue for retry (don't lose the signal — it's already in SQLite)
- Correction handling must find the right entry to supersede (match by content similarity or by source signal reference)

## Verification
1. An explicit_input signal creates both a user_signals row AND a context_entries row with embedding
2. A correction signal supersedes the old context entry and creates a new one
3. Positive feedback increases importance of related entries
4. Behavior signals are stored but not embedded
5. Pipeline continues working even if embedding API is down
