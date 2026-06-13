---
title: Wxpi — Context Pipeline
type: pipeline-doc
status: design
tags:
  - wxpi
  - context
  - pipeline
  - capture
  - embeddings
---

# Wxpi — Context Pipeline

How user signals flow from raw interaction to stored, searchable context.

---

## 1. Pipeline Overview

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ 1. CAPTURE│───▶│2. NORMAL-│───▶│3. CLASSI-│───▶│4. STORE  │───▶│5. INDEX  │
│           │    │   IZE    │    │   FY     │    │          │    │(embed)   │
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
                                                                       │
                                                                       ▼
                                                                ┌──────────┐
                                                                │6. LOG to │
                                                                │ Markdown │
                                                                └──────────┘
```

Each stage is independent and testable. The pipeline runs asynchronously — the UI never blocks on context processing.

---

## 2. Stage 1: Capture

### 2.1 Signal Sources

**Explicit (user-initiated):**

| Source | Event Type | Example Payload |
|--------|-----------|-----------------|
| Settings form | `explicit_input` | `{ "theme": "dark", "notifications": false }` |
| Chat message | `explicit_input` | `{ "text": "I prefer concise answers" }` |
| Preference toggle | `explicit_input` | `{ "compact_view": true }` |
| Correction | `correction` | `{ "original": "...", "corrected": "...", "field": "preference" }` |
| Feedback (thumbs up/down) | `feedback` | `{ "target": "suggestion_123", "rating": "positive" }` |

**Implicit (inferred from behavior):**

| Source | Event Type | Trigger | Example Payload |
|--------|-----------|---------|-----------------|
| App open | `behavior` | `onAppForeground` | `{ "time": "08:12", "day": "monday", "session": 47 }` |
| Screen dwell | `behavior` | `onScreenChange` after >30s | `{ "screen": "task_list", "duration_sec": 240 }` |
| Feature use | `behavior` | Any feature interaction | `{ "feature": "create_task", "count": 1 }` |
| Session end | `behavior` | `onAppBackground` | `{ "session_duration_sec": 360, "screens_visited": 5 }` |
| Notification dismiss | `behavior` | `onNotificationDismiss` | `{ "notification_id": "reminder_001", "dwell_ms": 200 }` |

### 2.2 Capture Implementation

```typescript
// Event bus — all UI components emit to this
type SignalEvent = {
  type: 'explicit_input' | 'behavior' | 'feedback' | 'correction' | 'system';
  source: string;           // which component/screen
  payload: Record<string, any>;
  timestamp: string;        // ISO 8601
  sessionId: string;
};

// Capture module subscribes to event bus
class CaptureModule {
  private eventBus: EventBus;
  private pipeline: ContextPipeline;

  onSignal(event: SignalEvent) {
    // Non-blocking: enqueue for async processing
    this.pipeline.enqueue(event);
  }
}
```

---

## 3. Stage 2: Normalize

Raw signals arrive in different formats. Normalization converts them into a consistent structure.

### 3.1 Normalization Rules

```typescript
interface NormalizedSignal {
  id: string;                    // UUID v4, generated
  userId: string;                // from user_profile
  signalType: string;            // validated against enum
  category: string;              // derived from source + payload
  source: string;                // original source
  payload: string;               // JSON-stringified, cleaned
  confidence: number;            // 0.0-1.0
  timestamp: string;              // ISO 8601 UTC, normalized
  sessionId: string;
}

