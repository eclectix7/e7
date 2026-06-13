---
title: Wxpi — Storage Schema
type: schema-doc
status: design
tags:
  - wxpi
  - sqlite
  - vector
  - markdown
  - schema
---

# Wxpi — Storage Schema

Defines the three storage layers: SQLite structured tables, vector embeddings, and markdown file conventions.

---

## 1. SQLite — Structured Tables

### 1.1 `user_profile`

The single source of truth for who the user is. One row per app install.

```sql
CREATE TABLE user_profile (
  id              TEXT PRIMARY KEY,          -- UUID v4
  display_name    TEXT,
  created_at      TEXT NOT NULL,             -- ISO 8601 UTC
  updated_at      TEXT NOT NULL,             -- ISO 8601 UTC
  app_version     TEXT,
  device_info     TEXT,                      -- JSON: { os, model, locale }
  config          TEXT DEFAULT '{}'          -- JSON blob for app settings
);
```

### 1.2 `user_signals`

Every captured user action or inferred signal. Append-only. This is the raw feed that drives everything.

```sql
CREATE TABLE user_signals (
  id              TEXT PRIMARY KEY,          -- UUID v4
  user_id         TEXT NOT NULL REFERENCES user_profile(id),
  signal_type     TEXT NOT NULL,             -- enum: explicit_input | behavior | feedback | correction | system
  category        TEXT NOT NULL,             -- e.g. 'preference', 'habit', 'interaction', 'content'
  source          TEXT NOT NULL,             -- e.g. 'settings_form', 'chat_message', 'navigation', 'dwell_time'
  payload         TEXT NOT NULL,             -- JSON: the actual signal data
  confidence      REAL DEFAULT 1.0,          -- 0.0-1.0, how confident we are in this signal
  context_hash    TEXT,                      -- hash of the embedding vector for dedup
  created_at      TEXT NOT NULL,             -- ISO 8601 UTC

  INDEX idx_signals_user_time (user_id, created_at),
  INDEX idx_signals_type (signal_type, category),
  INDEX idx_signals_source (source)
);
```

**Signal type taxonomy:**

| Type | Meaning | Example |
|------|---------|---------|
| `explicit_input` | User deliberately provided info | Filled out a preference form |
| `behavior` | Inferred from usage patterns | Opens app every morning at 8am |
| `feedback` | User rated or reacted to something | Liked a suggestion, dismissed a prompt |
| `correction` | User fixed something the assistant got wrong | "No, I meant X not Y" |
| `system` | App-generated, not user-initiated | Session started, app updated |

### 1.3 `preferences`

Normalized, deduplicated user preferences. Updated by the self-improvement engine. Each row is a single atomic preference.

```sql
CREATE TABLE preferences (
  id              TEXT PRIMARY KEY,          -- UUID v4
  user_id         TEXT NOT NULL REFERENCES user_profile(id),
  pref_key        TEXT NOT NULL,             -- e.g. 'theme', 'language', 'notification_time'
  pref_value      TEXT NOT NULL,             -- the preference value (string or JSON)
  source_signal_ids TEXT NOT NULL,           -- JSON array of user_signals.id that support this
  confidence      REAL DEFAULT 1.0,          -- aggregated confidence from source signals
  is_active       INTEGER DEFAULT 1,         -- 0 = superseded, 1 = current
  first_seen_at   TEXT NOT NULL,
  updated_at      TEXT NOT NULL,

  UNIQUE(user_id, pref_key, is_active),      -- only one active value per key
  INDEX idx_prefs_user (user_id, is_active)
);
```

### 1.4 `context_entries`

Semantic context chunks — each represents a meaningful "fact" or "observation" about the user, stored with its embedding for retrieval.

