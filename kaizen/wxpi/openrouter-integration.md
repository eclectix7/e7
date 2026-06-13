---
title: Wxpi — OpenRouter Integration
type: integration-doc
status: design
tags:
  - wxpi
  - openrouter
  - api
  - embeddings
  - llm
---

# Wxpi — OpenRouter Integration

How Wxpi communicates with OpenRouter for embeddings and LLM inference.

---

## 1. API Overview

Wxpi uses OpenRouter as its **sole external API**. All calls are standard HTTP POST requests — no SDK required.

```
Wxpi App ──HTTPS──▶ https://openrouter.ai/api/v1/
                      ├── /embeddings          (vector generation)
                      ├── /chat/completions   (LLM inference)
                      └── /models             (model discovery, optional)
```

---

## 2. Authentication

Store the OpenRouter API key securely using Expo's `SecureStore`:

```typescript
import * as SecureStore from 'expo-secure-store';

async function getApiKey(): Promise<string> {
  const key = await SecureStore.getItemAsync('openrouter_api_key');
  if (!key) throw new Error('OpenRouter API key not configured');
  return key;
}

// Set once during app setup
async function setApiKey(key: string): Promise<void> {
  await SecureStore.setItemAsync('openrouter_api_key', key);
}
```

---

## 3. Embedding API

### 3.1 Endpoint

```
POST https://openrouter.ai/api/v1/embeddings
```

### 3.2 Recommended Model

| Model | Dimensions | Cost | Speed | Notes |
|-------|-----------|------|-------|-------|
| `openai/text-embedding-3-small` | 1536 | $0.02/1M tokens | Fast | **Recommended** — best cost/quality ratio |
| `openai/text-embedding-3-large` | 3072 | $0.13/1M tokens | Medium | Higher quality, 2x dimensions |
| `openai/text-embedding-ada-002` | 1536 | $0.10/1M tokens | Fast | Legacy, being deprecated |

**Recommendation:** Use `text-embedding-3-small`. The quality difference vs large is negligible for user context, and it's 6.5x cheaper.

### 3.3 Implementation

```typescript
const OPENROUTER_BASE = 'https://openrouter.ai/api/v1';

async function embedText(text: string): Promise<number[]> {
  const response = await fetch(`${OPENROUTER_BASE}/embeddings`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${await getApiKey()}`,
      'Content-Type': 'application/json',
      'HTTP-Referer': 'https://wxpi.app',       // optional, for OpenRouter analytics
      'X-Title': 'Wxpi Context Memory',          // optional, for OpenRouter analytics
    },
    body: JSON.stringify({
      model: 'openai/text-embedding-3-small',
      input: text,
    }),
  });

  if (!response.ok) {
    const error = await response.text();
    throw new Error(`Embedding failed: ${response.status} — ${error}`);
  }

  const data = await response.json();
  return data.data[0].embedding;
}

// Batch embedding (more efficient for multiple texts)
async function embedBatch(texts: string[]): Promise<number[][]> {
  const response = await fetch(`${OPENROUTER_BASE}/embeddings`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${await getApiKey()}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      model: 'openai/text-embedding-3-small',
      input: texts,  // array of strings
    }),
  });

  const data = await response.json();
  return data.data.map((d: any) => d.embedding);
}
```

### 3.4 Error Handling

```typescript
async function embedWithRetry(text: string, maxRetries = 3): Promise<number[]> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await embedText(text);
    } catch (err) {
      if (attempt === maxRetries) throw err;

      // Exponential backoff: 1s, 2s, 4s
      const delay = Math.pow(2, attempt - 1) * 1000;
      await new Promise(r => setTimeout(r, delay));
    }
  }
}
```

---

## 4. Chat Completions API

### 4.1 When Wxpi Calls the LLM

| Use Case | Model Suggestion | Max Tokens | Temperature |
|----------|-----------------|------------|-------------|
| User chat / assistant | `anthropic/claude-sonnet-4` or `openai/gpt-4o-mini` | 1024 | 0.7 |
| Context summarization | `openai/gpt-4o-mini` | 512 | 0.3 |
| Pattern analysis (self-improvement) | `anthropic/claude-sonnet-4` | 2048 | 0.2 |
| Quick suggestions | `openai/gpt-4o-mini` | 256 | 0.5 |

### 4.2 Implementation

```typescript
interface ChatMessage {
  role: 'system' | 'user' | 'assistant';
  content: string;
}

