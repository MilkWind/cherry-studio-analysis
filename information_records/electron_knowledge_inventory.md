# Electron Knowledge Inventory Missing From Existing `information_records`

This file enumerates Electron-development knowledge that is implemented in Cherry Studio but was not already covered as a dedicated tutorial in the existing `information_records` set. The older records already explain the preload API surface, selection assistant internals, and some feature behavior. The missing part was the Electron shell implementation behind those features.

---

## Beginner's Prerequisites: What You Need to Know Before Reading

> **If you are an Electron absolute beginner**, read this section first. It explains the core concepts that every other document in this collection assumes you already understand.

### What Is Electron? (The 30-Second Version)

Electron is a tool that lets you build desktop applications using the same languages used to build websites: **HTML**, **CSS**, and **JavaScript**. Under the hood, Electron combines two things:

1. **Chromium** — the open-source engine that powers Google Chrome. It handles everything you see: windows, buttons, text, images, web pages.
2. **Node.js** — a system that lets JavaScript run outside a browser, giving it access to your computer's operating system (files, network, processes, etc.).

Think of it this way: a regular website runs inside a browser tab and is sandboxed for security (it cannot read your files or control your OS). An Electron app is a website that runs in its own dedicated browser window, but with carefully controlled extra powers to interact with your computer — like saving files, showing system notifications, or putting an icon in your system tray.

### The Two (Actually Three) Types of Code in Every Electron App

This is the single most important concept to understand before reading any of these documents. An Electron app is split into different "processes" — separate running programs that communicate with each other through messages:

| Process | What It Can Do | What It Cannot Do | Analogy |
|---------|---------------|-------------------|---------|
| **Main Process** | Full access to Node.js and the operating system (files, system tray, native menus, network) | Cannot directly render HTML/CSS UI | The "backend" or "engine room" of your app |
| **Renderer Process** | Renders HTML/CSS/JavaScript UI (like a browser tab) | No direct access to Node.js or the OS (by default) | The "frontend" or "user interface" of your app |
| **Preload Script** | A bridge that runs before the renderer, with limited access to both worlds | Cannot freely use all Node.js APIs | A "secure concierge" that passes specific messages between main and renderer |

**Why this split exists (simple principle):** If the renderer could directly access your file system, any malicious website embedded in your app could delete your files. The main process acts as a gatekeeper — the renderer must ask "please save this file" through a message, and the main process decides whether to honor that request.

**In Cherry Studio specifically:**
- `src/main/` = Main process code (has full Node.js powers)
- `src/renderer/` = Renderer process code (the React UI, runs like a web page)
- `src/preload/` = Preload bridge (the controlled message-passing layer)

**Blind spot filled — "What is a process?":** A process is simply a running program. When you open Notepad, that's one process. When you open Chrome, each tab is typically its own process. Processes are isolated from each other by the operating system — one crashing does not (usually) take down the others. Electron uses this isolation intentionally: if the renderer (your UI) crashes, the main process can detect it and reload the window instead of the whole app disappearing.

### What Is IPC? (Inter-Process Communication)

IPC is how the main process and renderer process talk to each other. Since they are separate processes, they cannot share variables or call each other's functions directly. Instead, they send messages through Electron's built-in messaging system.

```
Renderer: "Hey main process, user clicked 'Save', here's the data"
   ↓ (ipcRenderer.send / ipcRenderer.invoke)
Main Process: "Got it, writing to disk... done!"
   ↓ (ipcMain.handle / webContents.send)
Renderer: "Save complete, updating the UI"
```

**Blind spot filled — "What is ipcMain / ipcRenderer?":**
- `ipcMain` = the message receiver on the main process side (like a phone that receives calls)
- `ipcRenderer` = the message sender on the renderer side (like a phone that makes calls)
- `ipcMain.handle()` = "When the renderer asks me to do X, here's how I'll respond" (request-response pattern)
- `webContents.send()` = the main process pushing a message to a specific renderer window without being asked (push pattern)

### What Is a `BrowserWindow`?

A `BrowserWindow` is Electron's name for "a desktop window that shows web content." Every window you see in an Electron app (the main app window, a settings popup, a mini assistant window) is a `BrowserWindow` instance. It is created in the main process and it loads an HTML file from your renderer code.

**Simple mental model:** `new BrowserWindow({...})` in Electron is like opening a new Chrome window, except you control everything about it — its size, whether it has a title bar, what URL it loads, whether it can be resized, etc.

