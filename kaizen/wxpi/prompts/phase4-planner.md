# Cursor Prompt — Phase 4: Action Planner

## Context

The Analyzer produces an analysis report. The Action Planner converts that report into a concrete list of proposed actions with confidence scores and HITL requirements.

## Requirements

Create an Action Planner module (`src/self-improvement/planner.ts`):

1. **Main Planning Function**:
   - `planActions(analysis: AnalysisReport): PlannedAction[]`
   - Processes conflicts first (highest priority), then patterns, then redundancy, then staleness
   - Returns ordered list of actions

2. **Action Types**:
   ```typescript
   interface PlannedAction {
     type: 'new_preference' | 'update_preference' | 'deactivate_preference' | 'merge_signals' | 'prune_context' | 'generate_summary' | 're_embed';
     // For new_preference:
     key?: string;
     value?: string;
     // For update/deactivate/prune:
     target?: string;
     targets?: string[];
     newValue?: string;
     // Common:
     confidence: number;
     autoApply: boolean;
     reason: string;
     supportingSignalIds?: string[];
   }
   ```

3. **Planning Rules**:
   - **Corrections**: Always auto-apply (confidence 1.0). Update the corrected preference immediately.
   - **New preferences from patterns**: Auto-apply if confidence > 0.85. Require HITL if 0.5-0.85.
   - **Preference updates**: Auto-apply if confidence > 0.9. Require HITL if 0.6-0.9.
   - **Preference deactivation**: Never auto-apply. Always require HITL.
   - **Signal merging**: Always auto-applies (consolidation, not destructive).
   - **Context pruning**: Auto-apply if importance < 0.2. Require HITL if 0.2-0.5.
   - **Summary generation**: Always auto-applies.
   - **Re-embedding**: Always auto-applies.

4. **Deduplication**: If two actions target the same preference, keep only the higher-confidence one.

## Verification
1. Correction conflicts produce update_preference actions with autoApply=true
2. High-confidence patterns (>0.85) produce new_preference with autoApply=true
3. Medium-confidence patterns (0.5-0.85) produce new_preference with autoApply=false
4. Redundant entries produce merge_signals actions
5. Stale entries with importance < 0.2 produce prune_context with autoApply=true
6. Duplicate actions are deduplicated
