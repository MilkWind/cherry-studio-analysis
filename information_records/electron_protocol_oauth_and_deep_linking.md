# Electron Custom Protocol, OAuth Callback, And Deep-Link Flow

Cherry Studio uses the custom protocol `cherrystudio://` to receive OAuth callbacks and other in-app actions from the operating system.

Primary file:
- `src/main/services/ProtocolClient.ts`

Related entry points:
- `src/main/index.ts`
- `src/preload/index.ts`

---

## Beginner's Prerequisites: Concepts You Need Before Reading

### What Is a "Protocol" (URI Scheme)?

When you type `https://www.example.com` into a browser, the `https://` part is a **protocol** (also called a "URI scheme"). It tells your operating system "use the program that handles HTTPS to open this." Your OS has a registry of protocols and their associated programs:

| Protocol | Opens With |
|----------|-----------|
| `https://` | Your default web browser |
| `mailto:` | Your default email client |
| `spotify:` | Spotify desktop app |
| `slack:` | Slack desktop app |
| `cherrystudio:` | Cherry Studio |

A **custom protocol** is simply registering your own prefix (like `cherrystudio://`) and telling the OS "when someone opens a link with this prefix, launch my app."

**Simple mental model:** Protocols are like phone area codes. `https://` connects to "the browser company." `mailto:` connects to "the email company." Registering `cherrystudio://` is like getting your own area code — now the OS knows how to route those calls to you.

### What Is "Deep Linking"?

A "deep link" is a URL that does not just open an app — it opens the app to a **specific screen or action** inside the app. Compare:

- `cherrystudio://` → Just launches Cherry Studio (opens the main window)
- `cherrystudio://settings/updates` → Launches Cherry Studio AND navigates to the update settings page
- `cherrystudio://mcp/install?url=...` → Launches Cherry Studio AND starts an MCP package installation

The "deep" part means the link reaches deep into the app's navigation structure, not just the front door.

### What Is OAuth and Why Does It Need a Custom Protocol?

OAuth is the standard way to implement "Sign in with Google/GitHub/Apple/etc." in desktop apps. The flow works like this:

```
1. Cherry Studio opens your browser → https://provider.com/login?redirect=cherrystudio://callback
2. You log in to the provider (Google/GitHub/etc.) in your browser
3. Provider says "login successful, now redirecting to cherrystudio://callback?code=abc123"
4. Your browser tells the OS "open cherrystudio://callback?code=abc123"
5. OS launches Cherry Studio with that URL
6. Cherry Studio reads the `code=abc123` from the URL
7. Cherry Studio exchanges the code for an access token (server-to-server)
8. Login complete!
```

**Why can't the browser just send the result back directly?** The login happened in the browser, but Cherry Studio is a desktop app — they are separate programs. The browser cannot "reach into" the desktop app to deliver the result. The custom protocol is the bridge: the browser tells the OS, the OS tells the app.

**Why not use a local web server instead?** Some Electron apps run a tiny HTTP server on `localhost` to receive OAuth callbacks (the browser redirects to `http://localhost:54321/callback`). This approach has downsides:
- Firewalls or security software might block it
- Another app might be using the same port
- It breaks if the app is not running at the exact moment of redirect
- Some OAuth providers do not allow `localhost` redirect URLs

Custom protocols avoid all these issues. The OS handles the routing, and it works even if the app was not running (the OS launches it).

### Key Concepts Glossary

**"Callback URL" / "Redirect URI":** The URL that the OAuth provider sends the user back to after they log in. This is the `cherrystudio://oauth/callback` part.

**"Access token":** A secret string that proves "this user authorized this app." The app includes it in API requests to the provider. Think of it as a temporary keycard — it grants access without needing the user's password.

**"Authorization code":** A short-lived, one-time-use code that the app exchanges for an access token. It is an intermediate step: code → exchange → token. This is more secure than getting the token directly because the token exchange happens server-to-server, not in the browser.

**"MCP" (Model Context Protocol):** A protocol for connecting AI assistants to external tools and data sources. `cherrystudio://mcp/...` URLs are for installing MCP packages.

