---
title: Wxpi — Self-Improvement Engine
type: engine-doc
status: design
tags:
  - wxpi
  - self-improvement
  - feedback-loop
  - orchestration
  - hitl
---

# Wxpi — Self-Improvement Engine

The system that allows Wxpi to **orchestrate its own context quality improvements** over time — analyzing user signals, detecting patterns, proposing changes, and applying them with human-in-the-loop oversight.

---

## 1. Design Philosophy

The self-improvement engine is not a background optimization — it's a **first-class system component** with the same architectural rigor as the capture and retrieval pipelines.

Core principles:
- **Transparent** — Every improvement action is logged to Markdown for human audit
- **Conservative** — When uncertain, ask the user (HITL). Never silently override user intent
- **Reversible** — All changes are soft-delete, not hard-delete. History is preserved.
- **Measurable** — Track whether improvements actually improved personalization quality

---

## 2. Engine Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                  Self-Improvement Engine                      │
│                                                              │
│  ┌────────────┐   ┌────────────┐   ┌────────────────────┐   │
│  │  Trigger    │   │  Analyzer  │   │  Action Planner    │   │
│  │  Scheduler  │──▶│            │──▶│                    │   │
│  └────────────┘   └────────────┘   └─────────┬──────────┘   │
│                                               │              │
│                                    ┌──────────▼──────────┐   │
│                                    │  HITL Gate          │   │
│                                    │  (Human Approval)   │   │
│                                    └──────────┬──────────┘   │
│                                               │              │
│                                    ┌──────────▼──────────┐   │
│                                    │  Executor           │   │
│                                    │  (Apply Changes)    │   │
│                                    └──────────┬──────────┘   │
│                                               │              │
│                                    ┌──────────▼──────────┐   │
│                                    │  Verifier           │   │
│                                    │  (Measure Impact)   │   │
│                                    └──────────┬──────────┘   │
│                                               │              │
│                                    ┌──────────▼──────────┐   │
│                                    │  Audit Logger       │   │
│                                    │  (Markdown + SQLite)│   │
│                                    └─────────────────────┘   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 3. Trigger Conditions

The engine runs when any of these conditions are met:

### 3.1 Signal Threshold

```typescript
const SIGNAL_THRESHOLD = 50; // new signals since last run

async function checkSignalThreshold(userId: string): Promise<boolean> {
  const lastRun = await getLastImprovementTime(userId);
  const newSignals = await db.query(`
    SELECT COUNT(*) as count FROM user_signals
    WHERE user_id = ? AND created_at > ?
  `, [userId, lastRun]);

  return newSignals[0].count >= SIGNAL_THRESHOLD;
}
```

### 3.2 Time-Based

```typescript
const TIME_THRESHOLD_HOURS = 24;

async function checkTimeThreshold(userId: string): Promise<boolean> {
  const lastRun = await getLastImprovementTime(userId);
  const hoursSince = (Date.now() - new Date(lastRun).getTime()) / 3600000;
  return hoursSince >= TIME_THRESHOLD_HOURS;
}
```

### 3.3 User-Requested

```typescript
// User explicitly asks: "improve my context" or "update my profile"
// Triggers immediate run, bypassing thresholds
```

### 3.4 Conflict Detected

```typescript
// During normal retrieval, if two active preferences contradict each other,
// trigger an immediate improvement cycle to resolve
// Example: pref A says "morning_reminders: true", pref B says "morning_reminders: false"
```

---

## 4. Analyzer Module

The analyzer reads recent signals and produces an **analysis report**.

### 4.1 Analysis Steps

```typescript
async function analyze(userId: string): Promise<AnalysisReport> {
  const recentSignals = await getRecentSignals(userId, { since: getLastImprovementTime(userId) });
  const currentPrefs = await getActivePreferences(userId);
  const contextEntries = await getContextEntries(userId);

  return {
    signalCount: recentSignals.length,
    patterns: detectPatterns(recentSignals),
    conflicts: detectConflicts(currentPrefs, recentSignals),
    gaps: detectContextGaps(contextEntries, recentSignals),
    redundant: findRedundantEntries(contextEntries),
    stale: findStaleEntries(contextEntries),
  };
}
```

### 4.2 Pattern Detection

