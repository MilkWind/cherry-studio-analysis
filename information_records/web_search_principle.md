# Cherry Studio - Web Search Principle & Search Result Processing

## Overview

The web search feature in Cherry Studio follows a layered, plugin-based architecture using the AI SDK (`@cherrystudio/ai-core`). It supports **10 external search providers** plus **model-native built-in search** for providers like OpenAI, Anthropic, Gemini, etc.

---

## Architecture Layers

```
[UI Layer]
  WebSearchButton → WebSearchQuickPanelManager (provider selection)
  MessageWebSearch (in-chat status display)
        │
[Plugin Layer: aiCore/plugins/searchOrchestrationPlugin.ts]
  ├── onRequestStart: LLM-based intent analysis (XML prompt)
  ├── transformParams: injects search tools into generation request
  └── onRequestEnd:   stores conversation memory
        │
[Tool Layer: aiCore/tools/WebSearchTool.ts]
  ├── execute: calls WebSearchService.processWebsearch()
  └── toModelOutput: formats results with REFERENCE_PROMPT
        │
[Service Layer: services/WebSearchService.ts]
  ├── Parallel search execution (Promise.allSettled)
  ├── RAG compression (temporary knowledge base)
  ├── Cutoff compression (char/token truncation)
  └── Status updates to runtime store
        │
[Provider Wrapper: providers/WebSearchProvider/index.ts]
  ├── SpanManagerService tracing
  └── Blacklist filtering
        │
[Provider Factory: WebSearchProviderFactory.ts]
  ├── API-based: Tavily, Zhipu, Exa, ExaMCP, Bocha, Querit, Searxng
  └── Local-based: Google, Bing, Baidu (BrowserWindow HTML scraping)
        │
[Content Fetch: utils/fetch.ts]
  ├── Mozilla Readability (article extraction)
  ├── TurndownService (HTML → Markdown)
  └── Twitter/X oEmbed support
```

---

## 1. Intent Analysis (`searchOrchestrationPlugin.ts`)

When the user sends a message with web search enabled:

1. **`onRequestStart`** checks if the assistant has `webSearchProviderId`, knowledge bases, or memory enabled
2. Calls `analyzeSearchIntent()` which sends a specially-crafted XML prompt (`SEARCH_SUMMARY_PROMPT`) to a language model
3. The LLM returns XML like:

```xml
<websearch>
  <question>latest TypeScript features 2025</question>
  <question>TypeScript roadmap</question>
</websearch>
```

Or if no search is needed:

```xml
<websearch>
  <question>not_needed</question>
</websearch>
```

4. The XML is parsed into `ExtractResults` via `fast-xml-parser` (`utils/extract.ts`)

```typescript
// utils/extract.ts
const parser = new XMLParser({
  isArray: (name) => name === 'question' || name === 'links'
})
return parser.parse(text) as ExtractResults
```

5. If intent analysis fails, falls back to using the raw user message as the search query

---

## 2. Tool Injection (`searchOrchestrationPlugin.transformParams`)

In `transformParams`, the plugin adds tools to `params.tools` based on the analysis result:

```typescript
// aiCore/plugins/searchOrchestrationPlugin.ts:325
if (analysisResult?.websearch && assistant.webSearchProviderId) {
  const needsSearch = analysisResult.websearch.question?.[0] !== 'not_needed'
  if (needsSearch) {
    params.tools['builtin_web_search'] = webSearchToolWithPreExtractedKeywords(
      assistant.webSearchProviderId,
      analysisResult.websearch,  // pre-extracted questions
      context.requestId
    )
  }
}
```

The tool uses pre-extracted keywords so the LLM doesn't need to re-analyze intent when calling the tool.

---

## 3. Tool Execution (`WebSearchTool.ts`)

When the LLM invokes the `builtin_web_search` tool:

```typescript
// aiCore/tools/WebSearchTool.ts:42
execute: async ({ additionalContext }) => {
  let finalQueries = [...extractedKeywords.question]
  
  // LLM can override queries with additionalContext
  if (additionalContext?.trim()) {
    finalQueries = [additionalContext.trim()]
  }
  
  if (finalQueries[0] === 'not_needed') return { results: [] }
  
  const extractResults: ExtractResults = {
    websearch: { question: finalQueries, links: extractedKeywords.links }
  }
  
  return await WebSearchService.processWebsearch(webSearchProvider, extractResults, requestId)
}
```

---

## 4. Search Execution (`WebSearchService.ts`)

### 4.1 Core Flow

```typescript
// services/WebSearchService.ts:412
public async processWebsearch(provider, extractResults, requestId):
  // 1. Reset status
  await setWebSearchStatus(requestId, { phase: 'default' })
  
  // 2. Special case: "summarize" + links → fetch page contents directly
  if (questions[0] === 'summarize' && links?.length > 0) {
    const contents = await fetchWebContents(links)
    return { query: 'summaries', results: contents }
  }
  
  // 3. Parallel search for all questions
  const searchPromises = questions.map(q => this.search(provider, q))
  const searchResults = await Promise.allSettled(searchPromises)
  
  // 4. Aggregate results
  searchResults.forEach(result => {
    if (result.status === 'fulfilled') finalResults.push(...result.value.results)
    if (result.status === 'rejected') throw result.reason  // fail-fast
  })
  
  // 5. Apply compression (RAG or Cutoff)
  // ...
  
  return { query: questions.join(' | '), results: finalResults }
```

