# Cursor Prompt — Phase 1: SQLite Schema

## Context

Wxpi is an Expo (React Native) app that needs a local-first context memory system. We're using `expo-sqlite` for structured storage. This is Phase 1 — we need to create the database schema and initialization logic.

## Requirements

Create a database module that:

1. Opens/creates an SQLite database named `wxpi-context.db` using `expo-sqlite`
2. Creates the following tables if they don't exist:
   - `user_profile` — single row per app install (id, display_name, created_at, updated_at, app_version, device_info JSON, config JSON)
   - `user_signals` — append-only signal log (id, user_id, signal_type, category, source, payload JSON, confidence, context_hash, created_at) with indexes on (user_id, created_at), (signal_type, category), (source)
   - `preferences` — normalized user preferences (id, user_id, pref_key, pref_value, source_signal_ids JSON, confidence, is_active, first_seen_at, updated_at) with unique constraint on (user_id, pref_key, is_active) and index on (user_id, is_active)
   - `context_entries` — semantic context chunks (id, user_id, entry_type, content, source_ids JSON, importance, access_count, last_accessed, created_at, updated_at) with indexes on (user_id, entry_type) and (importance DESC)
   - `self_improvement_log` — audit trail (id, user_id, trigger_type, analysis_summary, action_taken, action_type, affected_ids JSON, user_approved, created_at) with index on (user_id, created_at)
3. Creates a `context_embeddings` table for vector storage (id, entry_id, embedding JSON, model, created_at) with index on (entry_id)
4. Provides an `initializeDatabase()` function that runs on app startup
5. Provides a `getDatabase()` function that returns the database instance
6. Uses UUID v4 for all primary keys
7. Uses ISO 8601 UTC strings for all timestamps
8. Enables WAL mode for better concurrent read performance

## File Structure

Create: `src/database/schema.ts` — table creation SQL and initialization
Create: `src/database/index.ts` — public API (initializeDatabase, getDatabase, closeDatabase)

## Constraints

- Use `expo-sqlite` (not react-native-sqlite-storage — we want to stay within Expo's managed workflow)
- All SQL must use parameterized queries (no string interpolation)
- Include proper error handling with typed errors
- Write the code in TypeScript with strict typing
- Include JSDoc comments on all public functions

## Verification

After implementation, verify:
1. Database file is created in the app's document directory
2. All 6 tables are created with correct columns and types
3. All indexes are created
4. WAL mode is enabled
5. A test insert into user_profile succeeds
