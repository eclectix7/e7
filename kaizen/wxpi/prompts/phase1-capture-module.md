# Cursor Prompt — Phase 1: Capture Module

## Context

Wxpi is an Expo app building a context memory system. The database schema is already created (Phase 1a). Now we need the Capture Module — the entry point for all user signals.

## Requirements

Create a Capture Module that:

1. **Event Bus** (`src/capture/event-bus.ts`):
   - Implements a typed event bus using a simple pub/sub pattern
   - Defines the `SignalEvent` type: `{ type: SignalType, source: string, payload: Record<string, any>, timestamp: string, sessionId: string }`
   - Defines `SignalType` as: `'explicit_input' | 'behavior' | 'feedback' | 'correction' | 'system'`
   - Provides `emit(event: SignalEvent)` and `subscribe(handler: (event: SignalEvent) => void)` functions
   - Returns unsubscribe function from subscribe

2. **Capture Module** (`src/capture/capture-module.ts`):
   - Subscribes to the event bus on initialization
   - For each signal, runs the normalization pipeline:
     - Generates UUID v4 for signal ID
     - Validates signal type against enum
     - Classifies category based on source + payload (rule-based: settings→preference, chat→interaction, navigation→navigation, dwell→engagement, rating→feedback, feature→feature_usage, else→general)
     - Calculates confidence score: explicit_input=1.0, correction=1.0, feedback=0.8, behavior=0.5, system=0.3
     - Sanitizes payload (remove any potential PII like emails, phone numbers — replace with [REDACTED])
     - Normalizes timestamp to ISO 8601 UTC
   - Checks for deduplication (same user, category, payload hash, within 1 hour)
   - Routes to storage: SQLite write + markdown append (fire-and-forget, non-blocking)
   - All operations are async and non-blocking — the UI never waits

3. **Signal Emitter Helpers** (`src/capture/emitters.ts`):
   - `emitExplicitInput(source: string, payload: Record<string, any>)` — for form inputs, settings changes
   - `emitBehavior(source: string, payload: Record<string, any>)` — for navigation, dwell time, feature usage
   - `emitFeedback(source: string, payload: { target: string, rating: 'positive' | 'negative' })` — for likes/dislikes
   - `emitCorrection(source: string, payload: { original: string, corrected: string, field: string })` — for user corrections
   - Each helper automatically fills in timestamp, sessionId, and signal type

## File Structure

```
src/capture/
├── event-bus.ts        # Typed pub/sub event bus
├── capture-module.ts   # Main capture + normalization logic
├── emitters.ts         # Helper functions for emitting signals
├── types.ts            # Shared types (SignalEvent, SignalType, NormalizedSignal)
└── index.ts            # Public API
```

## Constraints

- TypeScript strict mode
- No external dependencies beyond what Expo provides
- All async operations must have error handling (catch + log, never throw to caller)
- Deduplication check should not block storage — store first, deduplicate later during self-improvement
- Include JSDoc comments on all public functions

## Verification

After implementation, verify:
1. Event bus correctly delivers signals to subscribers
2. Normalization produces valid NormalizedSignal objects
3. Category classification works for all source types
4. Confidence scores match the specified values
5. Signals are written to SQLite (check with a query)
6. Signals are appended to daily markdown log file
7. UI thread is never blocked (test with rapid signal emission)