### 4.2 Individual Search

```typescript
// services/WebSearchService.ts:155
public async search(provider, query)
  → new WebSearchEngineProvider(provider, spanId).search(query, websearch)
```

---

## 5. Provider Layer

### 5.1 Wrapper (`providers/WebSearchProvider/index.ts`)

```typescript
export default class WebSearchEngineProvider {
  // 1. Factory creates concrete provider
  // 2. Wraps call in span tracing
  // 3. Filters results through blacklist
  
  public async search(query, websearch):
    const result = await withSpanResult(callSearch, traceParams, { query, websearch })
    return await filterResultWithBlacklist(result, websearch)
}
```

### 5.2 Factory (`WebSearchProviderFactory.ts`)

Maps 10 provider IDs to concrete classes:

| Provider ID | Class | API Endpoint |
|-------------|-------|-------------|
| `zhipu` | `ZhipuProvider` | `https://open.bigmodel.cn/api/paas/v4/web_search` |
| `tavily` | `TavilyProvider` | `https://api.tavily.com` |
| `exa` | `ExaProvider` | `https://api.exa.ai` |
| `exa-mcp` | `ExaMcpProvider` | `https://mcp.exa.ai/mcp` (free, SSE protocol) |
| `bocha` | `BochaProvider` | `https://api.bochaai.com/v1/web-search` |
| `querit` | `QueritProvider` | `https://api.querit.ai/v1/search` |
| `searxng` | `SearxngProvider` | self-hosted, fetches engine config + page content |
| `local-google` | `LocalGoogleProvider` | `https://www.google.com/search?q=%s` (HTML scraping) |
| `local-bing` | `LocalBingProvider` | `https://cn.bing.com/search?q=%s` (HTML scraping) |
| `local-baidu` | `LocalBaiduProvider` | `https://www.baidu.com/s?wd=%s` (HTML scraping) |

### 5.3 API-Based Providers (e.g., Tavily)

```typescript
// providers/WebSearchProvider/TavilyProvider.ts
public async search(query, websearch):
  const result = await this.tvly.search({ query, max_results: websearch.maxResults })
  return {
    query: result.query,
    results: result.results.slice(0, websearch.maxResults).map(r => ({
      title: r.title || 'No title',
      content: r.content || '',
      url: r.url || ''
    }))
  }
```

### 5.4 Local (Browser Scraping) Providers

```typescript
// providers/WebSearchProvider/LocalSearchProvider.ts
public async search(query, websearch, httpOptions):
  // 1. Open hidden Electron BrowserWindow with search URL
  content = await window.api.searchService.openUrlInSearchWindow(uid, url)
  
  // 2. Parse HTML to extract URLs (CSS selectors, provider-specific)
  searchItems = this.parseValidUrls(content).slice(0, websearch.maxResults)
  
  // 3. Fetch full page content for each URL (parallel)
  results = await Promise.all(fetchPromises)  // uses fetchWebContent()
  
  return { query, results: results.filter(r => r.content != noContent) }
```

- **Google**: selector `#search .MjjYud`
- **Bing**: selector `#b_results h2`, decodes Base64 redirect URLs
- **Baidu**: selector `#content_left .result h3`

---

## 6. Content Fetching (`utils/fetch.ts`)

Each result URL's content is fetched using `fetchWebContent()`:

```typescript
export async function fetchWebContent(url, format, usingBrowser, httpOptions):
  // 1. Validate URL format
  // 2. Get HTML (either via direct fetch or BrowserWindow)
  // 3. Extract main content using Mozilla Readability
  // 4. Convert to Markdown using TurndownService
  // 5. Special handling for Twitter/X via oEmbed API
  
  const doc = new DOMParser().parseFromString(html, 'text/html')
  const parsed = new Readability(doc).parse()
  
  if (format === 'markdown') {
    content = turndownService.turndown(parsed.content)
  }
  
  return { title: parsed.title, content, url }
```

- **Mozilla Readability**: Extracts the main article content, removing navigation, ads, etc.
- **TurndownService**: Converts the extracted HTML to clean Markdown
- **30-second timeout** with AbortSignal support
- **Twitter/X**: Special handling via `fetchXOEmbed()` for tweet content

---

## 7. Result Compression

After fetching all results, the service can compress them in two ways:

### 7.1 RAG Compression (`compressWithSearchBase`)

Uses embedding models to find the most relevant content:

```
1. Create a temporary LanceDB knowledge base
2. Add all raw search results as KnowledgeItems
3. Embed and index each item
4. For each question, query the knowledge base
5. Sort retrieved fragments by relevance score
6. Deduplicate (same content → keep highest score)
7. selectReferences() — Round Robin strategy across source URLs
8. consolidateReferencesByUrl() — merge fragments by source URL
9. Clean up the temporary knowledge base
```