```sql
CREATE TABLE context_entries (
  id              TEXT PRIMARY KEY,          -- UUID v4
  user_id         TEXT NOT NULL REFERENCES user_profile(id),
  entry_type      TEXT NOT NULL,             -- 'fact', 'pattern', 'summary', 'goal', 'preference_detail'
  content         TEXT NOT NULL,             -- the human-readable text that gets embedded
  source_ids      TEXT,                      -- JSON array of user_signals.id
  importance      REAL DEFAULT 0.5,          -- 0.0-1.0, how important this is for retrieval
  access_count    INTEGER DEFAULT 0,         -- how many times this was retrieved (for decay)
  last_accessed   TEXT,
  created_at      TEXT NOT NULL,
  updated_at      TEXT NOT NULL,

  INDEX idx_context_user (user_id, entry_type),
  INDEX idx_context_importance (importance DESC)
);
```

### 1.5 `self_improvement_log`

Audit trail of every self-improvement action the system took. Append-only. Human-readable.

```sql
CREATE TABLE self_improvement_log (
  id              TEXT PRIMARY KEY,          -- UUID v4
  user_id         TEXT NOT NULL REFERENCES user_profile(id),
  trigger_type    TEXT NOT NULL,             -- 'signal_threshold', 'time_based', 'user_requested'
  analysis_summary TEXT NOT NULL,            -- what the engine found
  action_taken    TEXT NOT NULL,             -- what it did
  action_type     TEXT NOT NULL,             -- 'new_preference', 'updated_preference', 'merged_signals', 'pruned_context', 'generated_summary'
  affected_ids    TEXT,                      -- JSON array of affected record IDs
  user_approved   INTEGER,                   -- NULL = auto-applied, 0 = rejected, 1 = approved
  created_at      TEXT NOT NULL,

  INDEX idx_improvement_user (user_id, created_at)
);
```

---

## 2. Vector Store — Semantic Search

### 2.1 Approach

Two options depending on Expo constraints:

**Option A: sqlite-vec (recommended if native build is acceptable)**
- Load the `sqlite-vec` extension into `expo-sqlite`
- Store embeddings in a virtual table
- Perform ANN (approximate nearest neighbor) search in SQL

**Option B: JS-side brute force (simpler, works with standard Expo)**
- Store embedding arrays as JSON blobs in a regular SQLite table
- Load all vectors into memory on retrieval
- Compute cosine similarity in JS
- Viable up to ~10K vectors (benchmark on target devices)

### 2.2 Embedding Table (Option A — sqlite-vec)

```sql
-- Requires sqlite-vec extension loaded
CREATE VIRTUAL TABLE IF NOT EXISTS context_embeddings USING vec0(
  id TEXT PRIMARY KEY,
  embedding FLOAT[1536]                       -- text-embedding-3-small = 1536 dims
);
```

### 2.3 Embedding Table (Option B — JS brute force)

```sql
CREATE TABLE context_embeddings (
  id              TEXT PRIMARY KEY,
  entry_id        TEXT NOT NULL REFERENCES context_entries(id),
  embedding       TEXT NOT NULL,              -- JSON array of floats
  model           TEXT NOT NULL,              -- which embedding model was used
  created_at      TEXT NOT NULL,

  INDEX idx_emb_entry (entry_id)
);
```

### 2.4 Embedding Strategy

| Content | Embedding Model | Dimensions | When |
|---------|----------------|------------|------|
| `context_entries.content` | text-embedding-3-small | 1536 | On create/update |
| `user_signals.payload` | text-embedding-3-small | 1536 | On create (for behavior signals only) |
| Query text (at retrieval) | text-embedding-3-small | 1536 | At retrieval time |

**Refresh policy:**
- New signals → embed immediately (async, non-blocking)
- Updated context entries → re-embed on change
- Full re-embed → weekly or on model change (rare)

---

## 3. Markdown Files — Narrative Storage

### 3.1 Directory Structure

```
wxpi-data/
├── context/
│   └── daily/
│       ├── 2025-01-15.md
│       ├── 2025-01-16.md
│       └── ...
├── improvements/
│   ├── 2025-01-15.md
│   └── ...
├── summaries/
│   ├── weekly/
│   │   ├── 2025-W03.md
│   │   └── ...
│   └── monthly/
│       ├── 2025-01.md
│       └── ...
└── user-profile.md
```

### 3.2 Daily Context Log (`context/daily/YYYY-MM-DD.md`)

