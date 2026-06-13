Good, I have full context on your architecture. Now let me write the plan.                                                                                         
                                                                                                                                                                        
     Here is the plan. I will write it to ~/e7/kaizen/rag-service-plan.md for your reference.                                                                           
                                                                                                                                                                        
     RAG SERVICE -- IMPLEMENTATION PLAN                                                                                                                                 
     ====================================                                                                                                                               
                                                                                                                                                                        
     Your existing architecture is LAMP + Convex + OpenRouter. This RAG service                                                                                         
     follows the same split: LAMP (HostPapa) for the document upload UI and file                                                                                        
     storage, Convex for the RAG pipeline (embeddings, vector storage, retrieval),                                                                                      
     and OpenRouter for both embedding generation and LLM chat completion.                                                                                              
                                                                                                                                                                        
     ARCHITECTURE OVERVIEW                                                                                                                                              
     =====================                                                                                                                                              
                                                                                                                                                                        
       HostPapa LAMP Server                    Convex Cloud                                                                                                             
       +-------------------+                   +------------------+                                                                                                     
       | PHP Upload UI     |   HTTPS REST       | RAG API          |                                                                                                    
       | (upload, list,    |------------------->| - /ingest        |                                                                                                    
       |  delete docs)     |                   | - /query         |                                                                                                     
       |                   |   WebSocket/SSE    | - /documents     |                                                                                                    
       | /uploads/ folder  |<-------------------|                  |                                                                                                    
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
                                                                                                                                                                        
     COMPONENT 1: PHP UPLOAD UI (HostPapa LAMP)                                                                                                                         
                                                                                                                                                                        
     Purpose                                                                                                                                                            
       Let you upload PDF and TXT files through a browser interface. Files live                                                                                         
       on the HostPapa server. Metadata lives in MySQL. The PHP layer is purely                                                                                         
       a UI + file management front-end; all intelligence happens in Convex.                                                                                            
                                                                                                                                                                        
     File Structure (HostPapa)                                                                                                                                          
                                                                                                                                                                        
       /rag/                                                                                                                                                            
       |-- index.php              Main dashboard: doc list, upload form, delete                                                                                         
       |-- upload.php             Handles file upload, validation, metadata save                                                                                        
       |-- delete.php             Soft-delete + optional hard-delete                                                                                                    
       |-- config.php             DB credentials, Convex endpoint, allowed types                                                                                        
       |-- assets/                                                                                                                                                      
       |-- |-- style.css          Minimal clean UI                                                                                                                      
       |-- uploads/               Raw uploaded files (PDF, TXT)                                                                                                         
       |-- |-- {uuid}.pdf                                                                                                                                               
       |-- |-- {uuid}.txt                                                                                                                                               
                                                                                                                                                                        
     Database Schema (MySQL on HostPapa)                                                                                                                                
                                                                                                                                                                        
       CREATE TABLE documents (                                                                                                                                         
         id          CHAR(36) PRIMARY KEY,      -- UUID v4                                                                                                              
         filename    VARCHAR(255) NOT NULL,     -- Original upload name                                                                                                 
         file_type   ENUM('pdf','txt') NOT NULL,                                                                                                                        
         file_path   VARCHAR(512) NOT NULL,     -- Relative to /uploads/                                                                                                
         file_size   INT UNSIGNED NOT NULL,     -- Bytes                                                                                                                
         title       VARCHAR(500) DEFAULT NULL,-- Optional user-given title                                                                                             
         description TEXT DEFAULT NULL,         -- Optional user-given summary                                                                                          
         tags        JSON DEFAULT NULL,         -- ["tag1","tag2"]                                                                                                      
         status      ENUM('uploaded','processing','indexed','failed')                                                                                                   
                     NOT NULL DEFAULT 'uploaded',                                                                                                                       
         convex_doc_id VARCHAR(255) DEFAULT NULL,                                                                                                                       
         chunk_count INT UNSIGNED DEFAULT 0,                                                                                                                            
         error_msg   TEXT DEFAULT NULL,                                                                                                                                 
         created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,                                                                                                               
         updated_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,                                                                                   
         deleted_at  TIMESTAMP NULL,                                                                                                                                    
         INDEX idx_status (status),                                                                                                                                     
         INDEX idx_type (file_type),                                                                                                                                    
         FULLTEXT idx_search (title, description, filename)                                                                                                             
       );                                                                                                                                                               
                                                                                                                                                                        
       This table tracks what is on the LAMP side. Convex maintains its own                                                                                             
       document + chunk tables with vector embeddings.                                                                                                                  
                                                                                                                                                                        
     Upload Flow (PHP)                                                                                                                                                  
                                                                                                                                                                        
       1. User submits file + optional title/description/tags via HTML form.                                                                                            
       2. PHP validates: file type (pdf/txt), size limit (e.g. 25MB), MIME check.                                                                                       
       3. PHP generates UUID, moves file to /uploads/{uuid}.{ext}.                                                                                                      
       4. PHP inserts row in MySQL with status 'uploaded'.                                                                                                              
       5. PHP triggers async call to Convex /ingest endpoint (fire-and-forget                                                                                           
          via Convex HTTP action or a cron-triggered sync).                                                                                                             
       6. Status flips to 'processing' -> 'indexed' (or 'failed').                                                                                                      
       7. UI polls or shows status via page refresh.                                                                                                                    
                                                                                                                                                                        
     PHP Pages                                                                                                                                                          
                                                                                                                                                                        
       index.php                                                                                                                                                        
         - List all active documents with status badges.                                                                                                                
         - Search/filter by title, tags, type.                                                                                                                          
         - Delete button per document (soft-delete).                                                                                                                    
         - Link to trigger re-ingest.                                                                                                                                   
                                                                                                                                                                        
       upload.php                                                                                                                                                       
         - multipart form: file input, title, description, tags (comma-separated).                                                                                      
         - Drag-and-drop zone (vanilla JS, no framework needed).                                                                                                        
         - Progress bar via XHR.                                                                                                                                        
                                                                                                                                                                        
       delete.php                                                                                                                                                       
         - POST handler, CSRF token.                                                                                                                                    
         - Soft-delete in MySQL, schedule Convex /delete call.                                                                                                          
                                                                                                                                                                        
     COMPONENT 2: CONVEX RAG BACKEND                                                                                                                                    
                                                                                                                                                                        
     Purpose                                                                                                                                                            
       Houses the entire intelligence pipeline: text extraction, chunking,                                                                                              
       embedding, vector storage, retrieval, contextual prompting, and                                                                                                  
       LLM completion via OpenRouter.                                                                                                                                   
                                                                                                                                                                        
     File Structure (Convex project)                                                                                                                                    
                                                                                                                                                                        
       /convex/                                                                                                                                                         
       |-- schema.ts              Table definitions + vector index                                                                                                      
       |-- documents.ts           Doc CRUD + ingest orchestration                                                                                                       
       |-- chunks.ts              Chunk queries + vector search                                                                                                         
       |-- rag.ts                 Main query endpoint (retrieve -> prompt -> LLM)                                                                                       
       |-- openrouter.ts          OpenRouter API client (embeddings + chat)                                                                                             
       |-- text_processing.ts     PDF/TXT text extraction helpers                                                                                                       
       |-- http.ts                HTTP routes (ingest, query, delete)                                                                                                   
                                                                                                                                                                        
     Convex Schema (schema.ts)                                                                                                                                          
                                                                                                                                                                        
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
         // Vector index defined separately below                                                                                                                       
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
                                                                                                                                                                        
     Text Extraction (text_processing.ts)                                                                                                                               
                                                                                                                                                                        
       PDF: Extract raw text on the Convex side by sending file content.                                                                                                
       TXT: Straightforward UTF-8 read.                                                                                                                                 
                                                                                                                                                                        
       Note: Since files live on HostPapa, the ingest flow must either:                                                                                                 
         (a) Send file content from LAMP to Convex during upload, or                                                                                                    
         (b) Make files accessible via signed URL / public URL for Convex                                                                                               
             HTTP actions to fetch.                                                                                                                                     
                                                                                                                                                                        
       Recommended: Option (a). During upload, PHP reads file content (or                                                                                               
       extracts PDF text on the server using a lightweight library like                                                                                                 
       smalot/pdfparser), then sends the extracted text to Convex.                                                                                                      
       This avoids exposing file storage publicly.                                                                                                                      
                                                                                                                                                                        
     Chunking Strategy                                                                                                                                                  
                                                                                                                                                                        
       Method: Recursive character split with overlap.                                                                                                                  
       Chunk size: ~1000 tokens.                                                                                                                                        
       Overlap: ~200 tokens (to preserve context at boundaries).                                                                                                        
       Split on: paragraphs first, then sentences, then hard cut.                                                                                                       
       Store chunk_index for ordering.                                                                                                                                  
                                                                                                                                                                        
     Embedding + Storage (documents.ts -- ingest action)                                                                                                                
                                                                                                                                                                        
       1. Receive: lamUuid, filename, fileType, title, description, tags,                                                                                               
          extractedText.                                                                                                                                                
       2. Chunk the text.                                                                                                                                               
       3. Call OpenRouter /embeddings (model: text-embedding-3-small) --                                                                                                
          batch chunks if possible.                                                                                                                                     
       4. Store document row + chunk rows in Convex tables.                                                                                                             
       5. Update status to 'indexed', set totalChunks.                                                                                                                  
                                                                                                                                                                        
     OpenRouter Integration (openrouter.ts)                                                                                                                             
                                                                                                                                                                        
       Base URL: https://openrouter.ai/api/v1                                                                                                                           
       Auth: Bearer <OPENROUTER_API_KEY> (stored in Convex env vars)                                                                                                    
                                                                                                                                                                        
       Embeddings:                                                                                                                                                      
         POST /v1/embeddings                                                                                                                                            
         model: "openai/text-embedding-3-small"  (or any OpenRouter-                                                                                                    
                available embedding model)                                                                                                                              
         input: array of chunk texts                                                                                                                                    
                                                                                                                                                                        
       Chat (query):                                                                                                                                                    
         POST /v1/chat/completions                                                                                                                                      
         model: user-configurable default (e.g. "anthropic/claude-sonnet-4"                                                                                             
                or "google/gemini-2.5-flash")                                                                                                                           
         messages: [{role: "system", ...}, {role: "user", ...}]                                                                                                         
                                                                                                                                                                        
       These are standard OpenAI-compatible calls -- no special SDK needed,                                                                                             
       just fetch().                                                                                                                                                    
                                                                                                                                                                        
     Retrieval + RAG Query (rag.ts)                                                                                                                                     
                                                                                                                                                                        
       Endpoint: POST /query  (convex-http or internal action called by PHP)                                                                                            
                                                                                                                                                                        
       Request body:                                                                                                                                                    
       {                                                                                                                                                                
         "prompt": "Summarize the key arguments in my uploaded paper",                                                                                                  
         "document_ids": ["uuid1", "uuid2"],  // optional: scope to specific docs                                                                                       
         "model": "anthropic/claude-sonnet-4", // optional override                                                                                                     
         "chunk_limit": 8                     // optional, default 6                                                                                                    
       }                                                                                                                                                                
                                                                                                                                                                        
       Flow:                                                                                                                                                            
                                                                                                                                                                        
       1. Embed the prompt using OpenRouter /embeddings.                                                                                                                
       2. Vector search in Convex:                                                                                                                                      
          - If document_ids provided: filter chunks WHERE lamUuid IN (...)                                                                                              
            then ANN search on embedding.                                                                                                                               
          - If no document_ids: search across all indexed chunks.                                                                                                       
          - Return top-K chunks (default 6, max 20).                                                                                                                    
       3. Build context string from retrieved chunks:                                                                                                                   
          "[Document: {filename}, chunk {n}/{total}]\n{chunk_content}"                                                                                                  
       4. Build prompt:                                                                                                                                                 
          System: "You are a RAG assistant. Answer based ONLY on the following                                                                                          
          context. If the context doesn't contain the answer, say 'This                                                                                                 
          information is not in your documents.'"                                                                                                                       
          User: "Context:\n{context}\n\nQuestion: {prompt}"                                                                                                             
       5. Call OpenRouter /chat/completions with the composed prompt.                                                                                                   
       6. Return:                                                                                                                                                       
          {                                                                                                                                                             
            "response": "...",                                                                                                                                          
            "sources": [                                                                                                                                                
              {"documentId": "...", "filename": "...", "chunkIndex": 0,                                                                                                 
               "similarity": 0.92}                                                                                                                                      
            ],                                                                                                                                                          
            "model": "...",                                                                                                                                             
            "tokens_used": {"prompt": N, "completion": N}                                                                                                               
          }                                                                                                                                                             
                                                                                                                                                                        
       7. Log the query to query_log table.                                                                                                                             
                                                                                                                                                                        
     Deletion (documents.ts)                                                                                                                                            
                                                                                                                                                                        
       Soft-delete: Set status='deleted', optionally purge chunks.                                                                                                      
       Hard-delete: Remove document row + all chunk rows.                                                                                                               
                                                                                                                                                                        
       When PHP calls delete, Convex deletes chunks first, then marks                                                                                                   
       document deleted. Vector index updates automatically.                                                                                                            
                                                                                                                                                                        
     Alternative: Sync-Based Ingest                                                                                                                                     
                                                                                                                                                                        
       If sending file content from PHP to Convex in one request is too                                                                                                 
       large (big PDFs), use a sync queue:                                                                                                                              
                                                                                                                                                                        
       1. PHP uploads file, sets status='uploaded'.                                                                                                                     
       2. A scheduled Convex cron (every 30s) queries MySQL for new                                                                                                     
          'uploaded' documents (via Convex HTTP action hitting a PHP status                                                                                             
          endpoint).                                                                                                                                                    
       3. Convex fetches the file content from a PHP endpoint                                                                                                           
          (e.g. /rag/content.php?id={uuid}&key={secret}).                                                                                                               
       4. Convex processes and indexes.                                                                                                                                 
                                                                                                                                                                        
       This is more complex. Start with direct content transfer first.                                                                                                  
                                                                                                                                                                        
     COMPONENT 3: DATA FLOW SUMMARY                                                                                                                                     
                                                                                                                                                                        
     UPLOAD                                                                                                                                                             
       Browser -> POST /rag/upload.php -> validate -> save file -> MySQL row                                                                                            
       (status=uploaded) -> POST Convex /ingest (with extracted text + metadata)                                                                                        
       -> chunk -> embed -> store vectors -> MySQL update (status=indexed)                                                                                              
                                                                                                                                                                        
     QUERY (browser-based, optional lightweight UI)                                                                                                                     
       Browser -> POST /rag/query.php -> forwards to Convex /query -> vector                                                                                            
       search -> OpenRouter chat -> return response + sources -> display                                                                                                
                                                                                                                                                                        
       Or query directly via curl/Postman against Convex:                                                                                                               
         curl -X POST https://YOUR_CONVEX.convex.site/query \                                                                                                           
           -H "Content-Type: application/json" \                                                                                                                        
           -d '{"prompt":"What does X say about Y?","document_ids":["uuid1"]}'                                                                                          
                                                                                                                                                                        
     DELETE                                                                                                                                                             
       Browser -> POST /rag/delete.php -> soft-delete MySQL -> POST Convex                                                                                              
       /delete -> remove chunks + vector entries -> confirm                                                                                                             
                                                                                                                                                                        
     RE-INGEST                                                                                                                                                          
       Browser triggers re-ingest on a document -> Convex deletes old chunks                                                                                            
       -> re-extracts/re-chunks/re-embeds from stored file reference ->                                                                                                 
       re-indexes.                                                                                                                                                      
                                                                                                                                                                        
     COMPONENT 4: SECURITY                                                                                                                                              
                                                                                                                                                                        
       LAMP side:                                                                                                                                                       
         - API key or session auth for the PHP UI.                                                                                                                      
         - Validate all file uploads: MIME type, extension check, size limit.                                                                                           
         - Prevent directory traversal in file paths.                                                                                                                   
         - CSRF tokens on state-changing forms.                                                                                                                         
                                                                                                                                                                        
       Convex side:                                                                                                                                                     
         - OpenRouter API key stored as Convex environment variable (not in code).                                                                                      
         - Validate all inputs in actions.                                                                                                                              
         - Rate limit the /query endpoint (Convex rate limiter or app-level).                                                                                           
                                                                                                                                                                        
       Between LAMP and Convex:                                                                                                                                         
         - Shared secret in request headers (X-RAG-Secret) to authenticate                                                                                              
           ingest/delete calls from PHP to Convex.                                                                                                                      
         - This secret is stored in both PHP config and Convex env vars.                                                                                                
                                                                                                                                                                        
       Between Convex and OpenRouter:                                                                                                                                   
         - Standard Bearer token auth.                                                                                                                                  
                                                                                                                                                                        
     COMPONENT 5: OPENROUTER MODEL CHOICES                                                                                                                              
                                                                                                                                                                        
       Embeddings (fixed):                                                                                                                                              
         Model: openai/text-embedding-3-small                                                                                                                           
         Dimensions: 1536                                                                                                                                               
         Cost: $0.02 / 1M tokens (very cheap)                                                                                                                           
                                                                                                                                                                        
       Chat (configurable, defaults):                                                                                                                                   
         Fast/cheap:     anthropic/claude-haiku-4-5                                                                                                                     
         Balanced:       anthropic/claude-sonnet-4  (recommended default)                                                                                               
         Powerful:       anthropic/claude-opus-4                                                                                                                        
         Google:         google/gemini-2.5-flash                                                                                                                        
                                                                                                                                                                        
       Store a default model in Convex env vars. Let callers override per query.                                                                                        
                                                                                                                                                                        
     COMPONENT 6: IMPLEMENTATION PHASES                                                                                                                                 
                                                                                                                                                                        
       Phase 1: Foundation (Week 1)                                                                                                                                     
         [ ] Set up Convex project, deploy schema (documents, chunks, query_log).                                                                                       
         [ ] Wire up OpenRouter client in Convex (embeddings + chat).                                                                                                   
         [ ] Build PHP upload form + file handling + MySQL table on HostPapa.                                                                                           
         [ ] Test one-shot ingest of a small TXT file end-to-end.                                                                                                       
                                                                                                                                                                        
       Phase 2: Core RAG (Week 2)                                                                                                                                       
         [ ] Implement text chunking in Convex.                                                                                                                         
         [ ] Implement vector storage + Convex vector index.                                                                                                            
         [ ] Implement the /query flow: embed -> search -> prompt -> LLM.                                                                                               
         [ ] Build a minimal query UI in PHP (textarea + submit + display).                                                                                             
         [ ] Test with 3-5 documents of mixed size/types.                                                                                                               
                                                                                                                                                                        
       Phase 3: Polish (Week 3)                                                                                                                                         
         [ ] Add PDF text extraction on PHP side before sending to Convex.                                                                                              
         [ ] Add tags, filtering, document management in PHP UI.                                                                                                        
         [ ] Add query analytics (query_log) and simple dashboard.                                                                                                      
         [ ] Error handling + status tracking + retry for failed ingests.                                                                                               
         [ ] Re-ingest flow.                                                                                                                                            
         [ ] Security hardening (CSRF, rate limiting, input validation).                                                                                                
                                                                                                                                                                        
       Phase 4: Stretch                                                                                                                                                 
         [ ] Document-level access control (per-user doc visibility).                                                                                                   
         [ ] Streaming responses (SSE) from Convex to browser.                                                                                                          
         [ ] Batch upload (multiple files at once).                                                                                                                     
         [ ] Export query history.                                                                                                                                      
         [ ] Support for additional file types (md, docx via conversion).                                                                                               
                                                                                                                                                                        
     KEY TECHNICAL DECISIONS TO CONFIRM (HITL)                                                                                                                          
     ==========================================                                                                                                                         
                                                                                                                                                                        
       1. PDF text extraction: Do it on PHP side (using smalot/pdfparser or                                                                                             
          similar) before sending text to Convex, or send raw file bytes to                                                                                             
          Convex and extract there? PHP-side is simpler and avoids large                                                                                                
          payloads.                                                                                                                                                     
                                                                                                                                                                        
       2. Convex vector embedding dimension: 1536 matches OpenAI's default.                                                                                             
          If you use a different model (e.g. Gemini embeddings = 768 dims),                                                                                             
          the vector index must match.                                                                                                                                  
                                                                                                                                                                        
       3. PHP UI auth: Session-based (username/password into the PHP app) or                                                                                            
          API key? For a personal tool, session auth is fine.                                                                                                           
                                                                                                                                                                        
       4. Query interface: Do you want a full PHP UI for querying, or are you                                                                                           
          okay calling Convex directly from curl/Postman and building a query                                                                                           
          UI later?                                                                                                                                                     
                                                                                                                                                                        
       5. Which OpenRouter chat model as default? I recommend starting with                                                                                             
          claude-sonnet-4 for quality and cost balance.                                                                                                                 
                                                                                                                                                                        
     FILES TO CREATE / MODIFY                                                                                                                                           
     =========================                                                                                                                                          
                                                                                                                                                                        
       ~/e7/kaizen/rag-service-plan.md            This plan                                                                                                             
       HostPapa: /rag/config.php                  Config (DB, Convex URL, secrets)                                                                                      
       HostPapa: /rag/index.php                   Document list dashboard                                                                                               
       HostPapa: /rag/upload.php                  Upload handler                                                                                                        
       HostPapa: /rag/delete.php                  Delete handler                                                                                                        
       HostPapa: /rag/query.php                   Query proxy to Convex (optional)                                                                                      
       HostPama: /rag/content.php                 Content fetcher for sync (optional)                                                                                   
       Convex:   /convex/schema.ts                DB schema                                                                                                             
       Convex:   /convex/documents.ts             Ingest + delete                                                                                                       
       Convex:   /convex/chunks.ts                Chunk queries                                                                                                         
       Convex:   /convex/rag.ts                   Main query logic                                                                                                      
       Convex:   /convex/openrouter.ts            API client                                                                                                            
       Convex:   /convex/text_processing.ts       Chunking, text cleanup                                                                                                
       Convex:   /convex/http.ts                  HTTP route definitions                                                                                                
                                                                                                                                                                        
     This plan keeps each component focused and simple. The PHP side is a thin                                                                                          
     UI -- no intelligence, just file management and metadata. All RAG smarts                                                                                           
     live in Convex. OpenRouter gives you model flexibility without committing                                                                                          
     to one provider. The two-server architecture mirrors your existing E7                                                                                              
     Central Hub pattern (LAMP for web UI + Convex for backend smarts).                                                                                                 
                                                                                                                                                                        
     Want me to start building a specific component first?                  