**Round Robin selection** ensures each source URL gets fair representation:

```typescript
// utils/websearch.ts:selectReferences
// Groups references by source URL, then cycles through URLs
// selecting one reference from each group per round
while (selected.length < maxRefs && availableUrls.length > 0) {
  const currentUrl = availableUrls[roundIndex]
  selected.push(group.shift())
  roundIndex = (roundIndex + 1) % availableUrls.length
}
```

**Consolidation** merges multiple fragments from the same URL:

```typescript
// utils/websearch.ts:consolidateReferencesByUrl
// Groups fragments by source URL, then joins them with '\n\n---\n\n'
return Array.from(sourceGroups.values(), group => ({
  title: group.originalResult.title,
  url: group.originalResult.url,
  content: group.contents.join('\n\n---\n\n')
}))
```

### 7.2 Cutoff Compression (`compressWithCutoff`)

Simple truncation by character or token:

```typescript
const perResultLimit = Math.max(1, Math.floor(cutoffLimit / results.length))
// truncate each result's content to perResultLimit chars or tokens
```

---

## 8. Status Tracking

The search process updates the UI state through the runtime Redux store at each phase:

```
default → fetch_complete → rag → rag_complete / rag_failed → cutoff → default
```

Each status message includes:
- `phase`: current processing stage
- `countBefore` / `countAfter`: result counts for UI display
- `requestId`: ties the status to a specific chat request

---

## 9. Result Formatting for the LLM

After search completes, `WebSearchTool.toModelOutput` formats results using `REFERENCE_PROMPT`:

```
This tool searches for relevant information and formats results for easy citation.
Found 5 relevant sources. Use [number] format to cite specific information.

Please answer the question based on the reference materials

## Citation Rules:
- Cite at the end of sentences when appropriate using [number] format
- Multiple sources: [1][2]
- If reference content is not relevant, answer based on your knowledge

## My question is:
{user's question}

## Reference Materials:
```json
[
  { "number": 1, "title": "...", "content": "...", "url": "https://..." },
  { "number": 2, "title": "...", "content": "...", "url": "https://..." }
]
```
```

The LLM receives formatted references with citation instructions and generates answers with inline `[1]`, `[2]` citations.

---

## 10. Key Data Structures

```typescript
// Core result type
type WebSearchProviderResult = {
  title: string     // Page title
  content: string   // Extracted text in Markdown
  url: string       // Source URL
}

// Search response
type WebSearchProviderResponse = {
  query?: string              // Combined query string ("q1 | q2 | q3")
  results: WebSearchProviderResult[]
}

// Intent analysis output
type ExtractResults = {
  websearch?: { question: string[]; links?: string[] }
  knowledge?: { rewrite: string; question: string[] }
}

// Compression config
type CompressionConfig = {
  method: 'rag' | 'cutoff'
  embeddingModel?: Model
  rerankModel?: Model
  cutoffLimit?: number        // total chars/tokens budget
  cutoffUnit?: 'char' | 'token'
  documentCount?: number       // RAG document count per result
}
```

---

## 11. Feature Matrix

| Feature | API Providers (Tavily, etc.) | Local Providers (Google, Bing, Baidu) |
|---------|------------------------------|---------------------------------------|
| Authentication | API key (with key rotation) | None required |
| Network access | Direct HTTP to provider API | Electron BrowserWindow |
| Content returned | Provider's snippet + link | Full page HTML → Readability → Markdown |
| Parallel queries | Yes (Promise.allSettled) | Yes (Promise.allSettled) |
| Blacklist filtering | After search | After search |
| Content fetching | Not needed (API returns content) | Fetch each result URL individually |
| Setup | Configure API key | No setup needed |

---

## 12. Key Design Decisions

1. **Pre-extracted keywords**: Intent analysis runs once in the plugin's `onRequestStart` phase, and the extracted queries are bundled into the tool definition. The LLM receives a pre-configured tool, avoiding a second round of intent analysis.

2. **Parallel execution**: All search queries run concurrently via `Promise.allSettled()` for performance.

3. **Fail-fast on any failure**: If any search query in a batch fails, the error propagates (not silently ignored for individual queries).

4. **API key rotation**: Multiple API keys can be configured with comma separation, and `BaseWebSearchProvider.getApiKey()` rotates through them using persisted state.

5. **RAG compression lifecycle**: Temporary knowledge bases are always cleaned up in a `finally` block, regardless of success or failure.

6. **Two-tier blacklisting**: Results are filtered against user-configured `excludeDomains` and regex patterns, both at the provider wrapper level and via provider-native `domain_list` configuration for supported providers.

7. **Built-in + external coexistence**: Models with native web search (OpenAI, Anthropic, etc.) are represented alongside external providers in a unified UI, with `aiCore/utils/websearch.ts` building provider-specific parameter configs.
