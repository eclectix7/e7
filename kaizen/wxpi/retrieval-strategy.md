---
title: Wxpi — Retrieval Strategy
type: strategy-doc
status: design
tags:
  - wxpi
  - retrieval
  - context
  - semantic-search
  - prompt-engineering
---

# Wxpi — Retrieval Strategy

How context is fetched, assembled, and injected into LLM prompts.

---

## 1. Retrieval Overview

When the user triggers an AI action (chat, suggestion, automation), the system must quickly assemble a **context envelope** — a compact, relevant snapshot of everything the system knows about the user that's pertinent to this specific request.

```
User Request
    │
    ▼
┌──────────────────────────────────────────┐
│         Context Envelope Assembly         │
│                                          │
│  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │ Profile   │  │ Semantic │  │ Recent │ │
│  │ (SQLite)  │  │ Search   │  │ Context│ │
│  │           │  │ (Vector) │  │ (MMD)  │ │
│  └─────┬────┘  └────┬─────┘  └───┬────┘ │
│        └─────────────┼────────────┘      │
│                      ▼                   │
│            ┌──────────────────┐          │
│            │  Context Envelope │          │
│            │  (compact prompt) │          │
│            └──────────────────┘          │
└──────────────────────────────────────────┘
                      │
                      ▼
              OpenRouter LLM Call
```

---

## 2. Retrieval Phases

### Phase 1: Profile Pull (always, <10ms)

Pull the user's active preferences from SQLite. This is a simple indexed lookup.

```typescript
async function getProfileContext(userId: string): Promise<ProfileContext> {
  const prefs = await db.query(`
    SELECT pref_key, pref_value, confidence
    FROM preferences
    WHERE user_id = ? AND is_active = 1
    ORDER BY confidence DESC
  `, [userId]);

  return {
    preferences: prefs.map(p => `${p.pref_key}: ${p.pref_value}`),
    summary: generateProfileSummary(prefs),
  };
}
```

### Phase 2: Semantic Search (<100ms with sqlite-vec, <500ms brute force)

Embed the user's current query, then search for similar context entries.

```typescript
async function semanticSearch(
  userId: string,
  queryText: string,
  topK: number = 5
): Promise<ContextEntry[]> {
  // 1. Embed the query
  const queryEmbedding = await embedText(queryText);

  // 2. Search (sqlite-vec path)
  const results = await db.query(`
    SELECT ce.id, ce.content, ce.entry_type, ce.importance,
           vec_distance_cosine(ce.embedding, ?) as distance
    FROM context_entries ce
    JOIN context_embeddings emb ON emb.entry_id = ce.id
    WHERE ce.user_id = ?
    ORDER BY distance ASC
    LIMIT ?
  `, [queryEmbedding, userId, topK]);

  // 3. Filter by relevance threshold
  return results.filter(r => r.distance < 0.4); // cosine distance, lower = more similar
}
```

**Relevance threshold tuning:**

| Distance | Meaning | Action |
|----------|---------|--------|
| <0.2 | Highly relevant | Always include |
| 0.2-0.4 | Relevant | Include if room in context budget |
| 0.4-0.6 | Tangential | Include only if few other results |
| >0.6 | Not relevant | Exclude |

### Phase 3: Recent Context (<50ms)

Load the most recent daily markdown log for temporal context.

```typescript
async function getRecentContext(userId: string): Promise<string> {
  const today = new Date().toISOString().slice(0, 10);
  const yesterday = getYesterday();

  // Try today first, fall back to yesterday
  const todayPath = `${DOC_DIR}/context/daily/${today}.md`;
  const yesterdayPath = `${DOC_DIR}/context/daily/${yesterday}.md`;

  let content = '';
  if (await FileSystem.getInfoAsync(todayPath)) {
    content += await FileSystem.readAsStringAsync(todayPath);
  }
  if (await FileSystem.getInfoAsync(yesterdayPath)) {
    content += '\n' + await FileSystem.readAsStringAsync(yesterdayPath);
  }

  // Return last ~500 chars (most recent entries)
  return content.slice(-500);
}
```

---

## 3. Context Envelope Assembly

### 3.1 Assembly Logic

```typescript
interface ContextEnvelope {
  systemContext: string;     // injected into system prompt
  tokenBudget: number;      // max tokens for context
  entries: ContextEntry[];  // what was included
  truncated: boolean;       // whether we hit the budget
}

async function assembleEnvelope(
  userId: string,
  userQuery: string,
  tokenBudget: number = 2000
): Promise<ContextEnvelope> {
  const profile = await getProfileContext(userId);
  const semanticResults = await semanticSearch(userId, userQuery, 8);
  const recentContext = await getRecentContext(userId);

  // Build from most to least important, respecting token budget
  let usedTokens = 0;
  const sections: string[] = [];
  const includedEntries: ContextEntry[] = [];
  let truncated = false;

  // Section 1: User profile summary (always first)
  const profileSection = formatProfileSection(profile);
  const profileTokens = estimateTokens(profileSection);
  sections.push(profileSection);
  usedTokens += profileTokens;

  // Section 2: Relevant context entries (by relevance, then importance)
  const entriesSection: string[] = [];
  for (const entry of semanticResults) {
    const entryText = `- [${entry.entry_type}] ${entry.content} (confidence: ${entry.importance})`;
    const entryTokens = estimateTokens(entryText);

    if (usedTokens + entryTokens > tokenBudget * 0.7) {
      truncated = true;
      break;
    }
    entriesSection.push(entryText);
    includedEntries.push(entry);
    usedTokens += entryTokens;
  }
  if (entriesSection.length > 0) {
    sections.push('## Relevant Context\n' + entriesSection.join('\n'));
  }

  // Section 3: Recent activity (fill remaining budget)
  const remainingBudget = tokenBudget - usedTokens;
  const truncatedRecent = truncateToTokens(recentContext, remainingBudget * 0.5);
  if (truncatedRecent) {
    sections.push('## Recent Activity\n' + truncatedRecent);
  }

  return {
    systemContext: sections.join('\n\n'),
    tokenBudget,
    entries: includedEntries,
    truncated,
  };
}
```