**Blind spot — "What's the difference between a URI, URL, and URN?":**
- **URI** (Uniform Resource Identifier) — the general category. Anything that identifies a resource.
- **URL** (Uniform Resource Locator) — a URI that also tells you WHERE to find it. `https://example.com/page` is a URL (it tells you the location).
- **URN** (Uniform Resource Name) — a URI that names something without saying where it is. `urn:isbn:0451450523` identifies a book by ISBN but does not tell you where to get it.

In practice, "custom protocol URL" and "custom URI scheme" are used interchangeably. `cherrystudio://` is technically a URI scheme that forms URLs.

---

## 1. Registering The Protocol Client

The protocol name is:

```text
cherrystudio
```

Registration happens with `app.setAsDefaultProtocolClient(...)`.

Cherry Studio handles two cases:
- dev mode with `process.defaultApp`, passing the app entry script path
- normal packaged mode, registering the app directly

This distinction matters because Electron dev launches usually run through the Node/Electron executable plus your project entry, while packaged apps have a single executable.

### Plain-Language Explanation

**`app.setAsDefaultProtocolClient(protocol, path?)`:** This Electron API tells your operating system "when someone clicks a `protocol://` link, launch this application."

- **First argument:** The protocol name without `://` — just `cherrystudio`, not `cherrystudio://`
- **Second argument (optional):** The path to the executable to launch. Only needed in dev mode.

### Why Dev Mode and Packaged Mode Are Different

**In development mode:**
- Your app is not a standalone executable. It runs as: `electron ./src/main/index.ts`
- The OS does not know what "Cherry Studio" is — it only sees the `electron` binary
- So you must tell the OS: "run the electron binary with my project path as an argument"
- `process.defaultApp` is `true` in dev mode (it means "this is the default Electron app, not a packaged one")
- The registration passes extra arguments: `electron . cherrystudio://url`

**In packaged mode:**
- Your app IS a standalone executable: `Cherry Studio.app` (macOS) or `Cherry Studio.exe` (Windows) or `cherry-studio` (Linux)
- The OS already knows about your app from the installer
- Registration is straightforward: "when `cherrystudio://`, open Cherry Studio"
- No extra path arguments needed

**Simple mental model:**
```
Dev mode:   OS → electron binary → "also, here are the script arguments"
Packaged:   OS → Cherry Studio executable → the app knows what to do
```

**Blind spot — "What is `process.defaultApp`?":** This is a property that Electron sets on the Node.js `process` object. It is `true` when the app is running from the default Electron binary (as during development with `electron .`) and `false` or `undefined` when running as a packaged application. Code can use this to distinguish dev from production behavior without relying on environment variables.

### Universal Reuse — Your Own Electron Project

```typescript
import { app } from 'electron';

function registerProtocol() {
  const protocol = 'myapp';

  if (process.defaultApp) {
    // Dev mode: point to the electron binary with script path
    app.setAsDefaultProtocolClient(protocol, process.execPath, [
      path.resolve(process.argv[1])
    ]);
  } else {
    // Packaged mode: register the app directly
    app.setAsDefaultProtocolClient(protocol);
  }
}

// Call this BEFORE app.whenReady()
registerProtocol();
```

---

## 2. Where Deep Links Arrive

Electron can deliver protocol URLs from multiple places:

- `app.on('open-url', ...)` on macOS when the app is already running
- `process.argv` when the app is launched by a protocol URL
- `app.on('second-instance', ...)` when a new launch is redirected to the already-running instance

Cherry Studio supports all three, which is the correct pattern for a cross-platform deep-link-capable Electron app.

### Why Three Entry Points? (Operating System Differences)

The fundamental reason is that macOS, Windows, and Linux deliver protocol URLs to apps differently at the OS level.

**macOS — `open-url` event:**
- macOS has a concept of "launch services" that routes URLs to the appropriate app
- If the app is already running, the OS delivers the URL via an Apple Event (`open-url`)
- The app never relaunches — the existing process just receives a new event
- Electron translates this Apple Event into the `open-url` event on the `app` object

