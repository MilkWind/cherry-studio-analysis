# Cherry Studio: Multimedia Communication Between AI Models

## 1. Overall Architecture

The project uses a **4-layer architecture** for multimedia communication:

```
User Input (Renderer) 
  └─> Message Conversion (messageConverter.ts)
  └─> Plugin Pipeline (PluginBuilder.ts → pdfCompatibility, reasoning, etc.)
  └─> Provider/Model Resolution (AiProvider.ts)
  └─> AI SDK Runtime (packages/aiCore/runtime/executor.ts)
  └─> Provider SDK (CherryIN, NewAPI, AiHubMix, or native Anthropic/OpenAI/Google)
  └─> Network → AI Models (OpenAI, Claude, Gemini, etc.)
```

Responses flow back through a **chunk-based streaming pipeline**:
```
AI Model → AI SDK → StreamProcessingService → ChunkEvents → BlockManager → UI
```

---

## 2. Key Files and Their Roles

### Message Structure & Types

| File | Role |
|---|---|
| `src/renderer/src/types/newMessage.ts` | Core message types: `ImageMessageBlock`, `FileMessageBlock`, `MainTextMessageBlock`, `ThinkingMessageBlock`, `VideoMessageBlock`, `AudioMessageBlock` |
| `src/renderer/src/types/chunk.ts` | Streaming chunk types: `IMAGE_CREATED`, `IMAGE_DELTA`, `IMAGE_COMPLETE`, `AUDIO_START/DELTA/COMPLETE`, `VIDEO_SEARCHED` |
| `src/renderer/src/types/file.ts` | `FileMetadata` (id, name, ext, type, size), `FILE_TYPE` enum (IMAGE, VIDEO, AUDIO, TEXT, DOCUMENT, OTHER), `RemoteFile` types for Gemini/OpenAI/Mistral |
| `src/renderer/src/types/index.ts` | `GenerateImageResponse` type: `{type: 'url'|'base64', images: string[]}`, `GenerateImageParams`, `EditImageParams` |

### Message Conversion (Renderer → AI SDK Format)

