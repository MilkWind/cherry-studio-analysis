# Electron Webview Session Management In Cherry Studio

Cherry Studio embeds external web content for mini apps and related browser-like features. The Electron-specific logic is mainly in:

- `src/main/services/WebviewService.ts`
- `src/main/services/WindowService.ts`
- `src/renderer/src/components/MinApp/WebviewContainer.tsx`

---

## Beginner's Prerequisites: Concepts You Need Before Reading

### What Is a Webview and Why Does It Exist?

A `<webview>` is a special Electron HTML element that embeds an entire separate web page inside your app. Think of it as "a Chrome tab running inside your app window."

**The key difference from an `<iframe>` (this is important):**

| Aspect | `<iframe>` | `<webview>` |
|--------|-----------|-------------|
| **Process** | Runs in the same process as the host page | Runs in its own separate renderer process |
| **Crash safety** | If the embedded page crashes, your whole app freezes | If the embedded page crashes, only that webview dies — your app keeps running |
| **Cookies/storage** | Shares with the host page | Can have its own separate session (isolated cookies, storage, cache) |
| **Security** | Less isolated — embedded page can potentially affect the host | More isolated — separate process, separate session |
| **API access** | Standard browser APIs only | Electron-specific APIs (controlled by main process) |
| **Use case** | Embedding your own trusted content | Embedding third-party or untrusted external content |

**Simple mental model:** If `<iframe>` is like having a guest sleep on your couch (same house, shared kitchen), `<webview>` is like having a guest stay in a separate guest house on your property (separate building, own kitchen, but you still control the property). The guest house can burn down without affecting your main house.

### What Is a "Session" in Electron?

In Chromium (and therefore Electron), a **session** is a container that holds all browsing data for a set of web pages:

- **Cookies** — small pieces of data that websites store to remember you (login state, preferences)
- **Local Storage / Session Storage** — key-value data stores that websites use
- **Cache** — saved copies of images, scripts, and other resources to load faster next time
- **Permissions** — granted permissions like camera, microphone, notifications
- **Service Workers** — background scripts that websites register for offline support and push notifications

The **default session** is what your main app window uses. A **partition** is a named alternative session. When you create a `<webview>` with `partition="persist:webview"`, you are saying:

> "Give this webview its own separate session named 'webview', and save it to disk ('persist') so it survives app restarts."

**Blind spot — "'persist:' vs 'ephemeral' (in-memory) partitions":**
- `partition="persist:webview"` → saves session data to disk → cookies survive app restarts
- `partition="webview"` (without `persist:`) → keeps session data only in memory → data is lost when the app closes

`persist:` is the prefix, `webview` is the partition name. The actual files are stored in Electron's user data directory.

### What Is a "User Agent" and Why Does It Matter?

A **user agent (UA)** is a string of text that every browser sends with every web request. It tells the website "this is who I am." Example:

```
Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko)
  Chrome/120.0.0.0 Safari/537.36 CherryStudio/1.0.0 Electron/28.0.0
```

Websites read this string to decide:
- Whether to serve the mobile or desktop version
- Whether to show "Download our app!" banners
- Whether to block access ("This browser is not supported")
- Whether to apply specific compatibility workarounds

**Why Electron's default user agent can cause problems:** The `Electron/28.0.0` token in the user agent tells websites "this is an Electron app, not a regular Chrome browser." Some websites react negatively:
- "You are using an unsupported browser. Please use Chrome." (because they detect "Electron" and do not recognize it)
- Reduced functionality or degraded experience
- Complete blocking

**This is why Cherry Studio strips Electron from the webview user agent** — so embedded websites think they are loading in regular Chrome and behave normally.

### Key Electron APIs Referenced in This Document