### What Is `app` in Electron?

Throughout these documents you will see `app.something()`. The `app` object is Electron's built-in representation of your entire application. It controls lifecycle events (when the app starts, when it's ready, when it quits), and it is only available in the main process.

**Simple mental model:** `app` is like the "power button and settings" of your desktop application. It does not draw windows — it manages whether the application is running at all.

---

## How To Use This Inventory

Use this file as a reading map instead of a passive list.

- If you are changing app startup, shutdown, crash handling, protocol registration, or service boot order, start with `electron_main_process_lifecycle.md`.
- If you are changing `BrowserWindow`, tray, native menu, or close-to-tray behavior, start with `electron_window_tray_menu_architecture.md`.
- If you are adding OAuth redirects, deep-link commands, or external-app callbacks, start with `electron_protocol_oauth_and_deep_linking.md`.
- If you are embedding external pages, mini apps, or guest-content export features, start with `electron_webview_session_management.md`.
- If you are changing build targets, packaging, installer behavior, or updates, start with `electron_build_packaging_and_update_pipeline.md`.

Recommended learning order for a new contributor:

1. `electron_main_process_lifecycle.md`
2. `electron_window_tray_menu_architecture.md`
3. `electron_protocol_oauth_and_deep_linking.md`
4. `electron_webview_session_management.md`
5. `electron_build_packaging_and_update_pipeline.md`

That order matches how an Electron desktop app is assembled: boot the shell, create windows, connect OS entry points, embed guest content, then ship and update the product.

> **Beginner tip — why this order matters:** You cannot create a window before the app is ready (step 1 before step 2). You cannot handle a deep-link URL if you have no window to show the result in (step 2 before step 3). You cannot embed web content if your window does not exist (step 2 before step 4). And you cannot ship your app if you have not built all the pieces (step 5 depends on everything before it). This is not really an Electron rule — it is a logic rule: build the foundation before the walls.

---

## Group 1. Desktop App Bootstrap And Runtime Lifecycle

Completed function:
- Start the Electron app as a single-instance desktop shell with crash logging, startup switches, renderer diagnostics, service bootstrap, and shutdown cleanup.

New knowledge points:
- Why `src/main/index.ts` performs work before `app.whenReady()`
- How `requestSingleInstanceLock()` and `second-instance` support deep-link reopening
- How command-line switches are used for Windows animation fixes, Linux Wayland shortcuts, Linux WM class, and renderer crash call stacks
- How `crashReporter`, `web-contents-created`, and global process error handlers are combined
- How main-process services are bootstrapped in an Electron-safe order
- How shutdown cleanup is split across `before-quit` and `will-quit`

### Plain-Language Glossary for This Group

**"Bootstrap":** The sequence of steps that happen between "user double-clicks the app icon" and "the app window appears and is usable." Like turning on a computer — there is a boot sequence before you see the desktop.

**"Crash reporter":** A built-in system that records what went wrong when the app breaks unexpectedly. Like an airplane's black box — it logs information so developers can figure out what happened.

**"Single instance lock":** A mechanism that ensures only one copy of your app runs at a time. If the user double-clicks the app icon while it is already open, instead of opening a second copy, it brings the existing window forward.

**"Chromium switches" / "command-line switches":** Configuration flags passed to Chromium (the browser engine) before it starts. They control low-level behavior like how animations work or what GPU features are enabled. Think of them as "settings you must configure before turning on the engine."

**"Wayland":** A display system on Linux (like how Windows has its own window-drawing system). Some Linux desktops use Wayland instead of the older X11 system. Electron apps sometimes need special configuration to work correctly on Wayland.

**Blind spot — "Why can't we do everything after the app is ready?":** Chromium (the browser engine inside Electron) has a startup sequence. Some settings, like "don't use GPU hardware acceleration," must be set before Chromium initializes its graphics system. Once Chromium has started drawing things, changing these settings has no effect. This is like setting your TV's input source before turning it on — once it is displaying HDMI-1, you cannot change the physical HDMI port without restarting.

Tutorial:
- `information_records/electron_main_process_lifecycle.md`

Developer usage:
- Read this before moving code into or out of `src/main/index.ts`.
- Use it when deciding whether new logic belongs before `app.whenReady()`, inside lifecycle services, or during quit cleanup.
- Use it to avoid subtle startup bugs such as late protocol registration, duplicated app instances, or windows created before required switches are applied.

