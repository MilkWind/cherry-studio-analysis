# Cherry Studio Architecture: User Input to AI Model Response

## Overview

This document explains each layer in the end-to-end flow of a user message in Cherry Studio, from the React renderer UI to the AI model API call.

```
User Input (Renderer)
  └─> Message Conversion
  └─> Plugin Pipeline
  └─> Provider/Model Resolution
  └─> AI SDK Runtime
  └─> Provider SDK
  └─> Network → AI Models
```

---

## Layer 1: Renderer (React UI)

**Purpose:** Capture user input and dispatch it into the data flow.

**Key files:**
- `src/renderer/src/pages/home/Chat.tsx` — Main chat container, renders `<Messages>` and `<Inputbar>`
- `src/renderer/src/pages/home/Inputbar/Inputbar.tsx` — User input area with text entry, file upload, and send logic
- `src/renderer/src/pages/home/Messages/Messages.tsx` — Renders the conversation message list

**How it works:**
1. `Inputbar.tsx` handles user typing, file attachments, and mentions.
2. On send, `sendMessage()` (line ~235) uploads files via `FileManager.uploadFiles()`, builds a `MessageInputBaseParams` object with `{ assistant, topic, content, files, mentions, usage }`.
3. Calls `MessagesService.getUserMessage()` to create `Message` and `MessageBlock[]` objects.
4. Dispatches the Redux thunk: `dispatch(sendMessage(message, blocks, assistant, topic.id))`.

**Key types:**
- `Message` — `{ id, topicId, assistantId, role, status, blocks, model, mentions, ... }`
- `MessageBlock` — typed blocks: `MAIN_TEXT`, `THINKING`, `IMAGE`, `CODE`, `TOOL`, `FILE`, `ERROR`, etc.

**Entry point:** `src/renderer/src/store/thunk/messageThunk.ts` — the `sendMessage` thunk (line ~968).

---

## Layer 2: Message Conversion

**Purpose:** Convert Cherry Studio internal message types into AI SDK-compatible message format, apply context filters, and build final API parameters.

**Key files:**
- `src/renderer/src/services/ConversationService.ts` — `prepareMessagesForModel()` (line ~38)
- `src/renderer/src/aiCore/prepareParams/messageConverter.ts` — `convertMessagesToSdkMessages()` (line ~318)
- `src/renderer/src/aiCore/prepareParams/parameterBuilder.ts` — `buildStreamTextParams()` (line ~93)
- `src/renderer/src/utils/messageUtils/filters.ts` — Filter pipeline for context messages

**How it works:**
1. `ConversationService.prepareMessagesForModel()` retrieves all messages for the current topic, runs them through a **filter pipeline** (8 filters: context-clear, useful-only, error-only, trailing-assistant, adjacent-user, context-limit, empty, user-role-start).
2. `convertMessagesToSdkMessages()` transforms filtered `Message[]` into AI SDK `ModelMessage[]`:
   - User messages → `UserModelMessage` with `TextPart | ImagePart | FilePart` content
   - Assistant messages → `AssistantModelMessage` with `TextPart | ReasoningPart | FilePart` content
   - Vision models: merges images from the prior assistant message into the last user message.
3. `parameterBuilder.buildStreamTextParams()` builds the full AI SDK `StreamTextParams`:
   - Sets `messages`, `maxOutputTokens`, `temperature`, `topP`
   - Determines capabilities: `enableReasoning`, `enableWebSearch`, `enableGenerateImage`, `enableUrlContext`
   - Builds `providerOptions` for provider-specific parameters
   - Configures tools, system prompt, abort signals, custom headers
   - Returns `{ params, modelId, capabilities, webSearchPluginConfig }`.

**Key types (from the `ai` package):**
- `ModelMessage` = `UserModelMessage | AssistantModelMessage | SystemModelMessage`
- `StreamTextParams` — Full parameter set for `streamText()` from Vercel AI SDK

---

## Layer 3: Plugin Pipeline

**Purpose:** Intercept and transform the request/response at multiple lifecycle hooks — before the AI call, during streaming, and after completion.

