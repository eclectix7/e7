# Cursor Prompt — Phase 4: HITL Gate

## Context

The Action Planner produces proposed actions. Non-auto-apply actions need human approval. Create the HITL (Human-In-The-Loop) gate.

## Requirements

Create a HITL Gate module (`src/self-improvement/hitl-gate.ts`):

1. **HITL Configuration**:
   ```typescript
   interface HITLConfig {
     mode: 'always_ask' | 'high_confidence_auto' | 'full_auto';
     confidenceThreshold: number; // default 0.85
     notifyOnAutoApply: boolean;  // show toast for auto-applied actions
   }
   ```
   - Stored in SQLite `user_profile.config` JSON blob
   - Default: `high_confidence_auto` with threshold 0.85

2. **HITL Gate Function**:
   - `hitlGate(action: PlannedAction, config: HITLConfig): Promise<'approved' | 'rejected' | 'modified'>`
   - If action.autoApply is true and config mode allows: return 'approved' immediately
   - If config mode is 'full_auto': return 'approved' for all actions
   - Otherwise: show in-app proposal UI and wait for user response

3. **Proposal UI Component** (`src/self-improvement/proposal-ui.tsx`):
   - Non-blocking in-app notification (not a push notification)
   - Shows: title (action reason), description (what will happen), options: [Yes] [No] [Tell me more]
   - "Tell me more" shows expanded details, then re-prompts
   - Returns user's choice
   - Uses React Native Modal or a bottom sheet (use `@gorhom/bottom-sheet` if available, otherwise Modal)

4. **Logging**:
   - Every proposal (approved, rejected, or modified) is logged to the improvement markdown log
   - Format: `## HH:MM — [actionType] — [status]\nReason: ...\nUser decision: ...`

5. **Batch Proposals**:
   - If multiple actions need HITL approval, show them as a single batch (not one-by-one)
   - User can approve all, reject all, or review individually

## Verification
1. Auto-apply actions pass through without user interaction
2. HITL-required actions show the proposal UI
3. User's decision is correctly returned and logged
4. Batch proposals group multiple actions
5. HITL config from user_profile is respected
6. Rejection is logged and doesn't re-prompt for the same action
