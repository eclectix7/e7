# Cursor Prompt — Phase 4: Analyzer Module

## Context

The self-improvement engine needs an Analyzer that reads recent signals and produces an analysis report (patterns, conflicts, gaps, redundant entries, stale entries).

## Requirements

Create an Analyzer module (`src/self-improvement/analyzer.ts`):

1. **Main Analysis Function**:
   - `analyze(userId: string): Promise<AnalysisReport>`
   - Reads all signals since last improvement run
   - Reads all active preferences
   - Reads all context entries
   - Returns: `{ signalCount, patterns: Pattern[], conflicts: Conflict[], gaps: Gap[], redundant: ContextEntry[], stale: ContextEntry[] }`

2. **Pattern Detection**:
   - `detectPatterns(signals: NormalizedSignal[]): Pattern[]`
   - Time-of-day patterns: group behavior signals by hour range, flag if ≥5 observations in same range
   - Feature usage patterns: group by source, flag if ≥3 uses of same feature
   - Preference reinforcement: group preference signals by key, flag if ≥2 reinforcing signals
   - Each pattern has: type, description, confidence (0.5-0.95), supportingSignalIds

3. **Conflict Detection**:
   - `detectConflicts(prefs: Preference[], signals: NormalizedSignal[]): Conflict[]`
   - Check for contradictory active preferences (e.g., both "morning_reminders: true" and "morning_reminders: false")
   - Check for correction signals that contradict active preferences
   - Each conflict has: type, description, entries (IDs), resolution strategy

4. **Gap Detection**:
   - `detectGaps(entries: ContextEntry[], signals: NormalizedSignal[]): Gap[]`
   - Find signal categories with many signals but no corresponding context entries
   - Example: 20 behavior signals about "task_list" screen but no context entry about task list preferences

5. **Redundancy Detection**:
   - `findRedundantEntries(entries: ContextEntry[]): ContextEntry[]`
   - Find entries with >0.95 cosine similarity to each other
   - Return the lower-importance entry of each redundant pair

6. **Staleness Detection**:
   - `findStaleEntries(entries: ContextEntry[]): ContextEntry[]`
   - Find entries with importance < 0.3 AND last_accessed > 30 days ago AND access_count < 3

## Types

```typescript
interface Pattern {
  type: 'temporal_habit' | 'feature_preference' | 'preference_reinforcement';
  description: string;
  confidence: number;
  supportingSignalIds: string[];
}

interface Conflict {
  type: 'contradictory_preferences' | 'correction_vs_preference';
  description: string;
  entries: string[];
  signal?: string;
  resolution: 'keep_higher_confidence' | 'apply_correction';
}

interface Gap {
  category: string;
  signalCount: number;
  description: string;
}

interface AnalysisReport {
  signalCount: number;
  patterns: Pattern[];
  conflicts: Conflict[];
  gaps: Gap[];
  redundant: ContextEntry[];
  stale: ContextEntry[];
}
```

## Verification
1. Pattern detection finds time-of-day patterns from test signals
2. Conflict detection catches contradictory preferences
3. Gap detection identifies under-represented categories
4. Redundancy detection finds near-duplicate entries
5. Staleness detection finds old, unaccessed entries
6. Full analysis runs in <1 second for 500 signals
