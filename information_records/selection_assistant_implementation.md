# Selection Assistant - Implementation Principle

## Architecture Overview

The Selection Assistant is a **cross-platform text selection assistant** that detects when a user selects text in any desktop application and shows a floating toolbar with AI-powered actions (translate, explain, summarize, search, copy, quote, refine).

### Three-Layer Architecture

```
MAIN PROCESS (Electron Node.js)
  └── SelectionService (singleton, 1670 lines)
       ├── selection-hook (native node addon)
       ├── Toolbar BrowserWindow (floating, transparent)
       ├── Action BrowserWindows (preloaded pool)
       └── IPC Handlers (17 channels)

PRELOAD
  └── window.api.selection.* (IPC bridge)

RENDERER (React)
  ├── SelectionToolbar.tsx (toolbar UI)
  ├── SelectionActionApp.tsx (action window UI)
  ├── useSelectionAssistant hook (Redux ↔ Main)
  ├── selectionStore (Redux state management)
  └── Settings page (full configuration UI)
```

## Core Data Flow
+
### 1. Initialization (SelectionService.start())

- Called via `initSelectionService()` on app startup
- Loads the native `selection-hook` Node addon that hooks into OS-level text selection events
- On macOS, checks `isTrustedAccessibilityClient` for accessibility permission
- Creates the toolbar `BrowserWindow` (transparent, always-on-top, frameless)
- Preloads 1 action `BrowserWindow` into a pool for fast response
- Subscribes to config changes and `SelectionAssistantEnabled` toggle

### 2. Text Selection Detection

- The `selection-hook` native library monitors global OS text selection events
- When text is selected anywhere, emits a `text-selection` event containing:
  - `text` - the selected text
  - `programName` - the source application
  - `mousePosStart`/`mousePosEnd` - mouse coordinates
  - `startTop`/`startBottom`/`endTop`/`endBottom` - bounding rectangles
  - `posLevel` - positioning level (NONE, MOUSE_SINGLE, MOUSE_DUAL, SEL_FULL, SEL_DETAILED)

### 3. Filtering (shouldProcessTextSelection)

Evaluates selection events against three filter modes:
- **default**: Uses predefined OS-specific blacklist (excludes Explorer, Snipaste, Photoshop, etc.)
- **whitelist**: Only processes selections in user-listed apps
- **blacklist**: Processes all except user-listed apps

When in "selected" trigger mode, combines user blacklist with predefined blacklist.

### 4. Toolbar Positioning (processTextSelection → showToolbarAtPosition)

Determines optimal toolbar position based on `posLevel` and selection direction:
- **NONE** (cursor click): positions below cursor
- **MOUSE_SINGLE**: positions below mouse end position
- **MOUSE_DUAL**: checks single-line vs multi-line, positions top/bottom/left/right
- **SEL_FULL**/**SEL_DETAILED**: keyboard-based selection positioning

Converts physical to logical coordinates on Windows/Linux and ensures toolbar stays within screen boundaries.

### 5. Toolbar Interaction

Toolbar shows enabled action buttons; each action dispatches differently:
- **copy**: Writes to clipboard via `selectionHook.writeToClipboard()`
- **search**: Opens URL with search engine template (`{{queryString}}`)
- **quote**: Sends text to main window's input bar
- **translate/explain/summary/refine/custom**: Opens action window

Toolbar auto-hides on: outside click, mouse wheel, key press, or blur.

### 6. Action Window Execution

- A preloaded `BrowserWindow` is popped from the pool
- Action data is sent via IPC channel
- Window positioned relative to toolbar (or screen center)
- Renders `ActionTranslate` or `ActionGeneral` based on action ID
- Uses AI assistants/models for streaming responses
- Supports pinning, opacity control, minimize, and auto-close on blur

## Trigger Modes

| Mode | Behavior |
|------|----------|
| **Selected** (default) | Toolbar appears automatically when text is selected in any app |
| **Ctrl-key** (Windows only) | Press Ctrl key for 350ms while text is selected to trigger |
| **Shortcut** | User presses configurable keyboard shortcut to trigger manually |

## Default Action Items (selectionStore.ts)

1. **Translate** - AI translation with language detection
2. **Explain** - AI explanation of selected text
3. **Summary** - AI summarization
4. **Search** - Web search (Google default) or direct URL opening
5. **Copy** - Clipboard copy with success/fail animation
6. **Refine** - AI text refinement (disabled by default)
7. **Quote** - Quote text to chat input (disabled by default)

Users can add custom actions with custom prompts via the settings UI.

## Configuration (ConfigManager.ts)

| Config Key | Default | Description |
|------------|---------|-------------|
| `selectionAssistantEnabled` | `false` | Master enable/disable switch |
| `selectionAssistantTriggerMode` | `'selected'` | Toolbar trigger mode |
| `selectionAssistantFollowToolbar` | `true` | Action window follows toolbar |
| `selectionAssistantRemeberWinSize` | `false` | Remember action window size |
| `selectionAssistantFilterMode` | `'default'` | Application filter mode |
| `selectionAssistantFilterList` | `[]` | User-defined filter list |

## Key Files

| File | Purpose |
|------|---------|
| `src/main/services/SelectionService.ts` | Singleton service - core logic, toolbar/action window management, IPC handlers |
| `src/main/configs/SelectionConfig.ts` | Predefined blacklists and fine-tuned lists per OS |
| `src/main/services/ConfigManager.ts` | Config key definitions and getter/setter methods |
| `src/renderer/src/hooks/useSelectionAssistant.ts` | React hook bridging Redux store and main-process APIs |
| `src/renderer/src/store/selectionStore.ts` | Redux state management with 7 default action items |
| `src/renderer/src/types/selectionTypes.d.ts` | TypeScript type definitions |
| `src/renderer/src/windows/selection/toolbar/SelectionToolbar.tsx` | Floating toolbar UI component |
| `src/renderer/src/windows/selection/action/SelectionActionApp.tsx` | Action window component |
| `src/renderer/src/windows/selection/action/components/ActionTranslate.tsx` | Translation action handler |
| `src/renderer/src/windows/selection/action/components/ActionGeneral.tsx` | General action handler (summary, explain, refine) |
| `src/renderer/src/pages/settings/SelectionAssistantSettings/SelectionAssistantSettings.tsx` | Settings page UI |
| `src/preload/index.ts` | Exposes `window.api.selection.*` IPC bridge to renderer |
| `packages/shared/IpcChannel.ts` | IPC channel enum definitions (17 channels) |

## Platform-Specific Considerations

- **macOS**: Requires Accessibility permission (`isTrustedAccessibilityClient`). Special fullscreen app handling, dock icon management, `showInactive` to prevent focus stealing.
- **Windows**: Supports selection-based and Ctrl-key trigger modes. Uses `focusable: false` to prevent toolbar focus stealing.
- **Linux**: Detects Wayland vs X11. Wayland requires XWayland mode and `input` group access. Different coordinate handling due to Wayland/XWayland mismatch. Falls back to `blur` events when mouse-down hit-testing is unreliable.
- **All platforms**: Uses `screenToDipPoint` for coordinate space conversion on Windows/Linux.
