# Global Memory Implementation in Cherry Studio

## Overview

The global memory feature in Cherry Studio enables persistent memory capabilities across conversations and assistants. When enabled, it allows the application to store, retrieve, and utilize conversation memories and facts to enhance AI interactions.

## Architecture

### Core Components

1. **Memory Store (Redux)**
   - Located in `src/renderer/src/store/memory.ts`
   - Manages the global memory state including configuration and enabled status
   - Provides selectors and actions for memory management

2. **Memory Service**
   - Renderer process: `src/renderer/src/services/MemoryService.ts`
   - Main process: `src/main/services/memory/MemoryService.ts`
   - Handles all memory operations through IPC communication

3. **Memory Processor**
   - Processes conversation memories and extracts facts
   - Manages vector embeddings for semantic search

## Implementation Details

### 1. Global Memory State Management

The global memory state is managed through Redux with the following key elements:

```typescript
interface MemoryState {
  memoryConfig: MemoryConfig
  currentUserId: string
  globalMemoryEnabled: boolean // The key flag controlling memory functionality
}
```

The `globalMemoryEnabled` flag acts as a master switch that controls whether memory features are active throughout the application.

### 2. Memory Retrieval and Injection into Context

The global memory mechanism retrieves and injects memory into context during AI conversations when all of the following conditions are met:

1. Global memory is enabled (`globalMemoryEnabled` flag is true in the Redux store)
2. The specific assistant has memory enabled (`assistant.enableMemory` is true)
3. A conversation is actively being processed

#### When Memory Retrieval is Triggered

Memory retrieval is triggered by user actions that initiate AI conversations:

1. **Sending a Message**: When a user sends a message in a conversation with an assistant that has global memory and assistant memory enabled
2. **Generating a Response**: When the AI generates a response and needs to access historical context
3. **Tool Usage**: When the AI decides to use the `builtin_memory_search` tool during its reasoning process

#### The Retrieval Process

The retrieval process occurs in real-time during conversation processing:

1. **Tool Configuration**: During the `transformParams` phase in the search orchestration plugin, if both global memory and assistant memory are enabled, the `builtin_memory_search` tool is added to the AI request parameters
2. **AI Decision Making**: The AI determines when to use the memory search tool based on the conversation context and system prompts
3. **Tool Execution**: When the AI invokes the `builtin_memory_search` tool, it passes a query string to search for relevant memories
4. **Vector Similarity Search**: The MemorySearchTool executes a search against the SQLite database using vector similarity search to find relevant memories
5. **Context Injection**: Retrieved memories are returned to the AI as tool results and incorporated into the response generation process

This retrieval and injection process is controlled by the global memory flag acting as a master switch, ensuring memory functionality can be completely disabled application-wide when needed.

### 2. Enabling Global Memory

Global memory can be toggled in the Memory Settings page:

1. User navigates to Settings → Memory
2. Toggles the "Global Memory" switch
3. The `setGlobalMemoryEnabled` action updates the Redux store
4. The setting is persisted in the application state

### 3. Memory Operations Control

The global memory flag affects several key areas:

#### Memory Search Tool
In `src/renderer/src/aiCore/tools/MemorySearchTool.ts`:
```typescript
const globalMemoryEnabled = selectGlobalMemoryEnabled(store.getState())
if (!globalMemoryEnabled) {
  return [] // Short-circuits if global memory is disabled
}
```

#### Search Orchestration Plugin
In `src/renderer/src/aiCore/plugins/searchOrchestrationPlugin.ts`:
```typescript
// During memory storage
const globalMemoryEnabled = selectGlobalMemoryEnabled(store.getState())
if (!globalMemoryEnabled || !assistant.enableMemory) {
  return // Skip memory processing entirely
}

// During tool configuration
const globalMemoryEnabled = selectGlobalMemoryEnabled(store.getState())
if (globalMemoryEnabled && assistant.enableMemory) {
  params.tools['builtin_memory_search'] = memorySearchTool(assistant.id)
}
```

#### Assistant Memory Settings
In `src/renderer/src/pages/settings/AssistantSettings/AssistantMemorySettings.tsx`:
```typescript
const isMemoryEnabled = globalMemoryEnabled && isMemoryConfigured
// Memory toggle is disabled if global memory is off
```

### 4. Data Flow

1. **Configuration**: Memory configuration is stored in Redux and persisted
2. **Activation**: When global memory is enabled, memory tools become available
3. **Collection**: Conversations are processed to extract facts when global memory is enabled
4. **Storage**: Memories are stored in a SQLite database in the main process
5. **Retrieval**: Relevant memories are retrieved during conversations using vector similarity search
6. **Integration**: Memories are injected into AI prompts to provide context

## Key Features Controlled by Global Memory

1. **Conversation Memory Storage**: Automatic storage of conversation snippets when users engage in conversations with assistants that have memory enabled
2. **Fact Extraction**: Extraction of key facts from conversations during the memory processing phase after each user message
3. **Semantic Search**: Vector-based search through stored memories triggered by AI tool calls during conversation processing
4. **Context Enhancement**: Providing relevant historical context to AI responses when the AI determines that past memories are relevant to the current conversation
5. **User-Specific Memory**: Separate memory contexts for different users, ensuring privacy and personalized experiences

## Example Conversation Flow with Memory Retrieval

Here's an example of how the memory retrieval process works during a conversation:

### Initial Conversation (Storage Phase)

**User**: "Hi, my name is John and I'm working on a React project."

**Assistant**: "Nice to meet you, John! How can I help you with your React project?"

*Behind the scenes - Memory Storage*:
1. Global memory is enabled and assistant has memory enabled
2. After the exchange, the system processes the conversation to extract facts
3. Facts extracted: `{"name": "John", "project": "React"}`
4. These facts are stored in the SQLite database with vector embeddings

### Follow-up Conversation (Retrieval Phase)

**User**: "Can you help me debug my React component? I mentioned it earlier."

**Assistant**: "I recall you mentioned working on a React project earlier. Let me help you debug your component. Can you share the code that's causing issues?"

*Behind the scenes - Memory Retrieval*:
1. User sends a message that triggers AI response generation
2. During search orchestration, the `builtin_memory_search` tool is added to the AI request
3. The AI decides to use the memory search tool with a query like "user's React project"
4. The MemorySearchTool executes a vector similarity search in the database
5. Relevant memories about John and his React project are retrieved
6. These memories are injected into the AI's context, allowing it to reference previous conversation history
7. The AI generates a response that acknowledges the previous conversation about the React project

This flow demonstrates how memory is automatically stored during conversations and retrieved when relevant to provide context-aware responses.

## Dependencies

For global memory to function properly, the following must be configured:
- Embedding model (for vector representations)
- LLM model (for fact extraction)
- Proper database initialization in the main process

## Impact of Disabling Global Memory

When `globalMemoryEnabled` is false:
- All memory operations are skipped
- Memory search tools are not added to AI requests
- Conversation processing bypasses memory extraction
- All assistants behave as if memory is disabled regardless of individual settings
- UI elements related to memory show warning messages

This design ensures that memory functionality can be completely disabled application-wide with a single toggle, providing users with control over data persistence and processing.