| File | Role |
|---|---|
| `src/renderer/src/aiCore/prepareParams/messageConverter.ts` | **Central conversion hub**: Converts Cherry Studio messages to AI SDK `ModelMessage[]` format. Handles images, files, text, thinking blocks |
| `src/renderer/src/aiCore/prepareParams/fileProcessor.ts` | File handling: `convertFileBlockToFilePart()` (PDFs → base64 FileParts, images → base64), `handleOpenAILargeFileUpload()` (fileid://), `handleGeminiFileUpload()` |
| `src/renderer/src/aiCore/prepareParams/modelCapabilities.ts` | Model capability checks: `supportsImageInput()`, `supportsLargeFileUpload()`, `getFileSizeLimit()` |
| `src/renderer/src/config/models/vision.ts` | Vision model detection (regex-based): `isVisionModel()`, `isDedicatedImageModel()`, `isImageEnhancementModel()`, `isGenerateImageModel()` |

### AI SDK Runtime (Core Execution)

| File | Role |
|---|---|
| `packages/aiCore/src/core/runtime/executor.ts` | `RuntimeExecutor` class: `streamText()`, `generateText()`, `generateImage()`, `embedMany()`. Plugin engine integration |
| `packages/aiCore/src/core/runtime/pluginEngine.ts` | Plugin execution pipeline with middleware support |
| `packages/aiCore/src/core/runtime/types.ts` | `generateImageParams`, `generateImageResult` types |

### Provider Implementations

| File | Role |
|---|---|
| `packages/ai-sdk-provider/src/cherryin-provider.ts` | **CherryIN provider**: Routes models by ID prefix (`anthropic/*`, `google/*`) or endpointType. Provides `imageModel()`, `speechModel()`, `transcriptionModel()` |
| `src/renderer/src/aiCore/provider/custom/newapi-provider.ts` | **NewAPI provider**: Multi-backend gateway (OpenAI, Anthropic, Gemini, image-generation). Routes by `endpointType` |
| `src/renderer/src/aiCore/provider/custom/aihubmix-provider.ts` | **AiHubMix provider**: Routes by model ID prefix (claude→Anthropic SDK, gemini→Google SDK, others→OpenAI) |

### Image Handling

| File | Role |
|---|---|
| `src/renderer/src/aiCore/AiProvider.ts` | `generateImage()`, `editImage()`, `modernGenerateImage()`, `convertImageResult()` (converts base64 → data URIs) |
| `src/renderer/src/services/ApiService.ts` | High-level image generation: collects images from messages, calls `aiProvider.generateImage()` or `editImage()`, sends IMAGE_COMPLETE chunks |
| `src/renderer/src/services/messageStreaming/callbacks/imageCallbacks.ts` | **Response handling**: `onImageCreated()`, `onImageDelta()`, `onImageGenerated()` — saves base64 images to disk via `saveBase64Image()`, creates `ImageMessageBlock` entries |
| `src/renderer/src/aiCore/utils/image.ts` | Gemini-specific: `buildGeminiGenerateImageParams()` → `{responseModalities: ['TEXT', 'IMAGE']}` |
| `src/renderer/src/aiCore/plugins/pdfCompatibilityPlugin.ts` | Converts PDF FileParts to TextParts for providers without native PDF support |

### IPC Communication (Main ↔ Renderer)

| File | Role |
|---|---|
| `src/main/ipc.ts` | All IPC handlers: `File_Base64Image`, `File_Base64File`, `File_SaveBase64Image`, `File_Upload`, `FileService_*` |
| `src/main/services/FileStorage.ts` | `base64Image()`: reads file from disk → `{mime, base64, data: 'data:...'}`; `base64File()`: reads file → `{data: base64, mime}`; `saveBase64Image()`: saves data URL to disk |
| `src/preload/index.ts` | Preload API bridge: `window.api.file.base64Image()`, `window.api.file.saveBase64Image()`, `window.api.file.base64File()`, etc. |

### Remote File Services (File API Upload)

| File | Role |
|---|---|
| `src/main/services/remotefile/FileServiceManager.ts` | Routes to provider-specific services: GeminiService, OpenaiService, MistralService |
| `src/main/services/remotefile/OpenAIService.ts` | OpenAI Files API: `uploadFile()`, `retrieveFile()`, `deleteFile()` |
| `src/main/services/remotefile/GeminiService.ts` | Gemini Files API: upload with `mimeType`, track file state (ACTIVE/PROCESSING/FAILED) |

### Renderer Utilities

| File | Role |
|---|---|
| `src/renderer/src/utils/messageUtils/find.ts` | `findImageBlocks()`, `findFileBlocks()`, `findMainTextBlocks()`, `findThinkingBlocks()` |
| `src/renderer/src/services/FileManager.ts` | Renderer-side file management: `addFile()`, `uploadFile()`, `readBinaryImage()`, `addBase64File()` |
| `src/renderer/src/aiCore/utils/mcp.ts` | MCP multimodal handling: `hasMultimodalContent()` (detects image/audio/blob types), `mcpResultToTextSummary()` (replaces base64 with `[Image: ...]` placeholders) |

### Plugin Pipeline

| File | Role |
|---|---|
| `src/renderer/src/aiCore/plugins/PluginBuilder.ts` | Orchestrates all plugins: PDF compatibility, reasoning extraction, streaming simulation, cache control, web search |

---

## 3. How Images and Files Are Encoded and Sent

### Image Files (User Uploaded)

1. **Storage**: User-selected images are saved to the local files directory (`FileStorage.storageDir`) via `window.api.file.upload()`
2. **Retrieval**: During message conversion (`messageConverter.ts`), images are loaded via `window.api.file.base64Image(fileId)` which reads the file from disk and returns `{base64: string, mime: string, data: 'data:image/png;base64,...'}`
3. **Encoding**: The base64 data (without the `data:` prefix) is used with the MIME type:
   ```typescript
   { type: 'image', image: base64, mediaType: 'image/png' }
   ```
4. **Remote URL images**: Passed as URL directly: `{ type: 'image', image: url }`
5. **Data URL images**: Parsed via `parseDataUrl()`, base64 data extracted, passed as `{ type: 'image', image: data, mediaType }`
6. **Vision model check**: Images are only included if `isVisionModel(model)` returns true

### File Parts (PDFs, Documents, Text)

1. **PDF files** (`fileProcessor.ts`):
   - Small PDFs (< size limit): Converted to base64 `FilePart` via `window.api.file.base64File()` → `{ type: 'file', data: base64, mediaType: 'application/pdf', filename }`
   - Large PDFs: Uploaded via File API (`handleLargeFileUpload()`) → `{ type: 'file', data: 'fileid://openai_file_id' }`
   - For OpenAI models: Uploaded via `OpenAIService.uploadFile()`, returns `fileid://` reference
   - For Gemini models: Uploaded via `GeminiService.uploadFile()` with proper mimeType

2. **PDF Compatibility Plugin** (`pdfCompatibilityPlugin.ts`): At runtime, for providers that don't support native PDF `FilePart`s, the plugin extracts text from base64 PDF data and converts it to `TextPart`

3. **Text/Document files** (`fileProcessor.ts`):
   - Text files: Read via `window.api.file.read()` → `{ type: 'text', text: 'filename\ncontent' }`
   - Documents (Word, Excel): Extracted text via `window.api.file.read(id, true)` (force text extraction)

4. **MIME type correction for Anthropic** (`fileProcessor.ts`): `image/jpg` → `image/jpeg` (required by Anthropic API)

### Markdown Base64 Image Stripping

In assistant messages (`stripMarkdownBase64Images()`, `messageConverter.ts`): Replaces `![alt](data:image/...;base64,...)` with `![alt](image)` to avoid sending huge payloads to the API and prevent HTTP 413 errors.

---

## 4. How Responses with Multimedia Are Received

### Image Generation Responses

1. **Streaming flow**:
   - `IMAGE_CREATED` chunk → creates placeholder image block
   - `IMAGE_DELTA` chunk → updates with image URL/base64 data
   - `IMAGE_COMPLETE` chunk → final image data

2. **Image saving** (`imageCallbacks.ts`):
   - Base64 images: Saved to disk via `window.api.file.saveBase64Image(dataUrl)` → returns `FileMetadata`
   - URL images: Stored as direct URL references
   - Saved file metadata stored in `ImageMessageBlock.file`

3. **Non-streaming image generation** (`ApiService.ts`):
   - Calls `aiProvider.generateImage()` or `editImage()`
   - Result converted from `{base64, mediaType}` to `data:image/png;base64,...` format
   - Sent as single `IMAGE_COMPLETE` chunk

### Audio Responses

- `AUDIO_START` → `AUDIO_DELTA` (Base64 audio chunks) → `AUDIO_COMPLETE`
- Supported by providers with `speechModel()` and `transcriptionModel()` (CherryIN)

### MCP Tool Multimodal Results

- MCP tools can return image/audio/resource blob content
- `hasMultimodalContent()` detects these types
- `mcpResultToTextSummary()` converts them to text placeholders like `[Image: image/png, delivered to user]` to avoid exceeding message size limits (e.g., Kimi's 4MB limit)
- Actual media is displayed to user via `IMAGE_COMPLETE` chunks

---

## 5. Data Flow: User Input → API Call → Response

```
┌─────────────────────────────────────────────────────────────────┐
│  USER INPUT (Renderer)                                         │
│  Message with blocks: MainTextBlock + ImageBlock + FileBlock   │
└───────────────────────┬─────────────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│  MESSAGE CONVERSION (messageConverter.ts)                       │
│  1. findImageBlocks(msg), findFileBlocks(msg)                   │
│  2. isVisionModel(model) → convertImageBlockToImagePart()       │
│  3. For files: convertFileBlockToFilePart() (base64/File API)   │
│  4. Strip markdown base64 images from assistant messages        │
│  5. Build ModelMessage[]: {role, content: [TextPart|ImagePart   │
│     |FilePart|ReasoningPart]}                                  │
└───────────────────────┬─────────────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│  PARAMETER BUILDING (parameterBuilder.ts)                       │
│  1. Build system prompt + messages                              │
│  2. Build providerOptions (reasoning, web search, image gen)    │
│     - OpenAI: reasoning_effort, verbosity                       │
│     - Anthropic: thinking budget, beta headers                  │
│     - Gemini: responseModalities: ['TEXT','IMAGE']             │
│  3. Configure tools (MCP, web search)                           │
│  4. Build StreamTextParams (temperature, topP, maxTokens, etc.) │
└───────────────────────┬─────────────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│  PLUGIN PIPELINE (PluginBuilder.ts → Executor)                  │
│  - PDF compatibility: converts FilePart → TextPart for non-     │
│    native providers                                             │
│  - Reasoning extraction, streaming simulation, cache control    │
│  - Provider tool injection (web search, URL context)            │
│  - Prompt tool use plugin                                       │
└───────────────────────┬─────────────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│  PROVIDER ROUTING                                               │
│  - getAiSdkProviderId(provider) → 'openai','anthropic','google',│
│    'xai','bedrock','openai-compatible',etc.                     │
│  - Custom providers (CherryIN/NewAPI/AiHubMix) route by:        │
│    * endpoint_type field                                        │
│    * Model ID prefix (anthropic/*, google/*, claude*, gemini*)  │
│    * Heuristic model name matching                              │
└───────────────────────┬─────────────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│  AI SDK RUNTIME (executor.ts)                                   │
│  - streamText() → streaming response                            │
│  - generateImage() → image generation                           │
│  - Uses @ai-sdk/openai, @ai-sdk/anthropic, @ai-sdk/google, etc.│
└───────────────────────┬─────────────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│  STREAM PROCESSING (StreamProcessingService.ts)                 │
│  Chunk types:                                                   │
│  - TEXT_START/DELTA/COMPLETE → text content                     │
│  - THINKING_START/DELTA/COMPLETE → reasoning                    │
│  - IMAGE_CREATED/DELTA/COMPLETE → generated images              │
│  - AUDIO_START/DELTA/COMPLETE → audio output                    │
│  - MCP_TOOL_PENDING/IN_PROGRESS/COMPLETE → tool calls           │
│  - VIDEO_SEARCHED → video results from knowledge base           │
└───────────────────────┬─────────────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│  BLOCK MANAGER → UI UPDATE                                      │
│  - Creates/updates MessageBlocks in Redux store                 │
│  - imageCallbacks: saves base64 images to disk                  │
│  - textCallbacks: handles thoughtSignature for Gemini           │
│  - UI renders blocks (Markdown, images, video, audio, etc.)     │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. Provider-Specific Handling

### OpenAI
- Standard `ImagePart` format: `{type: 'image', image: base64|url, mediaType}`
- File upload: Uses OpenAI Files API (`fileid://` references)
- Image generation: Supports `dall-e`, `gpt-image` models via dedicated API or tool use
- Models like `qwen-long` use `purpose: 'file-extract'`

### Anthropic (Claude)
- `image/jpg` MIME type converted to `image/jpeg`
- PDF files limited to 32MB
- Extended thinking: reasoning blocks passed as `{type: 'reasoning', text}`
- Cache control: `anthropicCacheControl.tokenThreshold` via plugin

### Google Gemini
- `supportedUrls()` config differs: `{}` for Gemini (no URL image support in some contexts)
- File API: Large files uploaded via Gemini Files API
- Image generation: `buildGeminiGenerateImageParams()` adds `responseModalities: ['TEXT', 'IMAGE']`
- `thoughtSignature` passed through `TextPart.providerOptions.google.thoughtSignature`
- Model ID naming: `gemini-2.0-flash-preview-image-generation`, `gemini-2.5-flash-image`

### xAI (Grok)
- Vision models: `grok-vision-beta`, `grok-4`
- Image generation: `grok-2-image` via dedicated OpenAI-compatible endpoint

### Ollama
- PDF compatibility: `pdfCompatibilityPlugin` converts PDFs to text
- Supports image input via compatible vision models

### AWS Bedrock
- Anthropic beta headers: `addAnthropicHeaders()`
- Vertex AI gateway: Model routing by ID

### Proxy Providers (CherryIN, NewAPI, AiHubMix)
- **Multi-backend routing**: Single provider ID routes to different AI SDK providers based on model ID prefix or `endpoint_type`
- `supportedUrls()` for image handling varies by downstream provider
- PDF compatibility applies for non-native providers
- Provider options are mapped to the actual AI SDK provider key

---

## 7. Key Architecture Decisions

1. **Base64 encoding for local files**: All local images/files are read from disk, base64-encoded, and sent inline in the API request body. Remote URL images are passed as URLs directly.

2. **File API for large files**: For files exceeding provider size limits (e.g., 32MB for Anthropic PDFs, 20MB for Gemini), the system uploads them via the remote File API and passes `fileid://` references instead.

3. **Plugin-based PDF handling**: `pdfCompatibilityPlugin` runs as a pre-middleware to convert PDF `FilePart`s to `TextPart`s for providers that don't natively support PDF input.

4. **Image persistence**: Generated base64 images are saved to local disk (`saveBase64Image()`) to avoid sending huge `data:` URIs in subsequent conversation turns.

5. **Markdown base64 stripping**: During assistant message conversion, `stripMarkdownBase64Images()` replaces inline base64 data URI images with placeholder `![alt](image)` to prevent HTTP 413 errors.

6. **MCP multimodality safety**: Tool results with binary content (images, audio, blobs) are converted to text summaries to stay within message size limits.

7. **Chunk-based streaming**: All response types (text, thinking, image, audio, video, tools, citations) flow through a unified chunk pipeline with start/delta/complete events.