| API | What It Does |
|-----|-------------|
| `session.fromPartition(name)` | Gets (or creates) a named session |
| `session.defaultSession` | The session used by the main app window |
| `ses.setUserAgent(userAgent)` | Sets the user agent string for all requests from this session |
| `ses.webRequest.onBeforeSendHeaders(callback)` | Intercepts every outgoing HTTP request, allowing modification of headers |
| `webContents.setWindowOpenHandler(handler)` | Controls what happens when a page tries to open a new window |
| `webContents.fromId(id)` | Looks up a webContents by its numeric ID |
| `webviewTag: true` | Enables `<webview>` support in a BrowserWindow's webPreferences |
| `webContents.printToPDF(options)` | Renders the current page to a PDF buffer |
| `webContents.executeJavaScript(code)` | Runs arbitrary JavaScript inside a webContents and returns the result |

---

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

### Plain-Language Explanation

**Why `webviewTag: true` is not the default:** Electron disables the `<webview>` tag by default because it is powerful and potentially dangerous. A `<webview>` runs external content in a separate process with its own session — but it also has access to Electron-specific APIs that regular iframes do not. Electron wants developers to explicitly opt in to this power rather than having it available by accident.

**Why Cherry Studio uses `<webview>` instead of `<iframe>` for mini apps:**

1. **Process isolation:** Each mini app is a separate third-party web application. If one crashes or freezes, it should not take down the entire Cherry Studio window. `<webview>` provides this isolation; `<iframe>` does not.

2. **Session control:** Cherry Studio can control the cookies, user agent, and permissions of embedded content without affecting the main app's own session. With `<iframe>`, embedded pages share cookies with the main app — a security and privacy concern.

3. **Content export:** `<webview>` supports `printToPDF()` and `executeJavaScript()` through the main process, enabling the PDF export and HTML save features. `<iframe>` does not support these.

**Why use a shared partition (`persist:webview`)?**

All mini apps use the same session partition. This means:
- **Shared cookies:** If you log into GitHub in one mini app, other mini apps that embed GitHub pages will also see you as logged in. This is usually desired — login state should persist across the app.
- **Shared storage:** All mini apps share the same LocalStorage and cache space.
- **Survival across restarts:** `persist:` means the session data (cookies, storage) is saved to disk, so login state survives closing and reopening the app.

**Tradeoff to be aware of:** If two completely different mini apps must NOT share cookies (e.g., a corporate app and a personal app embedded side by side), they should use different partitions (e.g., `persist:work` and `persist:personal`). Cherry Studio uses one partition because all mini apps are considered part of the same user's toolset.

### Universal Reuse — Your Own Electron Project

```typescript
// Enabling webview in a BrowserWindow (main process)
const mainWindow = new BrowserWindow({
  webPreferences: {
    webviewTag: true,    // Enable <webview> tag
    preload: 'preload.js'
  }
});

// In your renderer HTML (React/plain HTML):
// <webview src="https://example.com" partition="persist:mywebview"></webview>
```

**When to use `<webview>` vs `<iframe>` in your own project:**
- Use `<webview>` when: embedding third-party content, need process isolation, need session control, need export features
- Use `<iframe>` when: embedding your own trusted content, simplicity matters more than isolation, the content does not need separate cookies/login state

---

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

### Plain-Language Explanation

**Step 1: Cleaning the user agent**

The default Electron user agent looks like:
```
Mozilla/5.0 (...) Chrome/120.0.0.0 Electron/28.0.0 CherryStudio/1.0.0
```

The cleaned version looks like:
```
Mozilla/5.0 (...) Chrome/120.0.0.0
```

By removing `Electron/28.0.0` and `CherryStudio/1.0.0`, the embedded page's requests look like they come from a regular Chrome browser. This avoids "unsupported browser" errors and ensures the website serves the standard desktop version.

**Step 2: Per-domain user agent overrides**

Some websites actually need to see the original user agent. Google services, for example, might use the user agent to determine feature support. Cherry Studio's `onBeforeSendHeaders` hook can keep the original user agent for specific domains:

```
Request to embed.example.com → cleaned user agent (no Electron/CherryStudio)
Request to google.com → original user agent (with Electron/CherryStudio)
```

