# Cursor Prompt — Phase 5: User Context Dashboard

## Context

Users need to see, understand, and control what the system knows about them. Build a context dashboard screen.

## Requirements

Create a Context Dashboard screen (`src/screens/context-dashboard.tsx`) with the following sections:

### 1. User Profile Card
- Display: display_name, member_since (created_at), app_version
- Edit button → opens simple form to update display_name
- Shows device_info (OS, model) as read-only

### 2. Active Preferences List
- List all active preferences from SQLite
- Format: `pref_key: pref_value (confidence: XX%)`
- Each preference has:
  - Edit button → inline edit of pref_value
  - Delete button → sets is_active=0 (soft delete)
  - "Why?" button → shows source signal count and first_seen date
- Add new preference button → simple form (key + value)

### 3. Context Entries Browser
- List context entries grouped by entry_type (fact, pattern, summary, goal, preference_detail)
- Each entry shows: content preview (first 100 chars), importance bar, last_accessed date
- Search/filter by entry_type
- Delete button on each entry (sets importance=0)
- Tap to expand full content

### 4. Improvement History
- List recent self-improvement actions from `self_improvement_log`
- Format: `YYYY-MM-DD HH:MM — [action_type] — [summary]`
- Color-coded: green=auto-applied, blue=user-approved, red=rejected
- Tap for full details

### 5. Usage Statistics
- Total signals captured (count from user_signals)
- Total context entries (count from context_entries)
- Active preferences (count from preferences)
- Days since first use
- Storage used: SQLite file size + markdown directory size
- API usage: estimated OpenRouter cost this month (track in a new `usage_log` table)

### 6. Controls
- **HITL Mode Toggle**: always_ask / high_confidence_auto / full_auto
- **Data Export**: button to export all context data as a single markdown file
- **Data Clear**: button to reset all context data (with confirmation dialog — "This will delete all learned preferences and context. This cannot be undone.")
- **Rebuild Index**: button to re-embed all context entries (useful after embedding model change)

### 7. Markdown File Viewer
- Browse and view markdown files from the wxpi-data directory
- List files by category: daily logs, improvements, summaries
- Tap to view full file content in a read-only screen

## UI/UX Constraints
- Uses React Native core components + Expo
- Navigation: tab-based or section-based scroll view
- All mutations (edit, delete, add) require confirmation
- Loading states for all async operations
- Empty states with helpful text ("No preferences yet — the system will learn as you use the app")
- Accessibility: all buttons have `accessibilityLabel`, all text has sufficient contrast

## Verification
1. Profile card displays correctly and is editable
2. Preferences list shows all active preferences with confidence
3. Context entries are grouped by type and searchable
4. Improvement history shows color-coded entries
5. HITL mode toggle persists to SQLite
6. Data export produces a valid markdown file
7. Data clear prompts confirmation and correctly soft-deletes all data
8. Rebuild index re-embeds all entries and shows progress