**Windows — new process with command-line arguments:**
- Windows registers protocol handlers in the registry
- When a protocol URL is clicked, Windows always launches a new process
- The URL is passed as a command-line argument to that process
- `process.argv` (the array of command-line arguments) contains the URL
- If single-instance lock is active, this second process quits and the URL is forwarded via `second-instance`

**Linux — similar to Windows:**
- Linux uses `.desktop` files and the `xdg-open` system
- Protocol URLs cause a new process launch with the URL in `process.argv`
- Behavior varies by desktop environment, but the `second-instance` pattern handles it

**Simple principle:** Your code must handle the superset of all platform behaviors. Even if you only develop on macOS, your code must handle the Windows/Linux paths too — otherwise deep links will silently fail for users on those platforms.

### Universal Reuse — Your Own Electron Project

The three-entry-point recipe for every Electron app with a custom protocol:

```typescript
import { app } from 'electron';

let deepLinkUrl: string | null = null;

// Entry point 1: macOS (app already running)
app.on('open-url', (event, url) => {
  event.preventDefault();  // Prevent default OS handling
  deepLinkUrl = url;
  handleDeepLink(url);
});

// Entry point 2: Windows/Linux (second instance blocked by lock)
app.on('second-instance', (event, commandLine) => {
  const url = commandLine.find(arg => arg.startsWith('myapp://'));
  if (url) {
    deepLinkUrl = url;
    handleDeepLink(url);
  }
});

// Entry point 3: All platforms (first launch with URL)
// Check in app.whenReady():
app.whenReady().then(() => {
  const url = process.argv.find(arg => arg.startsWith('myapp://'));
  if (url) handleDeepLink(url);
});
```

---

## 3. Why Single-Instance Lock Matters

Because Cherry Studio uses `requestSingleInstanceLock()`, a second launch is not allowed to keep running as a separate app instance.

Instead:
- the first instance is brought to the foreground
- the deep-link URL from the second launch is parsed and handled there

Without this, protocol callbacks could open duplicate app processes or lose the callback payload.

### Plain-Language Explanation

**The problem without single-instance lock:**

```
1. Cherry Studio is running (Instance A)
2. User clicks cherrystudio://oauth/callback?code=abc123 in browser
3. Windows launches a second copy (Instance B) with the URL
4. Instance B has the URL but no windows, no user session, no context
5. Instance B either: shows a second window (confusing) or sits there with the URL doing nothing
6. Instance A never receives the URL
7. OAuth login fails silently
```

**The solution with single-instance lock:**

```
1. Cherry Studio is running (Instance A)
2. User clicks cherrystudio://oauth/callback?code=abc123 in browser
3. Windows tries to launch Instance B with the URL
4. Instance B: "Is there already a lock?" → YES → "I should quit"
5. Instance B fires 'second-instance' event on Instance A, passing the URL
6. Instance B quits
7. Instance A receives the URL, processes the OAuth callback
8. Login succeeds!
```

**Without the lock:** duplicate instances, lost data, confused users.
**With the lock:** clean routing, single source of truth, reliable behavior.

This is why the lifecycle document (electron_main_process_lifecycle.md) emphasizes that single-instance locking and deep-link handling are tightly connected. One without the other leads to broken behavior on Windows and Linux.

---

## 4. Main-Process URL Routing

`handleProtocolUrl(url)` parses the incoming URL with the standard `URL` API and then routes by hostname:

- `cherrystudio://mcp/...` -> MCP install handling
- `cherrystudio://providers/...` -> provider/OAuth handling
- everything else -> generic payload broadcast to the renderer

The generic path sends:

```ts
mainWindow.webContents.send('protocol-data', {
  url,
  params: Object.fromEntries(params.entries())
})
```

So the renderer does not need direct access to OS launch arguments.

### Plain-Language Explanation

**How URL parsing works (the standard `URL` API):**

The built-in JavaScript `URL` class parses a URL string into its components:

```
URL: cherrystudio://mcp/install?package=github-tools&version=1.0

Parsed result:
  protocol: "cherrystudio:"
  hostname: "mcp"
  pathname: "/install"
  search:   "?package=github-tools&version=1.0"
  searchParams: URLSearchParams { "package" → "github-tools", "version" → "1.0" }
```

