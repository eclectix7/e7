# Cursor Prompt — Phase 6: LLM-Powered Analysis

## Context

Phase 4's analyzer uses rule-based pattern detection. Phase 6 replaces/augments it with LLM-powered analysis for deeper, more nuanced pattern recognition.

## Requirements

Create an LLM-powered analysis module (`src/self-improvement/llm-analyzer.ts`):

### 1. LLM Analyzer Function
- `analyzeWithLLM(userId: string, signals: NormalizedSignal[]): Promise<LLMAnalysisResult>`
- Sends a batch of recent signals to OpenRouter for analysis
- Uses `anthropic/claude-sonnet-4` for quality (or `openai/gpt-4o-mini` for cost savings)
- Temperature: 0.2 (consistent, analytical)
- Max tokens: 2048

### 2. Analysis Prompt Template
```
You are a user behavior analyst for the Wxpi app. Analyze the following user signals and provide a structured analysis.

User Profile:
{display_name}, using app since {created_at}, {signal_count} total signals

Active Preferences:
{preferences_list}

Recent Signals (last {N} signals):
{signals_formatted}

Provide a JSON response with this exact structure:
{
  "patterns": [
    {
      "type": "temporal_habit|feature_preference|content_preference|interaction_style|goal_signal",
      "description": "human-readable description",
      "confidence": 0.0-1.0,
      "supportingSignalIds": ["id1", "id2"],
      "suggestedPreference": { "key": "...", "value": "..." }
    }
  ],
  "conflicts": [
    {
      "description": "what conflicts",
      "entries": ["pref_id1", "pref_id2"],
      "resolution": "keep_first|keep_second|ask_user"
    }
  ],
  "insights": [
    {
      "type": "behavioral_narrative|preference_evolution|engagement_trend",
      "description": "a paragraph of narrative insight about the user"
    }
  ],
  "suggestedActions": [
    {
      "action": "new_preference|update_preference|prune_context|generate_summary",
      "details": "specific action details",
      "confidence": 0.0-1.0
    }
  ]
}
```

### 3. Response Parsing
- Parse the JSON response from the LLM
- Validate the structure (check all required fields)
- If parsing fails, fall back to the rule-based analyzer from Phase 4
- Log the raw LLM response to markdown for debugging

### 4. Hybrid Analysis Strategy
- Run both rule-based and LLM analyzers
- Merge results: use rule-based for high-confidence patterns (temporal, feature usage), use LLM for nuanced insights (behavioral narratives, preference evolution)
- Deduplicate overlapping patterns (keep higher confidence)
- LLM insights are stored as context_entries with entry_type `'summary'`

### 5. Cost Control
- LLM analysis runs at most once per day per user
- Batch size: max 100 signals per analysis (sample if more)
- Track token usage in `usage_log` table
- If monthly budget exceeded, fall back to rule-based only

### 6. Usage Log Table
```sql
CREATE TABLE usage_log (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  operation_type TEXT NOT NULL,  -- 'embedding' | 'chat' | 'analysis'
  model TEXT NOT NULL,
  tokens_input INTEGER,
  tokens_output INTEGER,
  cost_usd REAL,
  created_at TEXT NOT NULL,
  INDEX idx_usage_user_time (user_id, created_at),
  INDEX idx_usage_type (operation_type, created_at)
);
```

## Verification
1. LLM analyzer returns valid JSON matching the expected structure
2. Fallback to rule-based works when LLM response is invalid
3. Hybrid analysis merges results without duplicates
4. Cost tracking correctly logs token usage and estimated cost
5. Rate limiting prevents more than 1 LLM analysis per day
6. Monthly budget cap falls back to rule-based when exceeded