```typescript
function detectPatterns(signals: NormalizedSignal[]): Pattern[] {
  const patterns: Pattern[] = [];

  // Time-of-day patterns
  const timeGroups = groupByTimeOfDay(signals.filter(s => s.category === 'navigation'));
  for (const [timeRange, group] of Object.entries(timeGroups)) {
    if (group.length >= 5) { // minimum observations
      patterns.push({
        type: 'temporal_habit',
        description: `User consistently active during ${timeRange} (${group.length} observations)`,
        confidence: Math.min(0.5 + group.length * 0.05, 0.95),
        supportingSignals: group.map(s => s.id),
      });
    }
  }

  // Feature usage patterns
  const featureGroups = groupBy(signals.filter(s => s.category === 'feature_usage'), 'source');
  for (const [feature, group] of Object.entries(featureGroups)) {
    if (group.length >= 3) {
      patterns.push({
        type: 'feature_preference',
        description: `User frequently uses ${feature} (${group.length} times)`,
        confidence: Math.min(0.5 + group.length * 0.08, 0.9),
        supportingSignals: group.map(s => s.id),
      });
    }
  }

  // Preference reinforcement
  const prefSignals = signals.filter(s => s.category === 'preference');
  const prefGroups = groupBy(prefSignals, 'payload.key');
  for (const [key, group] of Object.entries(prefGroups)) {
    if (group.length >= 2) {
      patterns.push({
        type: 'preference_reinforcement',
        description: `Preference "${key}" reinforced ${group.length} times`,
        confidence: 0.9,
        supportingSignalIds: group.map(s => s.id),
      });
    }
  }

  return patterns;
}
```

### 4.3 Conflict Detection

```typescript
function detectConflicts(
  prefs: Preference[],
  signals: NormalizedSignal[]
): Conflict[] {
  const conflicts: Conflict[] = [];

  // Check for contradictory active preferences
  const prefMap = new Map(prefs.map(p => [p.pref_key, p]));
  for (const [key, pref] of prefMap) {
    const opposite = prefMap.get(`!${key}`); // convention: ! prefix = negation
    if (opposite && pref.is_active && opposite.is_active) {
      conflicts.push({
        type: 'contradictory_preferences',
        description: `Both "${key}" and "!${key}" are active`,
        entries: [pref.id, opposite.id],
        resolution: 'keep_higher_confidence',
      });
    }
  }

  // Check for signals contradicting active preferences
  const correctionSignals = signals.filter(s => s.signalType === 'correction');
  for (const signal of correctionSignals) {
    const correctedKey = extractKeyFromCorrection(signal);
    const activePref = prefMap.get(correctedKey);
    if (activePref && activePref.is_active) {
      conflicts.push({
        type: 'correction_vs_preference',
        description: `User corrected "${correctedKey}" — conflicts with active preference`,
        entries: [activePref.id],
        signal: signal.id,
        resolution: 'apply_correction', // corrections always win
      });
    }
  }

  return conflicts;
}
```

---

## 5. Action Planner

Converts analysis results into a concrete list of proposed actions.

### 5.1 Action Types

| Action | Description | Auto-Apply? | HITL Required? |
|--------|-------------|-------------|----------------|
| `new_preference` | Create a new preference from detected pattern | If confidence > 0.85 | If confidence 0.5-0.85 |
| `update_preference` | Strengthen or modify existing preference | If confidence > 0.9 | If confidence 0.6-0.9 |
| `deactivate_preference` | Mark a preference as superseded | Never | Always |
| `merge_signals` | Consolidate redundant signals into pattern | Yes | No |
| `prune_context` | Remove low-importance, low-access context entries | If importance < 0.2 | If importance 0.2-0.5 |
| `generate_summary` | Create/update weekly summary markdown | Yes | No |
| `re_embed` | Re-embed context entries after content changes | Yes | No |

### 5.2 Planning Logic

```typescript
function planActions(analysis: AnalysisReport): PlannedAction[] {
  const actions: PlannedAction[] = [];

  // Handle conflicts first (highest priority)
  for (const conflict of analysis.conflicts) {
    if (conflict.resolution === 'apply_correction') {
      actions.push({
        type: 'update_preference',
        target: conflict.entries[0],
        newValue: extractCorrection(conflict.signal),
        confidence: 1.0,
        autoApply: true, // corrections always auto-applied
        reason: `User correction: ${conflict.description}`,
      });
    }
  }

  // Handle patterns
  for (const pattern of analysis.patterns) {
    const existingPref = findExistingPreference(pattern);
    if (!existingPref && pattern.confidence > 0.5) {
      actions.push({
        type: 'new_preference',
        key: patternToPreferenceKey(pattern),
        value: patternToPreferenceValue(pattern),
        confidence: pattern.confidence,
        autoApply: pattern.confidence > 0.85,
        reason: pattern.description,
      });
    } else if (existingPref && pattern.confidence > existingPref.confidence) {
      actions.push({
        type: 'update_preference',
        target: existingPref.id,
        confidence: pattern.confidence,
        autoApply: pattern.confidence > 0.9,
        reason: `Reinforced: ${pattern.description}`,
      });
    }
  }

  // Handle redundancy
  if (analysis.redundant.length > 10) {
    actions.push({
      type: 'merge_signals',
      targets: analysis.redundant.map(e => e.id),
      autoApply: true,
      reason: `${analysis.redundant.length} redundant context entries detected`,
    });
  }

  // Handle staleness
  for (const stale of analysis.stale) {
    actions.push({
      type: 'prune_context',
      target: stale.id,
      autoApply: stale.importance < 0.2,
      reason: `Stale entry: last accessed ${stale.last_accessed}, importance ${stale.importance}`,
    });
  }

  return actions;
}
```