**Key files:**
- `packages/aiCore/src/core/runtime/pluginEngine.ts` — `PluginEngine.executeStreamWithPlugins()` (line ~289)
- `packages/aiCore/src/core/plugins/types.ts` — `AiPlugin` interface
- `packages/aiCore/src/core/plugins/manager.ts` — `PluginManager` (sorting & routing hooks)
- `src/renderer/src/aiCore/plugins/PluginBuilder.ts` — `buildPlugins()` (line ~46)

**Plugin lifecycle hooks (in order):**

| Hook | Mode | Purpose |
|------|------|---------|
| `configureContext` | Serial | Each plugin configures the shared request context |
| `onRequestStart` | Parallel | Fire-and-forget notifications (e.g., telemetry) |
| `resolveModel` | First-wins | Convert modelId string → `LanguageModel` instance |
| `transformParams` | Serial chain | Each plugin transforms the stream/generate params |
| `transformStream` | Collect | Each plugin returns an AI SDK `TransformStream` |
| (AI call executes) | — | The actual `streamText()` call with transformed params |
| `transformResult` | Serial chain | Transform the final result object |
| `onRequestEnd` | Parallel | Notify plugins of completion |
| `onError` | Parallel | Error handling |

**Built-in plugins (as constructed by `PluginBuilder`):**
- **Telemetry plugin** — traces requests in dev mode
- **PDF compatibility plugin** — converts PDF FileParts to TextParts for unsupported providers
- **Reasoning extraction plugin** — extracts `<think>` tags for OpenAI/Azure
- **Simulate streaming plugin** — wraps non-streaming responses in a stream
- **Anthropic cache plugin** — estimates token thresholds for Anthropic prompt caching
- **OpenRouter reasoning redaction plugin** — redacts reasoning for OpenRouter
- **DeepSeek DSML parser plugin** — converts leaked DSML tags to tool calls
- **No-think plugin** — suppresses thinking for OVMS MCP tools
- **Qwen thinking plugin** — adds thinking support for Qwen models
- **Skip Gemini thought signature plugin** — skips Gemini thought markers
- **Provider tool plugins** — web search, URL context tools
- **Search orchestration plugin** — MCP tool-based search orchestration
- **Prompt tool use plugin** — MCP tools via prompt injection

**The `AiPlugin` interface:**
```typescript
interface AiPlugin<TParams, TResult> {
  name: string
  enforce?: 'pre' | 'post'
  resolveModel?: (modelId: string, context) => Promise<LanguageModel | null>
  configureContext?: (context) => void | Promise<void>
  transformParams?: (params, context) => Partial<TParams> | Promise<Partial<TParams>>
  transformResult?: (result, context) => TResult | Promise<TResult>
  onRequestStart?: (context) => void | Promise<void>
  onRequestEnd?: (context, result) => void | Promise<void>
  onError?: (error, context) => void | Promise<void>
  transformStream?: (params, context) => TransformStream<TextStreamPart>
}
```

---

## Layer 4: Provider / Model Resolution

**Purpose:** Map a user-selected provider + model to the correct AI SDK provider configuration and settings.

**Key files:**
- `src/renderer/src/aiCore/provider/factory.ts` — `getAiSdkProviderId()` (line ~26)
- `src/renderer/src/aiCore/provider/providerConfig.ts` — `providerToAiSdkConfig()` (line ~108), `adaptProvider()` (line ~154)
- `src/renderer/src/services/ProviderService.ts` — CRUD for provider configurations
- `src/renderer/src/services/ModelService.ts` — model list management
- `src/renderer/src/store/llm.ts` — Redux store for LLM providers/models

**How it works:**
1. The user selects a model bound to a `Provider` object (stored in Redux/IndexedDB).
2. `factory.getAiSdkProviderId()` maps the provider to an AI SDK-compatible ID:
   - Special cases: `azure-responses`, `xai-responses`
   - Checks `appProviderIds` registry (from extensions)
   - Detects `api.openai.com` → `openai-chat`
   - Falls back to `provider.id`
3. `providerConfig.providerToAiSdkConfig()` builds the SDK configuration:
   - Extracts `baseURL` and endpoint path via `routeToEndpoint()`
   - Matches against config builders for special providers (copilot, cherryai, anthropic-oauth, ollama, azure, bedrock, vertex, cherryin, newapi, aihubmix)
   - Standard SDK providers → `buildGenericProviderConfig()`
   - Unsupported providers → `buildOpenAICompatibleConfig()`
   - Returns `ProviderConfig { providerId, providerSettings, endpoint }`