async function chat(
  messages: ChatMessage[],
  options: {
    model?: string;
    maxTokens?: number;
    temperature?: number;
  } = {}
): Promise<string> {
  const {
    model = 'openai/gpt-4o-mini',
    maxTokens = 1024,
    temperature = 0.7,
  } = options;

  const response = await fetch(`${OPENROUTER_BASE}/chat/completions`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${await getApiKey()}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      model,
      messages,
      max_tokens: maxTokens,
      temperature,
    }),
  });

  if (!response.ok) {
    const error = await response.text();
    throw new Error(`Chat failed: ${response.status} — ${error}`);
  }

  const data = await response.json();
  return data.choices[0].message.content;
}
```

### 4.3 Context-Enriched Chat Call

This is the main pattern — assemble context envelope, then call LLM:

```typescript
async function contextAwareChat(
  userId: string,
  userMessage: string
): Promise<string> {
  // 1. Assemble context envelope
  const envelope = await assembleEnvelope(userId, userMessage);

  // 2. Build messages with context
  const messages: ChatMessage[] = [
    {
      role: 'system',
      content: `You are a helpful assistant for the Wxpi app. Use the following user context to personalize your responses.\n\n${envelope.systemContext}`,
    },
    {
      role: 'user',
      content: userMessage,
    },
  ];

  // 3. Call LLM
  const response = await chat(messages, {
    model: 'openai/gpt-4o-mini',
    maxTokens: 1024,
    temperature: 0.7,
  });

  // 4. Store the interaction as a signal
  await captureModule.enqueue({
    type: 'behavior',
    source: 'chat_interaction',
    payload: { query_length: userMessage.length, response_length: response.length },
    timestamp: new Date().toISOString(),
    sessionId: getCurrentSessionId(),
  });

  return response;
}
```

---

## 5. Self-Improvement LLM Calls

The self-improvement engine uses the LLM for pattern analysis and summarization:

### 5.1 Pattern Analysis Prompt

```typescript
async function analyzePatternsWithLLM(signals: NormalizedSignal[]): Promise<string> {
  const signalSummary = signals.map(s =>
    `[${s.signalType}] ${s.category}: ${s.payload}`
  ).join('\n');

  const messages: ChatMessage[] = [
    {
      role: 'system',
      content: `You are a user behavior analyst. Analyze the following user signals and identify patterns, preferences, and potential conflicts. Output a structured JSON analysis with fields: patterns (array of {type, description, confidence, supportingSignalIds}), conflicts (array), gaps (array).`,
    },
    {
      role: 'user',
      content: `Analyze these ${signals.length} user signals:\n\n${signalSummary}`,
    },
  ];

  return await chat(messages, {
    model: 'anthropic/claude-sonnet-4',
    maxTokens: 2048,
    temperature: 0.2, // low temperature for consistent analysis
  });
}
```

### 5.2 Weekly Summary Generation

```typescript
async function generateWeeklySummary(
  userId: string,
  weekStart: string,
  weekEnd: string
): Promise<string> {
  const signals = await getSignalsInRange(userId, weekStart, weekEnd);
  const prefs = await getActivePreferences(userId);
  const improvements = await getImprovementsInRange(userId, weekStart, weekEnd);

  const messages: ChatMessage[] = [
    {
      role: 'system',
      content: `Generate a concise weekly user context summary in markdown. Include: usage patterns, confirmed preferences, new observations, and system changes. Keep it under 500 words.`,
    },
    {
      role: 'user',
      content: `
Week: ${weekStart} to ${weekEnd}
Signals: ${signals.length}
Active Preferences: ${JSON.stringify(prefs, null, 2)}
Improvements Applied: ${JSON.stringify(improvements, null, 2)}
Recent Signals Sample: ${JSON.stringify(signals.slice(-20), null, 2)}
      `,
    },
  ];

  const summary = await chat(messages, {
    model: 'openai/gpt-4o-mini',
    maxTokens: 1024,
    temperature: 0.3,
  });

  // Write to markdown
  const filePath = `${DOC_DIR}/summaries/weekly/${weekStart.slice(0, 7)}.md`;
  await FileSystem.writeAsStringAsync(filePath, summary);

  return summary;
}
```

---

## 6. Cost Management

### 6.1 Estimated Monthly Costs

| Operation | Frequency | Model | Est. Cost/Month |
|-----------|-----------|-------|-----------------|
| Embeddings | ~600/mo (20/day) | text-embedding-3-small | ~$0.01 |
| User chat | ~300/mo (10/day) | gpt-4o-mini | ~$0.15 |
| Self-improvement analysis | ~30/mo (1/day) | claude-sonnet-4 | ~$0.90 |
| Weekly summaries | ~4/mo | gpt-4o-mini | ~$0.02 |
| **Total** | | | **~$1.10/mo** |

### 6.2 Cost Controls

```typescript
// Rate limiting
const RATE_LIMITS = {
  embeddings: { perMinute: 20, perDay: 500 },
  chat: { perMinute: 10, perDay: 200 },
  analysis: { perMinute: 2, perDay: 10 },
};