Application scenarios:
- Add a new global Chromium switch.
- Add startup diagnostics or crash monitoring.
- Register a new main-process service that must start after Electron is ready.
- Add cleanup for a long-lived resource on quit.

### Universal Reuse — How These Patterns Apply to Any Electron Project

Almost every Electron desktop app needs these same things. When starting your own Electron project from scratch, you will encounter the same questions:

1. **"Where do I put my startup code?"** → Before `app.whenReady()` for OS/protocol configuration; inside `app.whenReady()` for window creation. This rule is universal — every Electron app follows it.
2. **"How do I prevent duplicate app instances?"** → `app.requestSingleInstanceLock()`. Universal for any app that registers protocol handlers or receives OS launch events.
3. **"How do I handle crashes?"** → `crashReporter` + global error handlers. Universal for any production desktop app.
4. **"How do I clean up on exit?"** → Split between `before-quit` (mark quitting state) and `will-quit` (actual resource cleanup). Universal pattern.

---

## Group 2. Multi-Window, Tray, And Native Menu Architecture

Completed function:
- Deliver the main window, quick assistant mini window, tray behavior, close-to-tray UX, and macOS application menu as native desktop features.

New knowledge points:
- How `WindowService` centralizes `BrowserWindow` creation and lifecycle
- How `electron-window-state` persists main and mini window geometry
- How platform-specific title bar, vibrancy, Mica, and frameless settings are chosen
- How minimize-to-tray and launch-to-tray behavior are implemented per OS
- How the tray menu reacts to config changes and locale changes
- How the macOS application menu dispatches UI navigation back into the renderer

### Plain-Language Glossary for This Group

**"System tray" / "tray":** The small icons near the clock on your taskbar (Windows) or menu bar (macOS). Apps like Discord, Spotify, and antivirus software put icons there so they can stay running in the background even when their main window is closed.

**"Tray menu" / "context menu":** The small popup menu that appears when you right-click (or left-click, on some systems) a tray icon. It typically shows options like "Open," "Quit," or feature toggles.

**"Close-to-tray":** A behavior where clicking the X button on a window does not quit the app — it just hides the window. The app keeps running in the background and can be reopened from the tray icon. Many chat and music apps work this way.

**"Launch-to-tray":** Starting the app without showing its main window — only the tray icon appears. Useful for apps that should start silently when the computer boots.

**"Frameless window":** A window without the standard OS title bar (the bar at the top with the app name and minimize/maximize/close buttons). The app draws its own custom title bar instead. This gives more design control but requires handling drag, resize, and close behavior manually.

**"Vibrancy" / "Mica":** Visual effects that make a window's background semi-transparent and blend with the wallpaper behind it. "Vibrancy" is the macOS term; "Mica" is the Windows 11 term. They give apps a modern, native "frosted glass" look.

**"Native menu":** The menu bar at the very top of the screen on macOS (File, Edit, View, etc.) or the menu bar attached to each window on Windows/Linux. "Native" means the operating system draws it, not your HTML/CSS code — so it looks and feels exactly like every other app on that platform.

**"Window geometry":** The position (x, y coordinates on screen) and size (width, height) of a window. "Persisting geometry" means saving these numbers when the app closes and restoring them when it reopens.

**Blind spot — "Why do we need a WindowService instead of just creating windows directly?":** In a small app with one window, creating a `BrowserWindow` directly in your main file works fine. But as soon as you have multiple windows with different behaviors (main window, settings, mini assistant), you end up with copy-pasted window-creation code scattered everywhere. A centralized service gives you one place to manage window policy — creation, reuse, focus, event handling. This is a universal software engineering principle (centralize related logic) applied to Electron specifically.

### Universal Reuse — How These Patterns Apply to Any Electron Project

1. **Window state persistence** (`electron-window-state` or similar): Any desktop app should remember where its windows were and what size they were. Users expect this. The package `electron-window-state` is the standard solution across the Electron ecosystem.
2. **Platform-specific window chrome**: Never force the same title bar style on all operating systems. macOS users expect hidden title bars with traffic-light buttons; Windows users expect a standard title bar. Your window creation code should check `process.platform` and adjust accordingly.
3. **Close-to-tray pattern**: Any app that "stays running in the background" needs this. Messaging apps, music players, backup tools, clipboard managers — all use this exact pattern.
4. **Native menus for cross-window navigation**: On macOS, the application menu is the standard way to access app-wide actions (Preferences, About, Quit). Your Electron app should provide one even if your UI has its own navigation.