4. `adaptProvider()` formats API host URLs (adds `/v1`, handles Anthropic/Gemini/Azure/Vertex/Ollama-specific formats).

**Key type:**
```typescript
type ProviderConfig<T> = {
  providerId: T        // e.g., 'openai', 'anthropic', 'azure'
  endpoint?: string    // path from baseURL
  providerSettings: AppProviderSettingsMap[T]
}
```

---

## Layer 5: AI SDK Runtime (`@cherrystudio/ai-core`)

**Purpose:** Core orchestrator that binds plugins, provider resolution, and the Vercel AI SDK together.

**Key files:**
- `packages/aiCore/src/core/runtime/executor.ts` — `RuntimeExecutor` (line ~30)
- `packages/aiCore/src/core/runtime/pluginEngine.ts` — `PluginEngine`
- `packages/aiCore/src/core/runtime/index.ts` — `createExecutor()` factory (line ~23)

**How it works:**
1. The renderer calls `AiProvider.modernCompletions()` (in `src/renderer/src/aiCore/AiProvider.ts`), which:
   - Calls `buildPlugins()` to get the plugin array
   - Calls `createExecutor(providerId, providerSettings, plugins)` from `@cherrystudio/ai-core`
2. `createExecutor()` (runtime/index.ts):
   - Looks up a provider extension from `extensionRegistry`
   - Creates a provider instance via `extensionRegistry.createProvider(providerId, settings)`
   - Gets a `modelResolver` from extension variants
   - Returns `new RuntimeExecutor(providerId, provider, options, plugins, modelResolver)`
3. `RuntimeExecutor.streamText()` (line ~93):
   - Creates a `createProviderRegistry({ [providerId]: provider })` from the Vercel AI SDK
   - Delegates to `pluginEngine.executeStreamWithPlugins(params, executorFn)`
   - The executor function calls the AI SDK's internal `_streamText()` with the registry
4. `PluginEngine.executeStreamWithPlugins()` orchestrates the full lifecycle (see Layer 3).
5. For non-streaming, `executor.generateText()` follows a similar path.

**Key types:**
```typescript
interface RuntimeConfig<TSettingsMap, T> {
  providerId: T
  provider: ProviderV3
  providerSettings: TSettingsMap[T]
  plugins?: AiPlugin[]
  modelResolver?: (modelId: string) => any
}
```

**Entry point from renderer:**
`src/renderer/src/aiCore/AiProvider.ts` — `modernCompletions()` (line ~231), which also creates an `AiSdkToChunkAdapter` to convert AI SDK stream parts into Cherry Studio's internal `Chunk` types.

---

## Layer 6: Provider SDK (Provider Extension System)

**Purpose:** Provide a pluggable, type-safe system for registering and instantiating AI provider SDK implementations.

**Key files:**
- `packages/aiCore/src/core/providers/core/ProviderExtension.ts` — Base extension class (line ~109)
- `packages/aiCore/src/core/providers/core/ExtensionRegistry.ts` — Registry for all extensions
- `packages/aiCore/src/core/providers/core/initialization.ts` — Registers core extensions (line ~249)
- `src/renderer/src/aiCore/provider/extensions/index.ts` — Render-specific extensions (line ~219)
- `packages/ai-sdk-provider/src/cherryin-provider.ts` — Custom multi-backend provider

**How it works:**
1. **Core extensions** registered in `initialization.ts`:
   - `OpenAIExtension` — with `chat` variant using `provider.chat(modelId)`
   - `AnthropicExtension` — with `webSearch`, `urlContext` tool factories
   - `AzureExtension` — with `responses` and `anthropic` variants
   - `GoogleExtension` — Gemini, with `webSearch`, `urlContext` tool factories
   - `XaiExtension` — Grok, with `responses` variant
   - `DeepSeekExtension`, `OpenRouterExtension`
   - `OpenAICompatibleExtension` — fallback for any OpenAI-compatible API
   - `CherryInExtension` — with `chat` variant

