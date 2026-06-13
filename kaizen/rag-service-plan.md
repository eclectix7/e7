# RAG Service -- Implementation Plan

**Created:** 2026-05-29
**Status:** Draft -- Pending Review
**Author:** OWL for E7 LLC

---

## Architecture Overview

Your existing architecture is LAMP + Convex + OpenRouter. This RAG service
follows the same split: LAMP (HostPapa) for the document upload UI and file
storage, Convex for the RAG pipeline (embeddings, vector storage, retrieval),
and OpenRouter for both embedding generation and LLM chat completion.

```
  HostPapa LAMP Server                    Convex Cloud
  +-------------------+                   +------------------+
  | PHP Upload UI     |   HTTPS REST      | RAG API          |
  | (upload, list,    |------------------>| - /ingest        |
  |  delete docs)     |                   | - /query         |
  |                   |   WebSocket/SSE    | - /documents     |
  | /uploads/ folder  |<-----------------|                  |
  | (raw txt + pdf)   |                   | Vector DB        |
  |                   |                   | (Convex internal  |
  | MySQL (optional)  |                   |  vector storage) |
  | - doc metadata    |                   |                  |
  | - chunk index     |                   +-------+----------+
  +-------------------+                           |
                                                  v
                                          +------------------+
                                          | OpenRouter API   |
                                          | - embeddings     |
                                          | - chat/LLM       |
                                          +------------------+
```

---

## Component 1: PHP Upload UI (HostPapa LAMP)

### Purpose

Let you upload PDF and TXT files through a browser interface. Files live
on the HostPapa server. Metadata lives in MySQL. The PHP layer is purely
a UI + file management front-end; all intelligence happens in Convex.

### File Structure (HostPapa)

```
/rag/
├── index.php              Main dashboard: doc list, upload form, delete
├── upload.php             Handles file upload, validation, metadata save
├── delete.php             Soft-delete + optional hard-delete
├── query.php              Query proxy to Convex (optional)
├── content.php            Content fetcher endpoint for Convex (optional)
├── config.php             DB credentials, Convex endpoint, allowed types
├── assets/
│   └── style.css          Minimal clean UI
└── uploads/               Raw uploaded files (PDF, TXT)
    ├── {uuid}.pdf
    └── {uuid}.txt
```

### Database Schema (MySQL on HostPapa)

```sql
CREATE TABLE documents (
  id            CHAR(36) PRIMARY KEY,       -- UUID v4
  filename      VARCHAR(255) NOT NULL,      -- Original upload name
  file_type     ENUM('pdf','txt') NOT NULL,
  file_path     VARCHAR(512) NOT NULL,      -- Relative to /uploads/
  file_size     INT UNSIGNED NOT NULL,      -- Bytes
  title         VARCHAR(500) DEFAULT NULL,  -- Optional user-given title
  description   TEXT DEFAULT NULL,          -- Optional user-given summary
  tags          JSON DEFAULT NULL,          -- ["tag1","tag2"]
  status        ENUM('uploaded','processing','indexed','failed')
                NOT NULL DEFAULT 'uploaded',
  convex_doc_id VARCHAR(255) DEFAULT NULL,
  chunk_count   INT UNSIGNED DEFAULT 0,
  error_msg     TEXT DEFAULT NULL,
  created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at    TIMESTAMP NULL,
  INDEX idx_status (status),
  INDEX idx_type (file_type),
  FULLTEXT idx_search (title, description, filename)
);
```

This table tracks what is on the LAMP side. Convex maintains its own
document + chunk tables with vector embeddings.

### Upload Flow (PHP)

1. User submits file + optional title/description/tags via HTML form.
2. PHP validates: file type (pdf/txt), size limit (e.g. 25MB), MIME check.
3. PHP generates UUID, moves file to /uploads/{uuid}.{ext}.
4. PHP inserts row in MySQL with status 'uploaded'.
5. PHP calls Convex /ingest endpoint with extracted text + metadata.
6. Convex processes: chunk -> embed -> store.
7. PHP polls or Convex callbacks to update status to 'indexed' (or 'failed').

### PHP Pages

**index.php**
- List all active documents with status badges.
- Search/filter by title, tags, type.
- Delete button per document (soft-delete).
- Link to trigger re-ingest.
- Simple query form (textarea + submit, displays response inline).