---

## 6. HITL Gate (Human-In-The-Loop)

Actions that aren't auto-applied go through the HITL gate.

### 6.1 HITL Flow

```
Planned Action (not auto-applied)
    │
    ▼
┌──────────────────────────────────────┐
│  1. Log to improvement/YYYY-MM-DD.md │
│     "PROPOSED: [action] because [reason]" │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│  2. Notify user (in-app, not push)   │
│     "I noticed [pattern]. Should I   │
│      use this to [action]?"          │
│                                      │
│     [Yes] [No] [Tell me more]        │
└──────────────────┬───────────────────┘
                   │
         ┌─────────┼─────────┐
         │         │         │
         ▼         ▼         ▼
    ┌────────┐ ┌────────┐ ┌────────┐
    │ Apply  │ │ Skip   │ │ Modify │
    │ Action │ │ Action │ │ Action │
    └────────┘ └────────┘ └────────┘
```

### 6.2 HITL Implementation

```typescript
async function hitlGate(action: PlannedAction): Promise<'approved' | 'rejected' | 'modified'> {
  // Log the proposal
  await logImprovementProposal(action);

  // Show in-app notification (non-blocking)
  const result = await showContextProposal({
    title: action.reason,
    description: `I'd like to ${action.type}: ${formatActionDetail(action)}`,
    options: ['Yes', 'No', 'Tell me more'],
  });

  if (result === 'Yes') {
    return 'approved';
  } else if (result === 'No') {
    // Log the rejection for learning
    await logRejection(action);
    return 'rejected';
  } else {
    // "Tell me more" — show details, then re-prompt
    await showActionDetails(action);
    return hitlGate(action); // recursive, but only one level deep
  }
}
```

### 6.3 Auto-Apply Configuration

Users can configure their HITL preference:

```typescript
interface HITLConfig {
  mode: 'always_ask' | 'high_confidence_auto' | 'full_auto';
  confidenceThreshold: number; // default: 0.85
  notifyOnAutoApply: boolean;  // show brief toast even for auto-applied
}