function normalize(raw: SignalEvent): NormalizedSignal {
  return {
    id: crypto.randomUUID(),
    userId: getCurrentUserId(),
    signalType: validateEnum(raw.type, SIGNAL_TYPES),
    category: classifyCategory(raw.source, raw.payload),
    source: raw.source,
    payload: JSON.stringify(sanitizePayload(raw.payload)),
    confidence: calculateConfidence(raw),
    timestamp: toUTCISO(raw.timestamp),
    sessionId: raw.sessionId,
  };
}
```

### 3.2 Confidence Scoring

| Signal Type | Base Confidence | Adjustments |
|------------|----------------|-------------|
| `explicit_input` | 1.0 | None — user said it directly |
| `correction` | 1.0 | None — user explicitly corrected |
| `feedback` | 0.8 | +0.1 if repeated, -0.2 if contradictory |
| `behavior` | 0.5 | +0.1 per repetition (cap 0.9), -0.1 if outlier |
| `system` | 0.3 | Informational only, not user intent |

### 3.3 Category Classification

```typescript
function classifyCategory(source: string, payload: any): string {
  // Rule-based classification
  if (source.includes('settings')) return 'preference';
  if (source.includes('chat')) return 'interaction';
  if (source.includes('navigation') || source.includes('screen')) return 'navigation';
  if (source.includes('dwell') || source.includes('session')) return 'engagement';
  if (payload.rating || payload.reaction) return 'feedback';
  if (payload.feature || payload.action) return 'feature_usage';
  return 'general';
}
```

---

## 4. Stage 3: Classify

After normalization, signals are classified for routing to the right storage and processing path.

### 4.1 Classification Decision Tree

```
Normalized Signal
    │
    ├─ Is it a correction?
    │   ├─ YES → Route to: PreferenceUpdater (immediate)
    │   │         Also: Invalidate conflicting context entries
    │   └─ NO ↓
    │
    ├─ Is it explicit input?
    │   ├─ YES → Route: SQLite (user_signals) + Embed + Markdown log
    │   └─ NO ↓
    │
    ├─ Is it feedback?
    │   ├─ YES → Route: SQLite (user_signals) + Update importance scores
    │   │         on related context entries
    │   └─ NO ↓
    │
    ├─ Is it behavior?
    │   ├─ YES → Route: SQLite (user_signals) + Batch for pattern detection
    │   │         (don't embed individually — wait for pattern consolidation)
    │   └─ NO ↓
    │
    └─ System signal → Route: SQLite only (no embedding, informational)
```

### 4.2 Deduplication

Before storing, check for near-duplicate signals:

```typescript
async function isDuplicate(signal: NormalizedSignal): Promise<boolean> {
  // Check: same user, same category, same payload hash, within 1 hour
  const recent = await db.query(`
    SELECT id FROM user_signals
    WHERE user_id = ?
      AND category = ?
      AND context_hash = ?
      AND created_at > datetime('now', '-1 hour')
    LIMIT 1
  `, [signal.userId, signal.category, hashPayload(signal.payload)]);

  return recent.length > 0;
}
```

---

## 5. Stage 4: Store

### 5.1 Storage Routing

```
                    ┌─────────────────────┐
                    │  Classified Signal   │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                ▼
     ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
     │ SQLite        │ │ Vector Store │ │ Markdown     │
     │ user_signals  │ │ (if embeddable)│ │ daily log    │
     └──────────────┘ └──────────────┘ └──────────────┘
```

### 5.2 SQLite Write

```typescript
async function storeSignal(signal: NormalizedSignal): Promise<void> {
  await db.run(`
    INSERT INTO user_signals (id, user_id, signal_type, category, source, payload, confidence, context_hash, created_at)
    VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
  `, [
    signal.id, signal.userId, signal.signalType, signal.category,
    signal.source, signal.payload, signal.confidence,
    hashPayload(signal.payload), signal.timestamp
  ]);
}
```

### 5.3 Vector Embedding (async, non-blocking)

Only embed signals that represent meaningful context (not raw behavior logs):

```typescript
async function maybeEmbed(signal: NormalizedSignal): Promise<void> {
  // Only embed explicit_input, corrections, and consolidated patterns
  if (!['explicit_input', 'correction'].includes(signal.signalType)) {
    return; // behavior signals get embedded only after pattern consolidation
  }

  const embedding = await openRouter.embed(signal.payload);

  // Create context entry
  const entryId = crypto.randomUUID();
  await db.run(`
    INSERT INTO context_entries (id, user_id, entry_type, content, source_ids, importance, created_at, updated_at)
    VALUES (?, ?, ?, ?, ?, ?, ?, ?)
  `, [
    entryId, signal.userId, 'fact', signal.payload,
    JSON.stringify([signal.id]), signal.confidence,
    signal.timestamp, signal.timestamp
  ]);

  // Store embedding
  await db.run(`
    INSERT INTO context_embeddings (id, entry_id, embedding, model, created_at)
    VALUES (?, ?, ?, ?, ?)
  `, [crypto.randomUUID(), entryId, JSON.stringify(embedding), 'text-embedding-3-small', signal.timestamp]);
}
```

### 5.4 Markdown Append

```typescript
async function appendToDailyLog(signal: NormalizedSignal): Promise<void> {
  const date = signal.timestamp.slice(0, 10); // YYYY-MM-DD
  const filePath = `${DOC_DIR}/context/daily/${date}.md`;

  const entry = `- **${signal.timestamp.slice(11, 16)}** [${signal.signalType}] ${signal.source}: ${summarizePayload(signal.payload)}\n`;

  await FileSystem.appendAsStringAsync(filePath, entry);
}
```

---

## 6. Stage 5: Index (Embedding)

### 6.1 When to Embed

| Signal Type | Embed Immediately? | When |
|------------|-------------------|------|
| `explicit_input` | Yes | Right after store |
| `correction` | Yes | Right after store (also triggers re-embed of corrected entry) |
| `feedback` | No | Used to adjust importance scores on existing entries |
| `behavior` | No | Batched — embedded during self-improvement cycle as patterns |
| `system` | No | Never embedded |

### 6.2 Embedding API Call

```typescript
async function embedText(text: string): Promise<number[]> {
  const response = await fetch('https://openrouter.ai/api/v1/embeddings', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${OPENROUTER_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      model: 'openai/text-embedding-3-small',
      input: text,
    }),
  });

  const data = await response.json();
  return data.data[0].embedding; // 1536-dim float array
}
```

### 6.3 Batch Embedding (for behavior patterns)

During self-improvement cycles, behavior signals are consolidated into patterns and embedded together:

```typescript
// Instead of embedding 50 individual "app opened" signals,
// consolidate into one pattern and embed that:
const pattern = "User opens app consistently between 7:45-8:30am on weekdays (observed 12 times over 2 weeks)";
const embedding = await embedText(pattern);
// Store as entry_type: 'pattern'
```

---

## 7. Pipeline Error Handling

| Failure | Handling |
|---------|----------|
| OpenRouter embed API fails | Retry 3x with exponential backoff. If still failing, queue for retry on next app launch. Signal is still stored in SQLite. |
| SQLite write fails | Log to markdown file as fallback. Queue for DB write retry. |
| Markdown write fails | Log to console. Non-critical — markdown is for human audit, not system function. |
| Deduplication check fails | Store anyway. Deduplicate during self-improvement cycle. |

---

## 8. Performance Budgets

| Operation | Target | Max |
|-----------|--------|-----|
| Signal capture to SQLite | <50ms | <200ms |
| Embedding API call | <2s | <5s |
| Markdown append | <20ms | <100ms |
| End-to-end (with embed) | <3s | <8s |
| End-to-end (without embed) | <100ms | <500ms |

All pipeline operations are **fire-and-forget from the UI perspective**. The user never waits.

---

## Related

- [[architecture|Architecture]] — where the pipeline fits in the system
- [[storage-schema|Storage Schema]] — exact table definitions
- [[retrieval-strategy|Retrieval Strategy]] — how stored context is queried
- [[self-improvement-engine|Self-Improvement Engine]] — how the pipeline feeds into self-improvement
