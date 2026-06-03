# Electron Webview Session Management In Cherry Studio

Cherry Studio embeds external web content for mini apps and related browser-like features. The Electron-specific logic is mainly in:

- `src/main/services/WebviewService.ts`
- `src/main/services/WindowService.ts`
- `src/renderer/src/components/MinApp/WebviewContainer.tsx`

## 1. Why This Project Uses `webview`

The main window enables:

```ts
webPreferences: {
  webviewTag: true
}
```

That allows renderer pages to mount Electron `<webview>` guests for mini apps. This is a deliberate desktop-shell choice and is different from using an iframe.

The project uses a shared partition:

```text
persist:webview
```

That means guest pages share a persistent Electron session across reloads and app restarts.

## 2. Custom User Agent For Embedded Apps

`initSessionUserAgent()` modifies the webview session user agent:
- removes the `CherryStudio/...` token
- removes the `Electron/...` token

This helps embedded sites behave more like they are being opened in a regular browser.

Cherry Studio then adds an `onBeforeSendHeaders` hook to:
- apply the cleaned user agent by default
- keep the original user agent for some domains like Google
- send `Accept-Language` based on app config

This is a strong example of Electron session-level request shaping.

## 3. Link-Opening Policy

Two layers manage guest navigation:

### Main window guest popups

`WindowService.setWindowOpenHandler()`:
- allows a small allowlist of OAuth/provider URLs to open inside Electron
- forces most other links to `shell.openExternal(...)`
- blocks unsafe schemes

### Individual webviews

`setOpenLinkExternal(webviewId, isExternal)` applies a `setWindowOpenHandler()` to a specific guest `webContents`.

If `isExternal` is true:
- safe URLs open in the system browser
- the guest popup is denied

If `isExternal` is false:
- guest popup creation is allowed

## 4. Guest Keyboard Shortcut Bridging

Webviews capture shortcuts inside the guest page, not the host React app. Cherry Studio solves that in main process.

`initWebviewHotkeys()` attaches `before-input-event` listeners to guest `webContents` and forwards selected keys to the host renderer:
- `Cmd/Ctrl+F`
- `Cmd/Ctrl+P`
- `Cmd/Ctrl+S`
- `Escape`
- `Enter`

The host renderer then uses `window.api.webview.onFindShortcut(...)` to open a custom search UI or trigger export actions.

This is a useful Electron knowledge point: guest pages need explicit event bridging when host UI should own keyboard UX.

## 5. Spellcheck And Session Access By WebContents ID

The renderer gets a webview's guest process id with:

```ts
webview.getWebContentsId()
```

Main process then resolves it with:

```ts
webContents.fromId(webviewId)
```

Cherry Studio uses that to:
- toggle spellcheck on the guest session
- set link-opening behavior
- print the guest page to PDF
- save the guest page as HTML

This pattern is common when the renderer owns the DOM `<webview>` element but only the main process can call privileged Electron APIs on the guest `webContents`.

## 6. Exporting Guest Content

Cherry Studio exposes two Electron-specific export helpers:

### Print to PDF

The main process:
- asks the guest page for its title
- shows a native save dialog
- calls `webview.printToPDF(...)`
- writes the returned buffer to disk

### Save as HTML

The main process:
- asks the guest page for its title
- shows a native save dialog
- runs `executeJavaScript(...)` inside the guest to capture `doctype + outerHTML`
- writes the HTML file to disk

This is a good example of mixing guest-page JS execution with native file dialogs from the Electron main process.

## 7. Practical Takeaways

- Use a persistent session partition when embedded apps should share cookies/state
- Normalize the guest user agent at the session layer
- Control popup behavior with `setWindowOpenHandler()`
- Forward important guest hotkeys back to the host renderer
- Use `webContents.fromId()` when the renderer only knows the DOM webview element

## 8. How To Apply This Knowledge In Development

Use this document when you need browser-like embedded content without giving renderer code full Electron privilege.

Choose the right design:
- Use a `<webview>` only when the feature needs a separate guest process, session control, or privileged main-process integration.
- Keep session, popup, export, and privileged guest APIs in main process.
- Use the persistent partition intentionally; if two mini apps must not share cookies, they need different partitions or a different embedding strategy.
- Decide early whether guest links should stay inside the app or move to the system browser.

Practical usage pattern:
1. Define the trust boundary for the guest content.
2. Choose the session partition and user-agent behavior.
3. Define popup, navigation, and shortcut policy before exposing the webview to users.
4. Add main-process helpers for any privileged guest operation such as printing, export, or spellcheck.

Common mistakes this avoids:
- Using `<webview>` where an ordinary iframe or external browser would be safer.
- Accidentally sharing cookies and session state across unrelated guest experiences.
- Letting guest pages spawn unrestricted windows.
- Expecting host shortcuts to work inside guest content without explicit forwarding.

## 9. Typical Application Scenarios

- Add a new mini app that needs login persistence across sessions.
- Let users export the currently embedded page as PDF or HTML.
- Open OAuth pages internally for an allowlisted flow but force all other links into the system browser.
- Add guest-level spellcheck or search behavior controlled by the host UI.

## 10. Relationship To The Other Electron Records

- This document relies on `electron_window_tray_menu_architecture.md` because guest-content rules are attached from the shell that owns the windows.
- It relies on `electron_main_process_lifecycle.md` because guest session initialization and global hooks must happen during the correct startup phase.
- It can intersect with `electron_protocol_oauth_and_deep_linking.md` when embedded content participates in login or provider callback flows.
- It affects `electron_build_packaging_and_update_pipeline.md` if embedded content depends on preload exposure, assets, or packaging exceptions.