Using `hostname` for routing is a design choice. It means the URL structure follows the pattern:
```
cherrystudio://<category>/<action>?<parameters>
```

This is similar to how HTTP works (`https://<hostname>/<path>?<query>`) and is easy to understand, document, and extend.

### Why Route in Main Process Instead of Renderer?

**Security:** The main process validates the URL before forwarding. Malformed or malicious URLs are rejected before reaching the UI.

**Availability:** The renderer might not exist yet when a protocol URL arrives (the app was launched FROM a protocol URL, so the window is still being created). The main process is always running.

**Capability:** The main process has the authority to install packages, write files, and make network requests that are needed to handle OAuth and MCP actions. The renderer should only display results, not perform privileged operations.

**The generic fallback (`else` branch):** Unknown URL hosts are forwarded to the renderer as a generic payload. This allows the renderer (React UI) to handle custom deep-link routes without needing changes to the main process code. The main process forwards the data; the renderer decides what to do with it.

**Blind spot — "What is `webContents.send()`?":** This is how the main process pushes a message to a specific renderer window. The first argument is a channel name (like `'protocol-data'`), and the second is the payload. The renderer listens on the same channel via `ipcRenderer.on('protocol-data', callback)`. This is a one-way push (main → renderer), not a request-response.

### Universal Reuse — Your Own Electron Project

```typescript
import { URL } from 'url';

function handleProtocolUrl(urlString: string, mainWindow: BrowserWindow) {
  try {
    const parsed = new URL(urlString);

    // Route by hostname
    switch (parsed.hostname) {
      case 'settings':
        // Navigate to a specific settings page
        mainWindow.webContents.send('navigate', `/settings/${parsed.pathname}`);
        break;

      case 'oauth':
        // Handle OAuth callback with authorization code
        const code = parsed.searchParams.get('code');
        handleOAuthCallback(code);
        break;

      default:
        // Forward unknown routes to renderer
        mainWindow.webContents.send('deep-link', {
          url: urlString,
          params: Object.fromEntries(parsed.searchParams.entries())
        });
    }
  } catch (err) {
    logger.error('Invalid protocol URL:', urlString);
  }
}
```

---

## 5. Renderer Consumption Through Preload

The preload layer exposes:

```ts
window.api.protocol.onReceiveData(callback)
```

That gives the renderer a clean subscription-based interface for protocol payloads. This is safer and cleaner than letting renderer code read from Node APIs directly.

### Plain-Language Explanation

**Why preload instead of direct IPC in renderer?**

The renderer process (by default) cannot access `ipcRenderer` directly. Electron's security best practice is to disable Node integration in the renderer (`nodeIntegration: false`) and use a preload script to expose a controlled API surface.

The preload script acts as a secure bridge:

```
Main Process                  Preload Script                 Renderer
─────────────                ───────────────                ────────
webContents.send()    →     contextBridge.expose    →     window.api.protocol
('protocol-data')           InMainWorld()                  .onReceiveData()
```

**The subscription pattern (`onReceiveData(callback)`):**

Instead of the renderer polling "is there new protocol data yet?", it subscribes once and is notified whenever data arrives. This is like subscribing to a newsletter rather than checking the mailbox every 5 minutes.

```typescript
// Renderer code (React component)
useEffect(() => {
  const unsubscribe = window.api.protocol.onReceiveData((data) => {
    console.log('Received protocol data:', data.url, data.params);
    // Handle the deep link in the UI
  });

  return unsubscribe;  // Clean up when component unmounts
}, []);
```

**Blind spot — "What is `contextBridge.exposeInMainWorld()`?":** This is the Electron API that safely copies specific functions from the preload script's isolated world into the renderer's JavaScript world. Without it, the preload and renderer run in separate JavaScript contexts that cannot see each other's variables. `exposeInMainWorld('api', { ... })` creates `window.api` in the renderer context, with only the functions you explicitly expose.

### Universal Reuse — Your Own Electron Project

Preload script exposing protocol data:

```typescript
// preload.ts
import { contextBridge, ipcRenderer } from 'electron';

contextBridge.exposeInMainWorld('api', {
  protocol: {
    onReceiveData: (callback: (data: { url: string; params: Record<string, string> }) => void) => {
      const handler = (_event: any, data: any) => callback(data);
      ipcRenderer.on('protocol-data', handler);
      // Return unsubscribe function
      return () => ipcRenderer.removeListener('protocol-data', handler);
    }
  }
});
```

---

## 6. Why This Matters For OAuth

The login flows described elsewhere rely on redirect URIs such as:

```text
cherrystudio://oauth/callback
```

The provider redirects the system browser to that URI, the OS reopens Cherry Studio, and the main process forwards the parsed callback data to whichever feature is waiting for it.

In Electron terms, the custom protocol acts as the bridge between external browser authentication and the running desktop app.

### The Complete OAuth Flow (End-to-End Walkthrough)

```
Step 1: User clicks "Sign in with Google" in Cherry Studio
  → Main process opens system browser:
    shell.openExternal('https://accounts.google.com/o/oauth2/auth?' +
      'redirect_uri=cherrystudio://providers/google/callback&' +
      'client_id=...&' +
      'scope=email profile')

Step 2: User logs in to Google in their browser
  → Google shows "Cherry Studio wants to access your account"
  → User clicks "Allow"

Step 3: Google redirects browser to cherrystudio://providers/google/callback?code=AUTH_CODE
  → Browser doesn't know what cherrystudio:// means
  → Browser asks OS: "What handles cherrystudio://?"
  → OS: "Cherry Studio does"

Step 4: OS delivers URL to Cherry Studio
  → macOS: via open-url event (already running)
  → Windows/Linux: launches new process → single-instance lock → second-instance event

Step 5: ProtocolClient.ts receives the URL
  → parse: hostname = "providers"
  → route: OAuth handler

Step 6: OAuth handler extracts the authorization code
  → Makes server-to-server request to Google:
    POST https://oauth2.googleapis.com/token
    { code: "AUTH_CODE", client_secret: "...", grant_type: "authorization_code" }
  → Google responds: { access_token: "...", refresh_token: "..." }

Step 7: Login complete
  → Tokens stored securely
  → Main process notifies renderer via IPC
  → UI updates to show logged-in state
```