---

## Group 3. Custom Protocol, OAuth Callback Routing, And Deep Links

Completed function:
- Receive `cherrystudio://` links for OAuth callbacks and MCP/provider actions, including packaged Linux AppImage deep-link registration.

New knowledge points:
- How `setAsDefaultProtocolClient()` is registered differently in dev and packaged modes
- How protocol URLs are consumed from `open-url`, startup args, and `second-instance`
- How main-process routing splits `mcp`, `providers`, and generic callback payloads
- How the renderer receives deep-link payloads through IPC instead of parsing OS launch arguments directly
- How AppImage builds create a `.desktop` handler and refresh the desktop database

### Plain-Language Glossary for This Group

**"Protocol" / "Custom protocol" / "URI scheme":** Just like `https://` tells your computer "open this in a web browser," a custom protocol like `cherrystudio://` tells your computer "open this in the Cherry Studio app." It is a way for websites, other apps, or the operating system itself to send data into your desktop app.

**"Deep link":** A URL that opens a specific screen or triggers a specific action inside an app, rather than just launching the app to its home screen. For example, `cherrystudio://settings/updates` might open the app directly to the update settings page.

**"OAuth callback":** OAuth is how "Sign in with Google/GitHub/Apple" works. The flow is: (1) your app opens the user's browser to the provider's login page, (2) the user logs in, (3) the provider redirects the browser back to your app with a special code. That redirect URL (e.g., `cherrystudio://oauth/callback?code=abc123`) is the "OAuth callback." Without custom protocol support, the browser cannot send the login result back to a desktop app.

**"AppImage":** A portable Linux application format. Instead of installing an app through a package manager, the user downloads a single `.AppImage` file that runs directly. It is similar to a `.app` bundle on macOS or a portable `.exe` on Windows.

**"`.desktop` file":** A configuration file on Linux that tells the system "this application exists, here is its icon, here is what it can open." It is what makes an app appear in the Linux application launcher and associates it with file types or URL protocols.

**Blind spot — "Why does protocol registration work differently in dev vs packaged mode?":** When you run `npm run dev`, Electron is not running your app directly — it is running the Electron executable and passing your project's entry file as an argument. The OS does not know about your app; it only knows about the Electron binary. So in dev mode, protocol registration must explicitly tell the OS "when you see `cherrystudio://`, run the Electron binary with my project as the argument." In packaged mode, your app is a single executable with its own identity, so the OS can register the protocol directly to your app.

### Universal Reuse — How These Patterns Apply to Any Electron Project

1. **Custom protocol for OAuth**: This is the standard way to handle OAuth in Electron apps. The alternative (running a local HTTP server to catch the callback) is fragile and can conflict with firewalls. Custom protocols are the recommended approach.
2. **Three-entry-point pattern** (`open-url` + `process.argv` + `second-instance`): This is the universal cross-platform recipe for catching protocol URLs in Electron. Different operating systems deliver protocol URLs through different channels, and your app must handle all three.
3. **Main-process parsing, renderer consumption**: Never expose raw OS launch data to your UI code. Parse and validate in the main process first. This is a universal security principle.
4. **AppImage desktop integration**: Any Electron app distributed as an AppImage on Linux needs this exact `.desktop` file dance. It is not handled automatically.

---

## Group 4. Embedded Webview Session And Guest-Content Control

Completed function:
- Run embedded mini-app webviews with a shared partition, custom user agent, controlled external-link behavior, guest hotkey forwarding, and export helpers.

New knowledge points:
- Why this project enables `webviewTag` and uses `persist:webview`
- How the webview session strips the app/electron signature from the user agent
- How `onBeforeSendHeaders` injects `Accept-Language` and per-domain UA overrides
- How `setWindowOpenHandler()` is used for both main-window guest pages and standalone webviews
- How guest `before-input-event` shortcuts are bridged back to the host renderer
- How Electron can print guest pages to PDF or serialize them to HTML from main process

### Plain-Language Glossary for This Group

