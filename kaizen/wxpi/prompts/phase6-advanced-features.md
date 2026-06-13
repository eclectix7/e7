# Cursor Prompt — Phase 6: Advanced Features

## Context

Core system is complete. These are stretch features for enhanced self-improvement and user control.

## Requirements

Implement the following advanced features:

### 1. Context Decay (`src/self-improvement/decay.ts`)

- Function `applyContextDecay(userId: string): Promise<void>`
- Reduces importance of context entries that haven't been accessed recently:
  - Not accessed in 7 days: importance × 0.9
  - Not accessed in 14 days: importance × 0.8
  - Not accessed in 30 days: importance × 0.5
  - Not accessed in 60 days: importance × 0.2
- Floor: importance never goes below 0.05 (keeps the entry alive)
- Entries with importance < 0.1 after decay are candidates for pruning (require HITL)
- Runs during self-improvement cycle
- Log decay actions to improvement log

### 2. Context Conflict Auto-Resolution (`src/self-improvement/conflict-resolution.ts`)

- For conflicts where one preference has significantly higher confidence (>0.3 difference):
  - Auto-resolve by deactivating the lower-confidence preference
  - Log the resolution
- For conflicts with similar confidence (within 0.3):
  - Always require HITL
- For temporal conflicts (e.g., "morning_reminders: true" on weekdays, "false" on weekends):
  - Suggest a compound preference: `morning_reminders: "weekdays_only"`
  - Require HITL approval for compound preferences

### 3. Data Export/Import (`src/data/export.ts`, `src/data/import.ts`)

**Export** (`exportContextData(userId: string): Promise<string>`):
- Generates a single markdown file with all user data:
  - User profile
  - All preferences (active and inactive)
  - All context entries
  - Self-improvement log
  - Usage statistics
- Save to `wxpi-data/export/YYYY-MM-DD-export.md`
- Share via Expo Sharing API

**Import** (`importContextData(filePath: string): Promise<void>`):
- Parses an export markdown file
- Validates structure
- Inserts data into SQLite (with conflict detection: skip duplicates by ID)
- Re-embeds all imported context entries
- Requires user confirmation before import

### 4. Multi-Signal Batch Embedding (`src/embeddings/batch.ts`)

- Instead of embedding signals individually, batch them:
  - Collect embeddable signals over a 5-minute window
  - Send all texts in a single `embedBatch()` call
  - Store all embeddings in a single transaction
- Reduces API calls by ~80% (1 batch call vs N individual calls)
- Fallback: if batch fails, retry individually

### 5. Smart Notification Timing (`src/notifications/smart-timing.ts`)

- Use learned temporal patterns to time notifications optimally:
  - If user is most active 7:45-8:30am on weekdays, schedule notifications for 8:00am
  - If user dismisses notifications at a certain time, avoid that time window
- Uses `expo-notifications` for scheduling
- Respects user's `morning_reminders` preference
- All notification timing decisions are logged to improvement log

### 6. Context-Aware Suggestions (`src/suggestions/engine.ts`)

- Function `generateSuggestions(userId: string, currentScreen: string): Promise<Suggestion[]>`
- Uses context envelope + current screen context to generate proactive suggestions:
  - "You usually create tasks at this time. Want to add one?"
  - "You haven't reviewed your backlog in 5 days. Take a look?"
  - "Based on your preferences, try enabling dark mode?"
- Max 3 suggestions per session to avoid overwhelming user
- Each suggestion has: text, action (deep link), confidence
- User can dismiss suggestions (emits feedback signal)

## File Structure

```
src/self-improvement/
├── decay.ts              # Context decay logic
├── conflict-resolution.ts # Auto-resolution for high-confidence conflicts
├── llm-analyzer.ts       # (from previous prompt)
├── batch-embed.ts        # Batch embedding optimization
└── suggestions.ts        # Suggestion engine

src/data/
├── export.ts             # Data export
└── import.ts             # Data import

src/notifications/
└── smart-timing.ts       # Optimal notification timing
```

## Verification
1. Context decay correctly reduces importance over time
2. High-confidence conflicts auto-resolve; similar-confidence conflicts require HITL
3. Export produces a valid, complete markdown file
4. Import correctly restores data from export file
5. Batch embedding reduces API calls by ≥50%
6. Smart notification timing adapts to learned patterns
7. Suggestions are relevant and respect user preferences
8. All features are covered by unit tests