**Blind spot — "Why is there both an authorization code AND an access token?":** Security. The authorization code is visible in the browser URL (step 3) — anyone looking at the browser history could see it. But the code is useless without the client secret (a private key known only to Cherry Studio's server-side code). The access token is never exposed in a URL — it is obtained in a server-to-server request (step 6) that includes the secret. This two-step exchange (code in public → token in private) is the OAuth 2.0 "Authorization Code Grant" and is the most secure OAuth flow for desktop apps.

---

## 7. Linux AppImage Deep-Link Support

Packaged Linux AppImages do not automatically gain protocol registration in every environment. Cherry Studio handles this with `setupAppImageDeepLink()`.

When running as an AppImage on Linux it:
- finds the current executable path
- creates `~/.local/share/applications/cherrystudio-url-handler.desktop`
- writes an `Exec=... %U` line
- declares `MimeType=x-scheme-handler/cherrystudio;`
- runs `update-desktop-database`

This is a project-specific example of Electron packaging support that must be added outside the normal protocol registration API.

### Plain-Language Explanation

**Why AppImages need extra work for protocol registration:**

A `.desktop` file is how Linux knows about installed applications. When you install an app through a package manager (apt, dnf, etc.), the package includes a `.desktop` file that is placed in a system directory. The package manager runs `update-desktop-database` to refresh the system's application registry.

But AppImages are downloaded and run directly — no installer, no package manager. Nothing places a `.desktop` file or registers the app with the system. So Cherry Studio must do this registration itself the first time it runs.

**What each part means:**

| File/Command | Purpose |
|-------------|---------|
| `~/.local/share/applications/` | The per-user directory where `.desktop` files live (no root needed) |
| `cherrystudio-url-handler.desktop` | The desktop entry file. The `-url-handler` suffix is a convention that tells the system "this entry is specifically for handling URLs" |
| `Exec=/path/to/cherry-studio %U` | The command to run when handling a URL. `%U` means "the URL goes here as an argument" |
| `MimeType=x-scheme-handler/cherrystudio;` | Tells the system "this app handles URLs with the cherrystudio scheme." The `x-scheme-handler/` prefix is a standard MIME type for custom URL schemes |
| `update-desktop-database` | Refreshes the system's cache of `.desktop` files so the new handler is recognized immediately |

**Why this must be done at runtime, not build time:** The build server does not know where the user will place the AppImage file. The path in the `Exec=` line must point to the actual location of the AppImage on the user's computer. Only the running app knows its own location.

**Blind spot — "What is a MIME type?":** MIME (Multipurpose Internet Mail Extensions) types were originally designed for email attachments but are now used broadly to identify types of files and resources. `text/html` identifies HTML files, `image/png` identifies PNG images. `x-scheme-handler/cherrystudio` is a custom MIME type that means "URLs with the cherrystudio scheme." The `x-` prefix marks it as an unofficial/experimental type.

### Universal Reuse — Your Own Electron Project

If your Electron app ships as an AppImage, you need this setup:

```typescript
import { app } from 'electron';
import * as fs from 'fs';
import * as path from 'path';
import { exec } from 'child_process';

function setupAppImageDeepLink() {
  if (process.platform !== 'linux') return;

  const desktopFileDir = path.join(
    process.env.HOME!,
    '.local/share/applications'
  );
  const desktopFilePath = path.join(
    desktopFileDir,
    'myapp-url-handler.desktop'
  );

  const desktopFileContent = `[Desktop Entry]
Name=MyApp URL Handler
Exec=${process.execPath} %U
Type=Application
NoDisplay=true
MimeType=x-scheme-handler/myapp;
`;

  fs.mkdirSync(desktopFileDir, { recursive: true });
  fs.writeFileSync(desktopFilePath, desktopFileContent);

  exec('update-desktop-database ' + desktopFileDir, (err) => {
    if (err) logger.error('Failed to update desktop database', err);
  });
}
```

Run this during the `app.whenReady()` phase.

---

## 8. Practical Takeaways

- Register protocol handlers differently in dev and packaged modes
- Support `open-url`, startup args, and `second-instance`
- Keep protocol parsing in main process
- Forward parsed data to renderer through preload IPC
- Expect extra Linux work for AppImage deep-link support

---

## 9. How To Apply This Knowledge In Development

Use this document when an external actor must send structured input into Cherry Studio.

Choose the right design:
- Use the custom protocol when the source is the OS, browser, installer, or another desktop app.
- Parse and validate the URL in main process, then forward only the required data to renderer.
- Keep protocol handlers generic enough to survive startup, relaunch, and already-running cases.
- Treat dev registration and packaged registration as separate execution paths and test both.

Practical usage pattern:
1. Define a stable `cherrystudio://...` route shape for the new feature.
2. Decide whether the handler performs privileged work in main process or only forwards data to renderer.
3. Add handling for all three entry paths: startup args, `open-url`, and `second-instance`.
4. Verify the existing window is focused or created before sending IPC payloads.

Common mistakes this avoids:
- Handling only one platform entry point.
- Parsing deep links in renderer where OS launch data is unavailable or unsafe.
- Losing callback data when a second launch redirects to the first instance.
- Shipping a feature that works in development but is not registered correctly after packaging.

---

## 10. Typical Application Scenarios

- Add a new OAuth provider redirect such as `cherrystudio://providers/<provider>/callback`.
- Add a website action that opens the app and imports configuration or installs an MCP package.
- Add a CLI or helper tool that wakes the app and passes a payload.
- Debug a report where macOS `open-url` works but Windows only works on the first launch.

---

## 11. Relationship To The Other Electron Records

- This document depends on `electron_main_process_lifecycle.md` because protocol registration and single-instance handling are lifecycle responsibilities.
- It depends on `electron_window_tray_menu_architecture.md` because successful deep-link handling often requires bringing an existing window to the foreground.
- It may interact with `electron_webview_session_management.md` when OAuth flows are opened in external browsers or limited internal popups.
- It must stay aligned with `electron_build_packaging_and_update_pipeline.md` because packaged builds, installers, and AppImage metadata control whether the OS can invoke the protocol at all.