**"Webview" (`<webview>` tag):** A special Electron HTML element that embeds an entire separate web page inside your app, running in its own isolated process. Think of it as "a Chrome tab embedded inside your app window." Unlike an `<iframe>` (which shares the host page's process and permissions), a `<webview>` has its own separate renderer process with its own cookies, storage, and security context.

**"Session" / "Session partition":** In Chromium (and therefore Electron), a "session" is a container for browsing data: cookies, local storage, cache, permissions. A "partition" is just a named session. The default partition is the one your main app uses. `persist:webview` means "create a separate session named 'webview' that saves its data to disk so it survives app restarts."

**"User agent" (UA):** A string of text that every browser sends with every web request, identifying itself. For example: `"Mozilla/5.0 (Windows NT 10.0) CherryStudio/1.0 Electron/28.0"`. Websites read this to decide what version of their page to serve. If a website sees "Electron" in the user agent, it might serve a degraded version or block access — so Cherry Studio removes those tokens for embedded content.

**"Accept-Language header":** A request header that tells a website "I prefer content in this language." Setting this at the session level ensures embedded pages automatically display in the user's chosen language.

**"Content Security Policy" (CSP):** A security mechanism where websites declare "I am only allowed to load resources from these specific sources." Cherry Studio strips CSP headers from embedded content responses so those pages can function inside the webview without being blocked by overly restrictive policies.

**Blind spot — "Why use `<webview>` instead of `<iframe>`?":**
- `<iframe>` runs in the same process as your app. If the embedded page crashes or freezes, your entire app can freeze.
- `<iframe>` shares cookies and storage with your main app, which is a security risk.
- `<webview>` runs in a separate process. If it crashes, only that embedded page dies. Your main app keeps running.
- `<webview>` lets you control the embedded page's session, user agent, and permissions independently from your main app.

**Blind spot — "What is a 'guest page' / 'guest content'?":** Throughout these documents, "guest" refers to the external web page loaded inside a `<webview>` or opened as a popup. It is "guest" because it is not part of your app's own code — it is a visitor from another website.

### Universal Reuse — How These Patterns Apply to Any Electron Project

1. **Use `<webview>` for untrusted or third-party content**: If your app embeds external websites, use `<webview>` for process isolation. Never load untrusted content in an `<iframe>`.
2. **User-agent normalization**: Any Electron app that embeds external web content should strip the `Electron/...` token from the user agent. Many websites behave poorly when they detect Electron.
3. **`setWindowOpenHandler` for popup control**: Every Electron app with embedded content should control what happens when guest pages try to open new windows. The default behavior is often not what you want.
4. **Guest shortcut bridging**: Keyboard shortcuts (Ctrl+F, Escape, etc.) are captured by the guest page, not your host app. You must explicitly forward them if you want your app to respond. This is a universal pain point.

---

## Group 5. Packaging, Multi-Entry Build, And Auto-Update Pipeline

Completed function:
- Build and package a multi-window Electron app for Windows, macOS, and Linux, then update it through Electron Updater with channel and mirror logic.

New knowledge points:
- How `electron-vite` builds separate `main`, `preload`, and multi-entry `renderer` targets
- Why the renderer build contains `index.html`, `miniWindow.html`, `selectionToolbar.html`, `selectionAction.html`, and `traceWindow.html`
- How `electron-builder.yml` defines protocol support, platform targets, file filtering, `asarUnpack`, and hooks
- How updater feed URLs are chosen from remote config, release channel, mirror, and current app version
- Why auto-download is allowed but auto-install-on-quit is disabled
- How localized release notes are extracted from marker blocks before being shown in the renderer

### Plain-Language Glossary for This Group

**"Build" / "Bundle":** The process of converting your source code (TypeScript, JSX, CSS modules) into plain JavaScript/HTML/CSS files that can actually run. During development, you have many files with imports; the build step combines and optimizes them into a few output files.

**"electron-vite":** A build tool specifically designed for Electron apps. It knows about Electron's three code types (main, preload, renderer) and builds each one appropriately. It is built on top of Vite, which is a fast build tool that handles TypeScript, JSX, hot reload, and more.

**"Entry point" / "Entry HTML":** The HTML file that a `BrowserWindow` loads when it opens. Each separate window in your app typically has its own entry HTML. `index.html` → main window; `miniWindow.html` → quick assistant window; etc.

**"electron-builder":** The standard tool for packaging Electron apps into installers. It takes your built code and wraps it into a `.exe` installer (Windows), `.dmg` disk image (macOS), or `.AppImage`/`.deb`/`.rpm` (Linux).

**"asar" (Atom Shell Archive):** A file format that bundles many files into a single archive file (similar to a `.zip` file but optimized for Electron). By default, `electron-builder` packs your entire app code into `app.asar`. This makes distribution cleaner and slightly faster. However, some things (native binaries, runtime resources) need to be real files on disk — they cannot work from inside an archive.

**"asarUnpack":** A configuration in `electron-builder.yml` that says "keep these specific files outside the asar archive as regular files." Used for things like `.node` native addons, external executables, or resource files that must be accessible as regular file paths.

**"Auto-update":** The mechanism that lets an Electron app download and install new versions of itself. `electron-updater` is the standard library for this. It checks a remote server for new versions, downloads the update in the background, and applies it when the user restarts the app.

**"Release channel":** Different tiers of updates. Common channels are: `latest` (stable, everyone gets it), `beta` (early testing), `rc` (release candidate, almost stable). Users can choose which channel they want.

**"NSIS installer":** NSIS (Nullsoft Scriptable Install System) is the most common installer format for Windows. It is what creates the familiar "Setup Wizard" experience where you click Next → Next → Install.

**Blind spot — "Why do we need separate build targets for main, preload, and renderer?":** These three pieces run in fundamentally different environments. The main process runs in Node.js (needs CommonJS or ESM with Node built-ins), the preload runs in a stripped-down Node-like environment (needs specific APIs), and the renderer runs in a browser environment (needs standard web APIs, no Node). A single build configuration cannot produce correct output for all three — each needs its own rules about what externals to keep, how to handle imports, and what format to output.

### Universal Reuse — How These Patterns Apply to Any Electron Project

1. **Separate renderer entries for separate windows**: Do not shoehorn multiple windows into one SPA entry with conditional routing. Each window type deserves its own HTML entry point. This keeps bundle sizes small and avoids loading code for windows that never open.
2. **`asarUnpack` for native dependencies**: Native Node.js modules (`.node` files), external binaries, and anything that must be accessed through a real OS filesystem path usually need `asarUnpack`. Ordinary Node `fs.readFileSync()` reads can often work directly from `app.asar`, so do not treat unpacking as a blanket rule.
3. **Channel-aware update feeds with mirror fallback**: If your app has users in China and elsewhere, the GitHub-based update feed may be slow or blocked in China. A mirror-based fallback (like GitCode) is a practical necessity for global distribution.
4. **Auto-download yes, auto-install no**: Letting the app download updates in the background is convenient. Silently replacing the app on quit can surprise users and lose their work state. The explicit-install pattern is user-friendlier.

---

## Coverage Notes

Existing records that already covered adjacent Electron topics:
- `information_records/window-api-reference.md`
- `information_records/selection_assistant_implementation.md`
- `information_records/login_methods_explanation.md`
- `information_records/package-scripts-explanation.md`

Those files remain useful, but they do not explain the main-process and packaging patterns above in enough project-specific detail.

## Relationship Between The Electron Topics

These documents describe one continuous pipeline rather than five isolated topics.

- Lifecycle is the entry point. It decides when protocol registration, tray creation, window creation, and service startup are safe.
- Window/tray/menu architecture depends on lifecycle because the app shell can only create and manage native UI after Electron is ready.
- Protocol/deep-link handling depends on lifecycle and window management because a protocol callback usually needs the existing main window to be focused and messaged.
- Webview/session management depends on window architecture because guest content lives inside windows and inherits session and popup policy from shell decisions.
- Build/packaging/update work depends on all previous topics because the packaged app must preserve protocol handlers, preload boundaries, window entry files, and runtime assets.

In practice, a feature often touches several documents at once. Example: adding an OAuth-powered mini app may require lifecycle changes for startup ordering, protocol changes for callbacks, window/webview changes for guest behavior, and packaging changes if new assets or updater notes are needed.

> **Beginner insight — "Why are these separate documents if they are so connected?":** They are separated for the same reason a car manual has separate chapters for the engine, transmission, and electrical system — each topic is deep enough to deserve its own focused explanation. But like a car, you need all systems working together for the vehicle to function. When debugging a problem, trace the signal through each layer: "Did the protocol URL arrive? (Group 3) → Is the app alive to receive it? (Group 1) → Is there a window to show the result? (Group 2) → Is the embedded content loading correctly? (Group 4) → Did the packaged build preserve all of this? (Group 5)"
