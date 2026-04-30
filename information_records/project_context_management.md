# How This Project Manages Context

This project manages context through several coordinated layers, each solving a different problem.

## 1) Conversation Context for Model Requests

The main conversation-context pipeline lives in `src/renderer/src/services/ConversationService.ts`.

- `ConversationService.filterMessagesPipeline(messages, contextCount)` prepares messages before model calls.
- The filtering order is deliberate:
  1. Keep only messages after the latest `clear` marker (`filterAfterContextClearMessages`).
  2. Keep useful assistant variants and remove redundant grouped outputs (`filterUsefulMessages`).
  3. Remove assistant messages that only contain errors (and remove related user prompts) (`filterErrorOnlyMessagesWithRelated`).
  4. Remove trailing assistant messages (`filterLastAssistantMessage`).
  5. Remove adjacent duplicate user turns (`filterAdjacentUserMessaegs`).
  6. Trim by window size using `takeRight(..., contextCount + 2)`.
  7. Re-apply clear-boundary logic, remove empty messages, and ensure the sequence starts from a user message.
- `prepareMessagesForModel` then converts filtered UI messages into SDK model messages (`convertMessagesToSdkMessages`).

Why this matters: it keeps prompt history focused, valid, and predictable while avoiding noisy or malformed turn sequences.

## 2) Context Size Is Assistant-Scoped

Context size is not global; it is configured per assistant.

- Assistant settings include `contextCount` (see assistant settings usage in `ConversationService` and `MessagesService`).
- `MessagesService.getContextCount` computes current context usage and max configured size.
- The code also handles special “max/unlimited” constants when needed.

Why this matters: different assistants can have different history depth, balancing quality and cost.

## 3) “Clear Context” Is a Hard Boundary

The `clear` message type is treated as a strict reset point.

- `filterAfterContextClearMessages` finds the latest `message.type === 'clear'` and slices everything before it.
- Any content before that boundary is excluded from new context windows.

Why this matters: users can explicitly reset model memory within a topic without deleting all messages.

## 4) Token-Oriented Context Estimation

Token estimation is handled in `src/renderer/src/services/TokenService.ts`.

- `estimateHistoryTokens` applies context filtering/windowing before estimating token usage.
- It combines existing usage metadata (when available) with computed estimates for messages/files.
- This aligns estimated token load with the same context rules used for actual requests.

Why this matters: token budgeting remains consistent with real prompt construction.

## 5) Long-Term Memory Context (Persistent, User-Scoped)

Persistent memory is managed by `src/renderer/src/services/MemoryService.ts`.

- Memory operations are routed via IPC (`window.api.memory.*`) to the main process.
- The service tracks `currentUserId` and attaches it to list/search/add operations.
- This memory layer is distinct from the short-term chat window.

Why this matters: the app can retain user-level memory across sessions while still limiting immediate prompt context.

## 6) UI Context via React Context Providers

Separate from LLM/chat context, UI runtime context is managed through React providers under `src/renderer/src/context/`.

Examples include:
- `ThemeProvider.tsx`
- `AntdProvider.tsx`
- `CodeStyleProvider.tsx`
- `NotificationProvider.tsx`
- `MessageEditingContext.tsx`

These providers handle application state concerns (theme, style systems, notifications, editing state), not model prompt history.

## Practical Summary

This project intentionally separates context into four domains:

1. Short-term conversation context (filtered + windowed message history),
2. Token-budget context control (estimation aligned with filtering),
3. Persistent user memory context (IPC-backed long-term memory),
4. UI runtime context (React providers).

That separation improves reliability (cleaner prompt inputs), controllability (assistant-specific history depth), and scalability (persistent memory without overloading immediate context windows).