// Default: 'high_confidence_auto' with threshold 0.85
```

---

## 7. Executor Module

Applies approved actions to the storage layer.

```typescript
async function executeAction(action: PlannedAction): Promise<void> {
  switch (action.type) {
    case 'new_preference':
      await db.run(`
        INSERT INTO preferences (id, user_id, pref_key, pref_value, source_signal_ids, confidence, is_active, first_seen_at, updated_at)
        VALUES (?, ?, ?, ?, ?, ?, 1, ?, ?)
      `, [crypto.randomUUID(), userId, action.key, action.value,
          JSON.stringify(action.supportingSignals), action.confidence,
          now(), now()]);
      break;

    case 'update_preference':
      // Soft-delete old, insert new
      await db.run(`UPDATE preferences SET is_active = 0, updated_at = ? WHERE id = ?`, [now(), action.target]);
      // ... insert updated version
      break;

    case 'merge_signals':
      // Consolidate into a pattern context entry
      const patternText = await consolidateSignals(action.targets);
      const entryId = crypto.randomUUID();
      await db.run(`
        INSERT INTO context_entries (id, user_id, entry_type, content, source_ids, importance, created_at, updated_at)
        VALUES (?, ?, 'pattern', ?, ?, ?, ?, ?)
      `, [entryId, userId, patternText, JSON.stringify(action.targets),
          0.7, now(), now()]);
      // Re-embed
      await maybeEmbed({ id: entryId, payload: patternText });
      break;

    case 'prune_context':
      await db.run(`UPDATE context_entries SET importance = 0, updated_at = ? WHERE id = ?`, [now(), action.target]);
      break;
  }

  // Log to improvement log
  await db.run(`
    INSERT INTO self_improvement_log (id, user_id, trigger_type, analysis_summary, action_taken, action_type, affected_ids, user_approved, created_at)
    VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
  `, [crypto.randomUUID(), userId, 'signal_threshold', action.reason,
      formatActionDetail(action), action.type,
      JSON.stringify(action.targets || [action.target]),
      action.autoApply ? null : 1, now()]);
}
```

---

## 8. Verifier Module

After applying improvements, monitor whether they actually helped.

```typescript
async function verifyImprovements(userId: string, appliedActions: PlannedAction[]): Promise<void> {
  // Wait 24h after applying, then check:
  // 1. Did the user interact with personalized features more?
  // 2. Did correction signals decrease? (fewer corrections = better context)
  // 3. Did feedback signals improve?

  const metrics = await getPersonalizationMetrics(userId, { since: getDayAfterLastImprovement() });

  for (const action of appliedActions) {
    const before = action.expectedMetrics;
    const after = metrics;

    if (after.correctionRate < before.correctionRate) {
      // Improvement verified
      await logVerification(action, 'improved');
    } else if (after.correctionRate > before.correctionRate * 1.2) {
      // Got worse — flag for review (don't auto-revert)
      await logVerification(action, 'degraded');
      await notifyUser(`I tried to ${action.type} but it seems to have made suggestions worse. I'll be more cautious about this in the future.`);
    }
  }
}
```

---

## 9. Self-Improvement Cycle Diagram

```
    ┌─────────────────────────────────────────────────────────────┐
    │                                                             │
    │   ┌──────────┐     ┌──────────┐     ┌──────────────────┐   │
    │   │ TRIGGER   │     │ ANALYZE  │     │ PLAN ACTIONS     │   │
    │   │           │     │          │     │                  │   │
    │   │ ·50 signals│────▶│ ·Patterns│────▶│ ·New prefs       │   │
    │   │ ·24h elapsed│    │ ·Conflicts│    │ ·Update prefs    │   │
    │   │ ·User request│   │ ·Gaps    │     │ ·Merge signals   │   │
    │   │ ·Conflict │     │ ·Redundant│    │ ·Prune context   │   │
    │   └──────────┘     └──────────┘     └────────┬─────────┘   │
    │                                              │              │
    │                                              ▼              │
    │                                   ┌──────────────────────┐  │
    │                                   │ HITL GATE            │  │
    │                                   │                      │  │
    │                                   │ Auto-apply if conf>0.85│ │
    │                                   │ Ask user if conf 0.5-0.85│ │
    │                                   │ Always ask for deletions│ │
    │                                   └────────┬─────────────┘  │
    │                                            │                │
    │                                            ▼                │
    │                                   ┌──────────────────────┐  │
    │                                   │ EXECUTE              │  │
    │                                   │                      │  │
    │                                   │ ·Update SQLite        │  │
    │                                   │ ·Re-embed vectors     │  │
    │                                   │ ·Update markdown      │  │
    │                                   └────────┬─────────────┘  │
    │                                            │                │
    │                                            ▼                │
    │                                   ┌──────────────────────┐  │
    │                                   │ VERIFY (24h later)   │  │
    │                                   │                      │  │
    │                                   │ ·Correction rate ↓?  │  │
    │                                   │ ·Feedback score ↑?   │  │
    │                                   │ ·Flag if degraded    │  │
    │                                   └────────┬─────────────┘  │
    │                                            │                │
    │                                            ▼                │
    │                                   ┌──────────────────────┐  │
    │                                   │ AUDIT LOG            │  │
    │                                   │                      │  │
    │                                   │ ·SQLite improvement_log│ │
    │                                   │ ·Markdown daily log   │  │
    │                                   │ ·User notification    │  │
    │                                   └──────────────────────┘  │
    │                                                             │
    └─────────────────────────────────────────────────────────────┘
```

---

## 10. Orchestration Summary

The full self-improvement orchestration flow:

```
App Usage → Signals → SQLite + Vectors + Markdown
                                    │
                          (trigger condition met)
                                    │
                                    ▼
                          Read & Analyze Signals
                                    │
                                    ▼
                          Detect Patterns & Conflicts
                                    │
                                    ▼
                          Plan Improvement Actions
                                    │
                          ┌─────────┴─────────┐
                          │                   │
                     High Confidence    Low Confidence
                     (>0.85)            (0.5-0.85)
                          │                   │
                          ▼                   ▼
                     Auto-Apply          HITL Approval
                          │                   │
                          └─────────┬─────────┘
                                    │
                                    ▼
                          Execute Changes
                                    │
                                    ▼
                          Log to Audit Trail
                                    │
                                    ▼
                          Verify (24h later)
                                    │
                                    ▼
                          Report to User
```

---

## Related

- [[architecture|Architecture]] — where the engine fits in the system
- [[context-pipeline|Context Pipeline]] — what feeds the engine
- [[retrieval-strategy|Retrieval Strategy]] — what the engine improves
- [[storage-schema|Storage Schema]] — tables the engine reads/writes