### 3.2 Context Envelope Format (what the LLM sees)

```
## User Profile
- Theme: dark (confidence: 0.98)
- Peak usage: weekday mornings (confidence: 0.92)
- Morning reminders: disabled (confidence: 1.0)
- Language: English

## Relevant Context
- [fact] User prefers concise responses over detailed explanations (confidence: 0.85)
- [pattern] User typically creates 2-3 tasks per session (confidence: 0.78)
- [preference_detail] User dislikes notification prompts during morning hours (confidence: 0.91)

## Recent Activity
- 08:12 App opened (session #47)
- 08:15 User set theme to "dark" via settings
- 08:22 User spent 4m on task list screen
```

---

## 4. Token Budget Management

### 4.1 Budget Allocation

| Section | Budget Share | For 2000 token budget |
|---------|-------------|----------------------|
| User profile | 15% | 300 tokens |
| Semantic results | 40% | 800 tokens |
| Recent context | 15% | 300 tokens |
| **Total context** | **70%** | **1400 tokens** |
| Remaining for query + response | 30% | 600 tokens |

### 4.2 Truncation Strategy

When context exceeds budget, truncate in this order:
1. **Recent context** — least important, most volatile
2. **Low-importance semantic results** — drop entries with importance < 0.5 first
3. **Low-confidence semantic results** — drop entries with confidence < 0.6
4. **Profile details** — keep only top 3 preferences

Never truncate below: profile summary + top 2 semantic results.

---

## 5. Retrieval Patterns by Use Case

### 5.1 Chat / Conversational AI

```
Query: user's message text
Top-K: 5-8 context entries
Recent: today + yesterday
Budget: 2000 tokens
Focus: preferences + interaction patterns + recent corrections
```

### 5.2 Proactive Suggestions

```
Query: "what should I suggest to the user right now?"
Top-K: 3-5 context entries
Recent: today only
Budget: 1000 tokens
Focus: habits + time-of-day patterns + recent behavior
```

### 5.3 Task Automation

```
Query: user's task description + current screen context
Top-K: 3 context entries
Recent: last 3 days
Budget: 1500 tokens
Focus: task preferences + feature usage patterns + past similar tasks
```

### 5.4 Self-Improvement Analysis

```
Query: "analyze recent user signals for patterns"
Top-K: all entries (no semantic filter — full scan)
Recent: last 7 days
Budget: 4000 tokens (higher — this is background analysis)
Focus: all signal types, behavior patterns, preference conflicts
```

---

## 6. Caching

### 6.1 Embedding Cache

Cache query embeddings to avoid redundant API calls:

```typescript
const embeddingCache = new Map<string, number[]>();

async function getCachedEmbedding(text: string): Promise<number[]> {
  const hash = simpleHash(text);
  if (embeddingCache.has(hash)) {
    return embeddingCache.get(hash)!;
  }
  const embedding = await embedText(text);
  embeddingCache.set(hash, embedding);
  // Evict oldest if cache > 100 entries
  if (embeddingCache.size > 100) {
    const firstKey = embeddingCache.keys().next().value;
    embeddingCache.delete(firstKey);
  }
  return embedding;
}
```

### 6.2 Profile Cache

Preferences rarely change within a session. Cache the profile context:

```typescript
let profileCache: { data: ProfileContext; timestamp: number } | null = null;
const PROFILE_CACHE_TTL = 5 * 60 * 1000; // 5 minutes

async function getCachedProfile(userId: string): Promise<ProfileContext> {
  if (profileCache && Date.now() - profileCache.timestamp < PROFILE_CACHE_TTL) {
    return profileCache.data;
  }
  const profile = await getProfileContext(userId);
  profileCache = { data: profile, timestamp: Date.now() };
  return profile;
}
```

---

## 7. Fallback Behavior

| Scenario | Fallback |
|----------|----------|
| No semantic results found | Use profile + recent context only |
| Vector search fails | Use profile + recent context + last 5 signals from SQLite |
| SQLite unavailable | Use markdown files only (read user-profile.md + today's log) |
| OpenRouter embed API fails | Use cached embedding if available; otherwise skip semantic search |
| Context envelope empty | Proceed with bare user query (no context injection) |

The system **degrades gracefully** — missing context reduces personalization quality but never breaks functionality.

---

## 8. Retrieval Performance Targets

| Operation | Target | Max | Notes |
|-----------|--------|-----|-------|
| Profile pull | <10ms | <50ms | Indexed SQLite query |
| Embedding (API) | <2s | <5s | Network dependent |
| Vector search (sqlite-vec) | <50ms | <200ms | In-process |
| Vector search (brute force) | <200ms | <500ms | Up to 10K vectors |
| Markdown read | <20ms | <100ms | Local filesystem |
| Full envelope assembly | <3s | <8s | Dominated by embedding API |

---

## Related

- [[architecture|Architecture]] — where retrieval fits in the system
- [[storage-schema|Storage Schema]] — table definitions for queried data
- [[context-pipeline|Context Pipeline]] — how data gets stored that we're now retrieving
- [[openrouter-integration|OpenRouter Integration]] — embedding API details