Append-only log of all signals captured that day. Human-readable.

```markdown
# Context Log — 2025-01-15

## Signals

- **08:12** [behavior] App opened (session #47)
- **08:15** [explicit_input] User set theme to "dark" via settings
- **08:22** [behavior] User spent 4m on task list screen
- **08:30** [feedback] User dismissed notification suggestion
- **09:01** [behavior] App opened again (session #48) — 2nd morning session

## Observations

- Consistent morning usage pattern (8am range)
- Prefers dark mode
- Not interested in notification prompts
```

### 3.3 Improvement Log (`improvements/YYYY-MM-DD.md`)

Every self-improvement action, logged for audit.

```markdown
# Improvement Log — 2025-01-15

## 14:30 — Signal Threshold Trigger (50 new signals)

### Analysis
- 12 signals indicate morning usage between 7:45-8:30am on weekdays
- 3 signals show consistent dark mode preference
- 1 correction: user said "don't suggest morning reminders"

### Actions Taken
1. **New preference**: `usage_peak = "weekday_mornings"` (auto-applied, confidence: 0.92)
2. **Updated preference**: `theme = "dark"` (reinforced, confidence: 0.98)
3. **New preference**: `morning_reminders = false` (from correction, confidence: 1.0)
4. **Merged**: 8 redundant "app opened" signals into pattern summary

### Pending HITL
- None (all high-confidence auto-applied)
```

### 3.4 Weekly Summary (`summaries/weekly/YYYY-WNN.md`)

Auto-generated by the self-improvement engine. High-level user context snapshot.

```markdown
# Weekly Summary — 2025 Week 03 (Jan 13-19)

## Usage Patterns
- Active days: 5/7 (Mon-Fri)
- Average sessions/day: 2.1
- Peak usage: 7:45-8:30am weekdays
- Total session time: 3h 42m

## Preferences Confirmed
- Theme: dark (confidence: 0.98)
- Morning reminders: disabled (confidence: 1.0)
- Language: English

## New This Week
- Started using task list feature (3 tasks created)
- Dismissed 2 notification suggestions

## System Changes
- 3 new preferences auto-applied
- 12 redundant signals merged
- Context entries: 47 → 38 (pruned)
```

### 3.5 User Profile Markdown (`user-profile.md`)

A living human-readable summary of the user. Updated by the self-improvement engine. This is the "at a glance" document.

```markdown
# User Profile — Todd (Kaizen)

> Auto-generated. Last updated: 2025-01-15T14:30:00Z

## Identity
- **Name:** Todd (prefers Kaizen)
- **Role:** Senior Full Stack Architect & Developer
- **Startup:** E7 LLC (Phase 1)

## Technical Profile
- **Stack:** LAMP, Node, React, jQuery, Expo
- **Style:** OO, Design Patterns, SOLID, GRASP
- **IDE:** Cursor
- **Apps:** 4 Android (Expo SDK 52-55), iOS pending

## Preferences
- Theme: dark
- Morning reminders: disabled
- Peak usage: weekday mornings (7:45-8:30am)

## Interests
- Math, physics, quantum mechanics, chaos theory
- Japanese culture, samurai/bushido
- Coffee, tea
- Neuro-hacking, bio-hacking

## Context Stats
- Total signals: 847
- Active preferences: 12
- Context entries: 38
- Self-improvements applied: 23
```

---

## 4. Storage Budget Guidelines

| Layer | Expected Size | Growth Rate | Cleanup Strategy |
|-------|--------------|-------------|------------------|
| SQLite structured | <5MB | ~100 signals/day | Archive signals >90 days |
| Vector store | <50MB | ~20 embeddings/day | Re-embed + deduplicate weekly |
| Markdown files | <2MB | ~5KB/day | Rotate daily logs >30 days, keep summaries |
| **Total** | **<60MB** | | |

---

## Related

- [[architecture|Architecture]] — how these stores fit into the system
- [[context-pipeline|Context Pipeline]] — how data flows into these tables
- [[retrieval-strategy|Retrieval Strategy]] — how data is queried from these tables