2. **Render-specific extensions** (extensions/index.ts):
   - `GoogleVertexExtension`, `GoogleVertexAnthropicExtension`
   - `GitHubCopilotExtension`, `BedrockExtension`
   - `PerplexityExtension`, `MistralExtension`, `HuggingFaceExtension`
   - `GatewayExtension`, `CerebrasExtension`, `GroqExtension`
   - `OllamaExtension`, `AiHubMixExtension`, `NewApiExtension`
   - `TogetherAIExtension`, `VoyageExtension`

3. **`ProviderExtension`** base class:
   - `createProvider(settings, variantSuffix?)` — creates/caches provider instances
   - Supports `create` (direct) or `import` (dynamic import) strategies
   - Handles **variants** (e.g., `openai-chat`, `azure-responses`) via `transform` functions
   - LRU cache (10 instances) with deduplication via hash

4. Each extension declares `toolFactories` that map capabilities (`webSearch`, `urlContext`) to provider-specific tool implementations.

5. The `merged.ts` file combines core + renderer extensions into a unified `AppProviderSettingsMap` type, providing end-to-end type safety.

**CherryIN provider** (`cherryin-provider.ts`) — a custom multi-backend provider that routes requests based on model ID prefix:
- `anthropic/` → Anthropic SDK (converts `Bearer` → `x-api-key`)
- `google/` → Gemini SDK (converts `Bearer` → `x-goog-api-key`)
- Otherwise → OpenAI-compatible SDK

---

## Layer 7: Network → AI Models

**Purpose:** Make the actual HTTP request to the AI model API and process the streaming response.

**Key files:**
- `node_modules/@ai-sdk/openai` / `@ai-sdk/anthropic` / `@ai-sdk/google` / etc. — Provider SDK packages that implement the `LanguageModelV3` interface
- `node_modules/@ai-sdk/provider` — Base `ProviderV3` interface
- `node_modules/ai` — Vercel AI SDK with `streamText()`, `generateText()`, `createProviderRegistry()`
- `src/renderer/src/aiCore/chunk/AiSdkToChunkAdapter.ts` — Converts AI SDK stream parts to Cherry Studio chunks

**How it works:**
1. The AI SDK (`streamText`) takes the transformed params including the `LanguageModel` instance (resolved by the plugin system).
2. Each provider SDK package (e.g., `@ai-sdk/openai`) implements `LanguageModelV3.doStream()` which:
   - Builds the HTTP request (URL, headers, JSON body) according to the provider's API spec
   - Sends an HTTP POST request using `fetch()` (native or custom)
   - Returns a `ReadableStream` of response chunks
3. Provider SDK packages included:
   - `@ai-sdk/openai` → `OpenAIChatLanguageModel`, `OpenAIResponsesLanguageModel`
   - `@ai-sdk/anthropic` → `AnthropicMessagesLanguageModel`
   - `@ai-sdk/google` → `GoogleGenerativeAILanguageModel`
   - `@ai-sdk/azure` → Azure OpenAI
   - `@ai-sdk/xai` → xAI (Grok)
   - `@ai-sdk/deepseek` → DeepSeek
   - `@openrouter/ai-sdk-provider` → OpenRouter
   - `@ai-sdk/openai-compatible` → Generic OpenAI-compatible
   - Custom: `cherryin-provider.ts` — multi-backend routing provider
4. The AI SDK returns a `fullStream` (`ReadableStream<TextStreamPart>`) containing:
   - `text-delta` — incremental text tokens
   - `reasoning` — reasoning/thinking tokens
   - `tool-call` / `tool-result` — tool invocations
   - `error` / `abort` — error or cancellation events
   - `finish` — completion metadata (usage, finishReason)

**Stream processing back in the renderer (`AiSdkToChunkAdapter`, line ~26):**
1. Reads the `fullStream` from the AI SDK
2. Converts each `TextStreamPart` to a Cherry Studio `Chunk` type:
   - `text-delta` → `ChunkType.TEXT`
   - `reasoning` → `ChunkType.THINKING`
   - `tool-call` → `ChunkType.TOOL_CALL`
   - `error` / `abort` → `ChunkType.ERROR`
   - `finish` → `ChunkType.BLOCK_COMPLETE` + `ChunkType.LLM_RESPONSE_COMPLETE`
3. Handles web search citations, MCP tool calls, and session updates
4. Tracks timing metrics: `time_first_token_millsec`, `time_completion_millsec`
5. Calls the `onChunk` callback for each chunk

---