**upload.php**
- multipart form: file input, title, description, tags (comma-separated).
- Drag-and-drop zone (vanilla JS, no framework needed).
- Progress bar via XHR.

**delete.php**
- POST handler, CSRF token.
- Soft-delete in MySQL, call Convex /delete to purge chunks.

**query.php** (optional lightweight UI)
- Receives prompt + optional document selection from form.
- Forwards to Convex /query endpoint.
- Displays response + sources.

### PDF Text Extraction (PHP side)

Use `smalot/pdfparser` (composer package) to extract text from PDFs on the
PHP side before sending to Convex. This keeps the Convex side simple and
avoids sending raw file bytes across the wire.

```bash
# On HostPapa (via SSH or composer)
composer require smalot/pdfparser
```

---

## Component 2: Convex RAG Backend

### Purpose

Houses the entire intelligence pipeline: text extraction, chunking,
embedding, vector storage, retrieval, contextual prompting, and
LLM completion via OpenRouter.

### File Structure (Convex project)

```
/convex/
├── schema.ts              Table definitions + vector index
├── documents.ts           Doc CRUD + ingest orchestration
├── chunks.ts              Chunk queries + vector search
├── rag.ts                 Main query endpoint (retrieve -> prompt -> LLM)
├── openrouter.ts          OpenRouter API client (embeddings + chat)
├── text_processing.ts     Text chunking helpers
├── http.ts                HTTP route definitions
└── ingest.ts              Ingest actions (called from PHP)
```

### Convex Schema (schema.ts)

```typescript
// Document catalog (mirrors LAMP metadata but adds Convex-side fields)
documents: defineTable({
  lamUuid: v.string(),             // Matches MySQL document.id
  filename: v.string(),
  fileType: v.union(v.literal("pdf"), v.literal("txt")),
  title: v.optional(v.string()),
  description: v.optional(v.string()),
  tags: v.array(v.string()),
  status: v.union(
    v.literal("uploaded"), v.literal("processing"),
    v.literal("indexed"), v.literal("failed")
  ),
  totalChunks: v.number(),
  error: v.optional(v.string()),
  createdAt: v.string(),
  updatedAt: v.string(),
})
  .index("by_lam_uuid", ["lamUuid"])
  .index("by_status", ["status"])
  .index("by_tags", ["tags"]),

// Text chunks with vector embeddings
chunks: defineTable({
  documentId: v.id("documents"),
  lamUuid: v.string(),
  chunkIndex: v.number(),
  content: v.string(),
  tokenCount: v.number(),
  embedding: v.array(v.number()),  // Float32 vector
})
  .index("by_document", ["documentId"])
  .index("by_lam_uuid", ["lamUuid"])
  .vectorIndex("by_embedding", {
    vectorField: "embedding",
    dimensions: 1536,              // text-embedding-3-small
    filterFields: ["lamUuid", "documentId"],
  }),

// Query log (optional, for analytics)
query_log: defineTable({
  prompt: v.string(),
  documentFilter: v.optional(v.array(v.string())),  // lamUuid list
  chunksRetrieved: v.number(),
  model: v.string(),
  response: v.string(),
  createdAt: v.string(),
}),
```

### Text Chunking Strategy (text_processing.ts)

- Method: Recursive character split with overlap.
- Chunk size: ~1000 tokens.
- Overlap: ~200 tokens (to preserve context at boundaries).
- Split on: paragraphs first, then sentences, then hard cut.
- Store chunk_index for ordering.

### Ingest Flow (ingest.ts / documents.ts)

1. Receive via HTTP action or internal call:
   - lamUuid, filename, fileType, title, description, tags, extractedText.
2. Create document row in Convex with status 'processing'.
3. Chunk the text using text_processing.ts.
4. Call OpenRouter /embeddings (model: text-embedding-3-small) --
   batch chunks if possible (up to 100 per request).
5. Store chunk rows with embeddings in Convex.
6. Update document status to 'indexed', set totalChunks.
7. Return result to caller.

### OpenRouter Integration (openrouter.ts)

```
Base URL: https://openrouter.ai/api/v1
Auth: Bearer <OPENROUTER_API_KEY> (stored in Convex env vars)
```

