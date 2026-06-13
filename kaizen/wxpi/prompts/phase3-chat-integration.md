# Cursor Prompt — Phase 3: Chat Integration

## Context

The Retrieve Module can assemble context envelopes. Now we need the chat integration — the function that takes a user message, enriches it with context, and calls OpenRouter for a personalized response.

## Requirements

Create a Chat module that:

1. **OpenRouter Chat Client** (`src/chat/client.ts`):
   - `chat(messages: ChatMessage[], options: ChatOptions): Promise<string>` — calls OpenRouter `/chat/completions`
   - Default model: `openai/gpt-4o-mini`
   - Supports fallback: if primary model fails, try `anthropic/claude-sonnet-4`, then `openai/gpt-4o`
   - Reads API key from `expo-secure-store`
   - Implements retry with backoff (max 2 retries)
   - Returns the response text

2. **Context-Aware Chat** (`src/chat/context-chat.ts`):
   - `contextAwareChat(userId: string, userMessage: string): Promise<string>`
   - Assembles context envelope using Retrieve Module
   - Builds messages array:
     ```
     [
       { role: 'system', content: 'You are a helpful assistant for Wxpi. Use the following user context to personalize responses.\n\n{envelope}' },
       { role: 'user', content: userMessage }
     ]
     ```
   - Calls chat client
   - After response, emits a behavior signal: `{ source: 'chat_interaction', payload: { query_length, response_length } }`
   - Returns the response text

3. **UI Integration** (`src/chat/chat-screen.tsx`):
   - Basic chat screen component (React Native)
   - Text input + send button
   - Message list (user messages + assistant responses)
   - Loading indicator while waiting for response
   - Error display if chat fails
   - Each user message emits an explicit_input signal via the capture module

## File Structure

```
src/chat/
├── client.ts         # OpenRouter chat client
├── context-chat.ts   # Context-aware chat function
├── chat-screen.tsx   # Chat UI component
├── types.ts          # ChatMessage, ChatOptions
└── index.ts          # Public API
```

## Constraints
- TypeScript strict mode
- Chat must work offline (queue messages, send when online)
- System prompt must not exceed token budget
- User messages are limited to 2000 characters
- Response display supports markdown rendering (use `react-native-markdown-display`)

## Verification
1. `chat()` returns a valid response from OpenRouter
2. `contextAwareChat()` includes user context in the system prompt
3. Fallback model chain works (test by using invalid primary model)
4. Chat screen renders messages correctly
5. Behavior signal is emitted after each chat interaction
6. Offline messages are queued and sent when connection returns