// Budget cap (optional user setting)
const MONTHLY_BUDGET_USD = 5.00;

async function checkBudget(): Promise<boolean> {
  const spent = await getMonthlySpend();
  return spent < MONTHLY_BUDGET_USD;
}
```

### 6.3 Fallback Model Chain

If the primary model is unavailable or overloaded, fall back:

```typescript
const MODEL_FALLBACKS: Record<string, string[]> = {
  'openai/gpt-4o-mini': ['openai/gpt-4o', 'anthropic/claude-sonnet-4'],
  'anthropic/claude-sonnet-4': ['openai/gpt-4o', 'openai/gpt-4o-mini'],
  'openai/text-embedding-3-small': ['openai/text-embedding-ada-002'], // last resort
};

async function chatWithFallback(
  messages: ChatMessage[],
  primaryModel: string,
  options: any
): Promise<string> {
  const models = [primaryModel, ...(MODEL_FALLBACKS[primaryModel] || [])];

  for (const model of models) {
    try {
      return await chat(messages, { ...options, model });
    } catch (err) {
      console.warn(`Model ${model} failed, trying fallback...`);
      continue;
    }
  }
  throw new Error('All models failed');
}
```

---

## 7. Request Headers

Always include these headers for OpenRouter:

```typescript
const headers = {
  'Authorization': `Bearer ${apiKey}`,
  'Content-Type': 'application/json',
  'HTTP-Referer': 'https://wxpi.app',    // your app URL (optional but recommended)
  'X-Title': 'Wxpi Context Memory',       // your app name (optional but recommended)
};
```

The `HTTP-Referer` and `X-Title` headers help OpenRouter identify your app in their analytics. They don't affect billing or routing.

---

## 8. Expo-Specific Considerations

### 8.1 Network Handling

```typescript
import NetInfo from '@react-native-community/netinfo';

async function safeApiCall<T>(fn: () => Promise<T>): Promise<T> {
  const netState = await NetInfo.fetch();
  if (!netState.isConnected) {
    throw new Error('No network connection');
  }
  return fn();
}
```

### 8.2 Background API Calls

For self-improvement cycles that run in the background:

```typescript
import * as TaskManager from 'expo-task-manager';
import * as BackgroundFetch from 'expo-background-fetch';

const SELF_IMPROVEMENT_TASK = 'self-improvement-cycle';

TaskManager.defineTask(SELF_IMPROVEMENT_TASK, async () => {
  try {
    await runSelfImprovementCycle();
    return BackgroundFetch.BackgroundFetchResult.NewData;
  } catch (err) {
    return BackgroundFetch.BackgroundFetchResult.Failed;
  }
});

// Register on app startup
async function registerBackgroundTasks() {
  await BackgroundFetch.registerTaskAsync(SELF_IMPROVEMENT_TASK, {
    minimumInterval: 60 * 60 * 6, // 6 hours minimum
    stopOnTerminate: false,
    startOnBoot: true,
  });
}
```

---

## Related

- [[architecture|Architecture]] — where OpenRouter calls fit in the system
- [[context-pipeline|Context Pipeline]] — embedding calls during capture
- [[retrieval-strategy|Retrieval Strategy]] — embedding calls during retrieval
- [[self-improvement-engine|Self-Improvement Engine]] — LLM calls for analysis