This is the "user-agent shaping" pattern — fine-grained control over what identity each embedded site sees.

**Step 3: `Accept-Language` header injection**

The `Accept-Language` HTTP header tells websites "I prefer content in this language." Cherry Studio reads the user's language preference from app config and injects it into all webview requests:

```
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8
```

This means "I prefer Chinese (China), but will accept generic Chinese, and English as a fallback." The `q=` values are quality weights (0-1) indicating preference strength.

Without this injection, embedded websites might default to English or use geolocation to guess the language, which may not match the user's actual preference.

**Blind spot — "What is `onBeforeSendHeaders`?":** This is a Chromium webRequest API exposed by Electron. It fires for EVERY HTTP request made by a session, RIGHT BEFORE the request is sent over the network. Your callback can inspect and modify the request headers. It is a powerful interception point — and also a performance-sensitive one, since it fires for every request (images, scripts, API calls, etc.). Keep the callback fast.

**Blind spot — "What is an HTTP header?":** HTTP requests and responses have two parts: the body (the actual content — HTML, JSON, image data) and headers (metadata about the request/response). Headers are key-value pairs like `User-Agent: ...`, `Accept-Language: ...`, `Content-Type: text/html`. Headers control how the request is handled (authentication, caching, content negotiation) without being part of the displayed content.

### Universal Reuse — Your Own Electron Project

```typescript
import { session } from 'electron';

function setupWebviewSession() {
  const ses = session.fromPartition('persist:mywebview');

  // 1. Set base user agent (strips Electron signature)
  const cleanUA = ses.getUserAgent()
    .replace(/MyApp\/[\d.]+/, '')
    .replace(/Electron\/[\d.]+/, '')
    .trim();
  ses.setUserAgent(cleanUA);

  // 2. Per-domain overrides
  ses.webRequest.onBeforeSendHeaders((details, callback) => {
    const url = new URL(details.url);

    // Keep original UA for Google
    if (url.hostname.includes('google.com')) {
      details.requestHeaders['User-Agent'] = originalUserAgent;
    }

    // Inject language preference
    details.requestHeaders['Accept-Language'] = getUserLanguage();

    callback({ requestHeaders: details.requestHeaders });
  });
}
```

---

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

### Plain-Language Explanation

**The problem this solves:** Embedded web pages contain links. When the user clicks a link inside a mini app webview, what should happen? There are three possible behaviors:

1. **Navigate the webview:** The link loads inside the embedded webview (the user stays in the app)
2. **Open in system browser:** The link opens the user's default browser (Chrome, Safari, etc.)
3. **Open an Electron popup window:** A new small Electron window opens for the link