**Embeddings:**
```
POST /v1/embeddings
model: "openai/text-embedding-3-small"
input: array of chunk texts (batch)
```

**Chat (query):**
```
POST /v1/chat/completions
model: from env var DEFAULT_MODEL or caller override
messages: [{role: "system", ...}, {role: "user", ...}]
```

These are standard OpenAI-compatible calls -- no special SDK needed,
just fetch().

### RAG Query Flow (rag.ts)

**Endpoint:** POST /query (convex-http action or internal action called by PHP)

**Request body:**
```json
{
  "prompt": "Summarize the key arguments in my uploaded paper",
  "document_ids": ["uuid1", "uuid2"],
  "model": "anthropic/claude-sonnet-4",
  "chunk_limit": 8
}
```

- `prompt` (required): The user's query.
- `document_ids` (optional): Scope search to specific documents by lamUuid.
  If omitted, search across all indexed documents.
- `model` (optional): Override the default chat model.
- `chunk_limit` (optional): Number of chunks to retrieve. Default 6, max 20.

**Flow:**

1. Embed the prompt using OpenRouter /embeddings.
2. Vector search in Convex:
   - If document_ids provided: filter chunks WHERE lamUuid IN (...)
     then ANN search on embedding.
   - If no document_ids: search across all indexed chunks.
   - Return top-K chunks (default 6, max 20).
3. Build context string from retrieved chunks:
   ```
   [Document: {filename}, chunk {n}/{total}]
   {chunk_content}
   ```
4. Build prompt:
   ```
   System: "You are a RAG assistant. Answer based ONLY on the following
   context. If the context doesn't contain the answer, say 'This
   information is not in your documents.' Cite your sources using the
   document filenames."

   User: "Context:\n{context}\n\nQuestion: {prompt}"
   ```
5. Call OpenRouter /chat/completions with the composed prompt.
6. Return:
   ```json
   {
     "response": "...",
     "sources": [
       {
         "documentId": "...",
         "lamUuid": "...",
         "filename": "...",
         "chunkIndex": 0,
         "similarity": 0.92
       }
     ],
     "model": "...",
     "tokens_used": {"prompt": 1500, "completion": 300}
   }
   ```
7. Log the query to query_log table.

### Deletion (documents.ts)

- **Soft-delete:** Set status='deleted', keep chunks for potential restore.
- **Hard-delete:** Remove document row + all chunk rows from Convex.
  Vector index updates automatically.

---

## Component 3: Data Flow Summary

### UPLOAD
```
Browser
  -> POST /rag/upload.php (file + metadata)
  -> validate + save file
  -> MySQL INSERT (status=uploaded)
  -> POST Convex /ingest (extracted text + metadata)
  -> Convex: chunk -> embed -> store vectors
  -> Convex: UPDATE status=indexed, totalChunks
  -> (optional) callback to PHP or PHP polls for status
  -> Browser shows 'indexed' status
```

### QUERY
```
Browser (via PHP UI)
  -> POST /rag/query.php (prompt + optional doc selection)
  -> Forwards to Convex /query
  -> Convex: embed prompt -> vector search -> build context
  -> OpenRouter: chat completion with context
  -> Return response + sources
  -> Browser displays answer with source citations
```

Direct Convex access (no PHP):
```bash
curl -X POST https://YOUR_CONVEX.convex.site/query \
  -H "Content-Type: application/json" \
  -d '{"prompt":"What does X say about Y?","document_ids":["uuid1"]}'
```

### DELETE
```
Browser
  -> POST /rag/delete.php (document_id, CSRF)
  -> Soft-delete in MySQL (set deleted_at)
  -> POST Convex /delete (lamUuid)
  -> Convex: delete all chunks for lamUuid
  -> Convex: delete document row or set status=deleted
  -> Confirm to PHP
  -> Browser removes/hides document from list
```

### RE-INGEST
```
Browser triggers re-ingest
  -> Convex: delete old chunks for lamUuid
  -> Re-extract text (from stored file ref or PHP resends)
  -> Re-chunk and re-embed
  -> Store new chunks
  -> Update status=indexed, new chunk count
```

---

## Component 4: Security

### LAMP Side
- Session-based or API key auth for the PHP UI.
- Validate all file uploads: MIME type, extension check, size limit.
- Prevent directory traversal in file paths.
- CSRF tokens on state-changing forms.
- Prepared statements for all SQL queries.