## End-to-End Data Flow Summary

```
1. Renderer (Inputbar.tsx)
   User types text, attaches files → dispatch(sendMessage)

2. Message Conversion (ConversationService + messageConverter + parameterBuilder)
   Filter context → Convert to AI SDK messages → Build StreamTextParams

3. Plugin Pipeline (PluginBuilder + PluginEngine)
   15+ plugins → configureContext → resolveModel → transformParams → stream transforms

4. Provider/Model Resolution (factory + providerConfig)
   Map provider config → AI SDK provider ID → ProviderConfig

5. AI SDK Runtime (RuntimeExecutor + PluginEngine)
   createExecutor() → streamText() → pluginEngine.executeStreamWithPlugins()
   → AI SDK _streamText() with transformed params

6. Provider SDK (@ai-sdk/openai/anthropic/google/...)
   LanguageModelV3.doStream() → build HTTP request → return ReadableStream

7. Network → AI Models
   HTTP POST → AI provider API → streaming response

8. Stream Processing (back in renderer)
   AiSdkToChunkAdapter → convert TextStreamPart → Cherry Studio Chunks
   → BlockManager + throttled IndexedDB writes → Redux store → React re-render
```

## Key Architectural Patterns

- **Plugin as Middleware:** The plugin system follows a middleware pattern with lifecycle hooks (`resolveModel`, `transformParams`, `streamTransforms`, `transformResult`) that intercept the request at every stage.
- **Extension Registry:** Providers are registered as extensions with type-safe configs, supporting variants and tool factories.
- **Two-Level SDK:** The Vercel AI SDK provides generic LLM orchestration (`streamText`, `generateText`), while `@ai-sdk/*` packages implement provider-specific HTTP communication.
- **Adapter Pattern:** `AiSdkToChunkAdapter` converts between the generic AI SDK stream format and Cherry Studio's internal chunk types.
- **Redux Thunk Orchestration:** The `messageThunk.ts` file ties the entire flow together, from saving user messages to processing assistant responses.

## Complete File Reference

| Layer | File Path |
|-------|-----------|
| Renderer | `src/renderer/src/pages/home/Chat.tsx` |
| Renderer | `src/renderer/src/pages/home/Inputbar/Inputbar.tsx` |
| Renderer | `src/renderer/src/pages/home/Messages/Messages.tsx` |
| Message Conversion | `src/renderer/src/services/ConversationService.ts` |
| Message Conversion | `src/renderer/src/aiCore/prepareParams/messageConverter.ts` |
| Message Conversion | `src/renderer/src/aiCore/prepareParams/parameterBuilder.ts` |
| Message Conversion | `src/renderer/src/utils/messageUtils/filters.ts` |
| Plugin Pipeline | `packages/aiCore/src/core/runtime/pluginEngine.ts` |
| Plugin Pipeline | `packages/aiCore/src/core/plugins/types.ts` |
| Plugin Pipeline | `packages/aiCore/src/core/plugins/manager.ts` |
| Plugin Pipeline | `src/renderer/src/aiCore/plugins/PluginBuilder.ts` |
| Provider Resolution | `src/renderer/src/aiCore/provider/factory.ts` |
| Provider Resolution | `src/renderer/src/aiCore/provider/providerConfig.ts` |
| AI SDK Runtime | `packages/aiCore/src/core/runtime/executor.ts` |
| AI SDK Runtime | `packages/aiCore/src/core/runtime/index.ts` |
| AI SDK Runtime | `src/renderer/src/aiCore/AiProvider.ts` |
| Provider Extension | `packages/aiCore/src/core/providers/core/ProviderExtension.ts` |
| Provider Extension | `packages/aiCore/src/core/providers/core/ExtensionRegistry.ts` |
| Provider Extension | `packages/aiCore/src/core/providers/core/initialization.ts` |
| Provider SDK | `packages/ai-sdk-provider/src/cherryin-provider.ts` |
| Stream Processing | `src/renderer/src/aiCore/chunk/AiSdkToChunkAdapter.ts` |
| Central Orchestrator | `src/renderer/src/store/thunk/messageThunk.ts` |
| API Service | `src/renderer/src/services/ApiService.ts` |
| Stream Processor | `src/renderer/src/services/messageStreaming/BlockManager.ts` |