The "right" answer depends on the link:
- An OAuth login page should open as a small popup window (so the user can log in and the popup can close automatically after redirect)
- A link to external documentation should open in the system browser (the user's full browser is better for browsing)
- A link to a malicious site should be blocked entirely

**How `setWindowOpenHandler()` works:**

When a web page tries to open a new window (via `window.open()` or `<a target="_blank">`), Chromium intercepts it and asks your handler: "Should I allow this?" Your handler inspects the URL and returns:

- `{ action: 'allow' }` → Open the popup in a new Electron window
- `{ action: 'deny' }` → Block the popup entirely
- Open externally (manual) → Call `shell.openExternal(url)` and return `{ action: 'deny' }`

**The OAuth allowlist:** Some OAuth flows require popup windows. For example, "Sign in with Google" might open a small popup where the user enters credentials. Cherry Studio maintains an allowlist of known OAuth provider URLs that are permitted to open popups. Everything else is either blocked or sent to the system browser.

**The two-layer approach:**
- Layer 1 (WindowService): General policy for the main window — most links → system browser, OAuth → allow popup
- Layer 2 (Individual webview): Per-webview policy — some mini apps might be "trusted" and allow popups, others might be restricted to external-only

**Unsafe schemes:** The handler blocks non-HTTP schemes like `file://`, `javascript:`, or `data:` to prevent security vulnerabilities. `javascript:` URLs can execute arbitrary code; `file://` URLs can access local files.

**Blind spot — "What is `shell.openExternal()`?":** This Electron API opens a URL in the user's default system browser (Chrome, Safari, Firefox, Edge — whatever the user set as default). It is the recommended way to open external links from an Electron app. It respects the user's browser choice and keeps the external content out of the Electron sandbox.

### Universal Reuse — Your Own Electron Project

```typescript
const OAUTH_DOMAINS = ['accounts.google.com', 'github.com/login'];

mainWindow.webContents.setWindowOpenHandler(({ url }) => {
  try {
    const parsed = new URL(url);

    // Block dangerous schemes
    if (!['http:', 'https:'].includes(parsed.protocol)) {
      return { action: 'deny' };
    }

    // Allow OAuth popups
    if (OAUTH_DOMAINS.some(d => parsed.hostname.includes(d))) {
      return { action: 'allow' };
    }

    // Everything else: open in system browser
    shell.openExternal(url);
    return { action: 'deny' };
  } catch {
    return { action: 'deny' };
  }
});
```

---

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

### Plain-Language Explanation

**The problem:** When a user presses Ctrl+F inside a `<webview>`, the embedded page (the "guest") receives the keyboard event first. If the embedded page is Google Docs, Google Docs shows its OWN find dialog. But Cherry Studio wants to show the HOST app's custom find/export UI instead.

**Why the host does not automatically receive the shortcut:** The `<webview>` element is like a separate browser tab. Keyboard focus is inside the guest page, and Chromium delivers key events to whichever element has focus. The host React app never sees the key press.

**The solution — `before-input-event`:** This Electron-specific event fires on the guest's `webContents` BEFORE the keyboard event is delivered to the guest page's JavaScript. It gives the host (main process) a chance to intercept the event before the guest processes it.

**The flow:**
```
1. User presses Ctrl+F while focused inside webview
2. Chromium: "About to deliver key event to guest page"
3. before-input-event fires on webContents
4. Main process handler: "Is this Ctrl+F? Yes → forward to host renderer"
5. Main process calls hostWindow.webContents.send('webview-shortcut', 'CmdOrCtrl+F')
6. Host renderer receives the shortcut → shows custom find bar
7. Guest page never sees Ctrl+F
```

**Why specific keys are forwarded:**

| Shortcut | Purpose |
|----------|---------|
| Cmd/Ctrl+F | Host custom find (instead of guest browser find) |
| Cmd/Ctrl+P | Host print/PDF export (instead of guest browser print) |
| Cmd/Ctrl+S | Host save/export (instead of guest browser save) |
| Escape | Close overlays, exit focused mode in host UI |
| Enter | Confirm actions in host UI |

These are standard desktop-app shortcuts that the host app wants to own. The guest page should not handle them independently.

**Blind spot — "What is `webContents` vs the `<webview>` DOM element?":**
- The `<webview>` DOM element lives in the **renderer process** — it is an HTML tag like `<div>` or `<img>`. The renderer creates it, styles it, and positions it.
- The `webContents` object lives in the **main process** — it is the low-level controller for the web page loaded inside the webview.
- The renderer cannot directly call `webContents` methods (that would be a security violation).
- The main process cannot directly manipulate the `<webview>` DOM element (the main process has no DOM).
- They communicate through IPC and the `getWebContentsId()` / `webContents.fromId()` bridge.

### Universal Reuse — Your Own Electron Project

```typescript
import { BrowserWindow, webContents } from 'electron';

function setupWebviewShortcutBridging(
  hostWindow: BrowserWindow,
  webviewContentsId: number
) {
  const wc = webContents.fromId(webviewContentsId);
  if (!wc) return;

  wc.on('before-input-event', (event, input) => {
    // Only intercept when a modifier key is held or it's Escape/Enter
    const isModifier = input.meta || input.control;
    const isSpecial = input.key === 'Escape' || input.key === 'Enter';

    if (!isModifier && !isSpecial) return;

    if (isModifier && ['f', 'p', 's'].includes(input.key.toLowerCase())) {
      // Forward to host renderer
      hostWindow.webContents.send('webview-shortcut', {
        key: input.key,
        control: input.control,
        meta: input.meta
      });
      event.preventDefault();  // Prevent guest from receiving it
    }
  });
}
```

---

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

### Plain-Language Explanation

**The fundamental problem:** The renderer creates and controls the `<webview>` DOM element (positioning it, setting its size, loading URLs into it). But the main process is the only place that can call privileged APIs on the webview's underlying `webContents` (like `printToPDF()`, `executeJavaScript()`, `setWindowOpenHandler()`).

**The solution — ID-based bridge:**

```
Renderer:  webviewEl.getWebContentsId() → returns 3 (a number)
  ↓ (IPC: "toggle spellcheck for webview ID 3")
Main:     webContents.fromId(3) → returns the webContents object
Main:     webContents.session.setSpellCheckerEnabled(true/false)
```

**Why the renderer cannot directly access `webContents`:** Electron security model. If the renderer could call `webContents.fromId()`, it could access ANY webContents in the app, including the main window's own webContents. This would let a compromised embedded page escalate from "guest in a webview" to "controlling the entire app." The ID-based bridge keeps control in the main process: the renderer can ASK for operations, but the main process decides whether to perform them.

**Each use case explained:**

- **Spellcheck toggle:** Electron/Chromium has a built-in spellchecker. The main process can enable/disable it per-session. The renderer sends a toggle request with the webview ID.

- **Link-opening behavior:** The renderer can configure per-webview whether links should stay in-app or go to the system browser. The main process applies `setWindowOpenHandler()` to the specific webContents.

- **Print to PDF:** The renderer cannot call `printToPDF()` — it is a main-process-only API. The renderer sends the webview ID, the main process resolves the webContents and calls `printToPDF()`.

- **Save as HTML:** The main process uses `executeJavaScript()` to extract the page's HTML content, which requires direct access to the webContents.

### Universal Reuse — Your Own Electron Project

```typescript
// Preload: expose the bridge
contextBridge.exposeInMainWorld('api', {
  webview: {
    getWebContentsId: () => {
      const webview = document.querySelector('webview');
      return webview?.getWebContentsId();
    },
    toggleSpellcheck: (enabled: boolean) => {
      const webview = document.querySelector('webview');
      const id = webview?.getWebContentsId();
      if (id) ipcRenderer.invoke('webview:toggleSpellcheck', id, enabled);
    }
  }
});

// Main process: handle the IPC
ipcMain.handle('webview:toggleSpellcheck', (event, webviewId, enabled) => {
  const wc = webContents.fromId(webviewId);
  if (wc) {
    wc.session.setSpellCheckerEnabled(enabled);
    return true;
  }
  return false;
});
```

---

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

### Plain-Language Explanation

**Why export from main process instead of renderer:**

The renderer (where the `<webview>` DOM element lives) cannot:
- Show a native OS save dialog (that requires main process access)
- Write files to disk (that requires Node.js `fs` module access)
- Call `printToPDF()` (that is a main-process-only Electron API)

So the renderer initiates the export (user clicks "Export as PDF"), but the main process executes it.

**Print to PDF flow:**
```
1. Renderer: "User wants to export webview ID 5 as PDF"
   ↓ IPC
2. Main: webContents.fromId(5) → get the webContents
3. Main: wc.executeJavaScript('document.title') → get page title "My Mini App"
4. Main: dialog.showSaveDialog({ defaultPath: 'My Mini App.pdf' }) → user picks save location
5. Main: wc.printToPDF({ marginsType: 1 }) → renders page to PDF buffer
6. Main: fs.writeFileSync(chosenPath, pdfBuffer) → writes to disk
7. Main: "Export complete" → send result to renderer via IPC
```

**Save as HTML flow:**
```
Same as above, but step 5 is:
  wc.executeJavaScript(`
    '<!DOCTYPE html>\\n' + document.documentElement.outerHTML
  `)
  → Returns the complete HTML source as a string
  → fs.writeFileSync(chosenPath, htmlString, 'utf-8')
```

**`executeJavaScript()` — injecting code into a webview:**

This API runs arbitrary JavaScript inside the webview's page context and returns the result. It is powerful but must be used carefully:
- The JavaScript runs with the full privileges of the guest page (can access its DOM, cookies, etc.)
- The main process receives the return value (must be JSON-serializable)
- It is like opening Chrome DevTools console on the guest page and typing a command

**Why use `executeJavaScript('document.title')` instead of having the renderer read the title?** The renderer cannot access the webview's DOM. The `<webview>` element has security boundaries — the host page's JavaScript cannot read the guest page's `document.title`. Only the main process (via `executeJavaScript`) can reach into the guest page's DOM.

**Blind spot — "What is a Buffer?":** In Node.js, a `Buffer` is a chunk of raw binary data. `printToPDF()` returns the PDF file's bytes as a `Buffer`. To save it as a file, you write the buffer to disk with `fs.writeFileSync(path, buffer)`. If you see `Buffer` in Electron code, think "raw file data before it is written to disk."

### Universal Reuse — Your Own Electron Project

```typescript
import { dialog, webContents } from 'electron';
import * as fs from 'fs';

async function exportWebviewAsPdf(webviewId: number) {
  const wc = webContents.fromId(webviewId);
  if (!wc) throw new Error('WebContents not found');

  // Get page title (for default filename)
  const title = await wc.executeJavaScript('document.title');
  const defaultName = (title || 'export') + '.pdf';

  // Show save dialog
  const { filePath } = await dialog.showSaveDialog({
    defaultPath: defaultName,
    filters: [{ name: 'PDF Files', extensions: ['pdf'] }]
  });

  if (!filePath) return;  // User cancelled

  // Generate PDF and save
  const pdfBuffer = await wc.printToPDF({
    marginsType: 1,  // No margins
    printBackground: true
  });

  fs.writeFileSync(filePath, pdfBuffer);
  return filePath;
}

async function exportWebviewAsHtml(webviewId: number) {
  const wc = webContents.fromId(webviewId);
  if (!wc) throw new Error('WebContents not found');

  const title = await wc.executeJavaScript('document.title');
  const defaultName = (title || 'export') + '.html';

  const { filePath } = await dialog.showSaveDialog({
    defaultPath: defaultName,
    filters: [{ name: 'HTML Files', extensions: ['html', 'htm'] }]
  });

  if (!filePath) return;

  const html = await wc.executeJavaScript(`
    '<!DOCTYPE html>\\n' + document.documentElement.outerHTML
  `);

  fs.writeFileSync(filePath, html, 'utf-8');
  return filePath;
}
```

---

## 7. Practical Takeaways

- Use a persistent session partition when embedded apps should share cookies/state
- Normalize the guest user agent at the session layer
- Control popup behavior with `setWindowOpenHandler()`
- Forward important guest hotkeys back to the host renderer
- Use `webContents.fromId()` when the renderer only knows the DOM webview element

---

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

---

## 9. Typical Application Scenarios

- Add a new mini app that needs login persistence across sessions.
- Let users export the currently embedded page as PDF or HTML.
- Open OAuth pages internally for an allowlisted flow but force all other links into the system browser.
- Add guest-level spellcheck or search behavior controlled by the host UI.

---

## 10. Relationship To The Other Electron Records

- This document relies on `electron_window_tray_menu_architecture.md` because guest-content rules are attached from the shell that owns the windows.
- It relies on `electron_main_process_lifecycle.md` because guest session initialization and global hooks must happen during the correct startup phase.
- It can intersect with `electron_protocol_oauth_and_deep_linking.md` when embedded content participates in login or provider callback flows.
- It affects `electron_build_packaging_and_update_pipeline.md` if embedded content depends on preload exposure, assets, or packaging exceptions.