### Convex Side
- OpenRouter API key stored as Convex environment variable (not in code).
- Validate all inputs in actions.
- Rate limit the /query endpoint.

### Between LAMP and Convex
- Shared secret in request headers (X-RAG-Secret) to authenticate
  ingest/delete calls from PHP to Convex.
- This secret is stored in both PHP config and Convex env vars.

### Between Convex and OpenRouter
- Standard Bearer token auth.

---

## Component 5: OpenRouter Model Choices

| Purpose | Model | Dimensions / Notes |
|---------|-------|--------------------|
| Embeddings | openai/text-embedding-3-small | 1536 dims, $0.02/1M tokens |
| Chat (fast/cheap) | anthropic/claude-haiku-4-5 | Good for testing |
| Chat (balanced) | anthropic/claude-sonnet-4 | Recommended default |
| Chat (powerful) | anthropic/claude-opus-4 | Best quality, higher cost |
| Chat (Google) | google/gemini-2.5-flash | Fast, strong reasoning |

Store the default chat model in Convex env vars. Let callers override per query.

---

## Component 6: Implementation Phases

### Phase 1: Foundation
- [ ] Set up Convex project, deploy schema (documents, chunks, query_log).
- [ ] Wire up OpenRouter client in Convex (embeddings + chat).
- [ ] Build PHP upload form + file handling + MySQL table on HostPapa.
- [ ] Test one-shot ingest of a small TXT file end-to-end.

### Phase 2: Core RAG
- [ ] Implement text chunking in Convex.
- [ ] Implement vector storage + Convex vector index.
- [ ] Implement the /query flow: embed -> search -> prompt -> LLM.
- [ ] Build a minimal query UI in PHP (textarea + submit + display).
- [ ] Test with 3-5 documents of mixed size/types.

### Phase 3: Polish
- [ ] Add PDF text extraction on PHP side (smalot/pdfparser).
- [ ] Add tags, filtering, document management in PHP UI.
- [ ] Add query analytics (query_log) and simple dashboard.
- [ ] Error handling + status tracking + retry for failed ingests.
- [ ] Re-ingest flow.
- [ ] Security hardening (CSRF, rate limiting, input validation).

### Phase 4: Stretch (Optional)
- [ ] Document-level access control (per-user doc visibility).
- [ ] Streaming responses (SSE) from Convex to browser.
- [ ] Batch upload (multiple files at once).
- [ ] Export query history.
- [ ] Support for additional file types (md, docx via conversion).

---

## Key Technical Decisions to Confirm (HITL)

1. **PDF text extraction:** PHP-side (using smalot/pdfparser) before sending
   text to Convex, or send raw file bytes to Convex? PHP-side is simpler
   and avoids large payloads.

2. **Embedding dimension:** 1536 matches OpenAI's text-embedding-3-small.
   If you use a different model (e.g. Gemini embeddings = 768 dims),
   the vector index must match.

3. **PHP UI auth:** Session-based (username/password) or API key?
   For personal use, session auth is fine.

4. **Query interface:** Full PHP UI for querying, or call Convex directly
   from curl/Postman and build a query UI later?

5. **Default chat model:** Recommended starting point is claude-sonnet-4
   for quality and cost balance. Confirm preference.

6. **Convex project:** New Convex project for RAG, or add to existing
   E7 Central Hub Convex project?

---

## File Inventory

### To Create on HostPapa
| File | Purpose |
|------|---------|
| /rag/config.php | DB credentials, Convex URL, shared secret |
| /rag/index.php | Document list dashboard |
| /rag/upload.php | Upload handler |
| /rag/delete.php | Delete handler |
| /rag/query.php | Query proxy to Convex (optional) |
| /rag/content.php | Content fetcher for Convex sync (optional) |

### To Create in Convex Project
| File | Purpose |
|------|---------|
| /convex/schema.ts | DB schema + vector index |
| /convex/documents.ts | Ingest + delete actions |
| /convex/chunks.ts | Chunk queries |
| /convex/rag.ts | Main query logic |
| /convex/openrouter.ts | OpenRouter API client |
| /convex/text_processing.ts | Chunking, text cleanup |
| /convex/http.ts | HTTP route definitions |
