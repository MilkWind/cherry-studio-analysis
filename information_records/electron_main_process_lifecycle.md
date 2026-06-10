# Electron Main Process Lifecycle In Cherry Studio

This tutorial explains how Cherry Studio boots, guards, monitors, and shuts down its Electron main process. The core implementation lives in `src/main/index.ts`.

---

## Beginner's Prerequisites: Concepts You Need Before Reading

> This section fills in the "unknown unknowns" — the concepts this document assumes you know but a beginner might not. Read this first.

### What Even Is a "Main Process"? (Expanded)

Every Electron app runs at least two separate programs simultaneously:

1. **The Main Process** (what this document is about) — This is the "boss" of your app. It runs with full Node.js powers: it can read/write files, access the network at a low level, control windows, talk to the operating system. There is exactly ONE main process per app. If it dies, the entire app dies.

2. **The Renderer Process** (your UI) — Each window gets its own renderer process. Renderers are like browser tabs: they display HTML/CSS/JavaScript but are sandboxed. They cannot touch the file system or OS directly.

**Simple analogy:** Think of a restaurant. The main process is the kitchen manager — they control the building, the equipment, the inventory. Renderer processes are the waitstaff — each serves their own table (window), interacts with customers (users), and relays orders (IPC messages) to the kitchen. The waitstaff cannot walk into the walk-in freezer and rearrange inventory (file system access), but they can ask the manager to do it.

### What Does "Lifecycle" Mean Here?

"Lifecycle" refers to the entire lifespan of the main process — from the moment the operating system launches your app until the moment it fully exits. Key moments in this lifespan:

```
App launched → Setup → Ready → Running → Quitting → Exited
              ↑        ↑       ↑          ↑          ↑
         (pre-ready)  (ready) (normal   (cleanup)  (gone)
                               operation)
```

Each of these phases has different capabilities and restrictions. You cannot create a window during "Setup" because Electron is not ready yet. You should not start new tasks during "Quitting" because the app is shutting down.

### Key Electron APIs Referenced in This Document

| API | What It Does (Plain Language) | Available During |
|-----|------------------------------|------------------|
| `app.whenReady()` | Returns a Promise that resolves when Electron is fully initialized and ready to create windows | Pre-ready and later |
| `app.on('event', callback)` | "When this event happens, run this function." The main way to respond to app-level events. | Always |
| `app.requestSingleInstanceLock()` | "Am I the only copy of this app running? If not, I should quit." | Pre-ready |
| `app.commandLine.appendSwitch()` | Adds a configuration flag to Chromium before it starts | Pre-ready ONLY |
| `crashReporter.start()` | Starts recording crash information to disk | Pre-ready or ready |
| `BrowserWindow` | Creates a desktop window that shows your UI | After ready |

**Blind spot — "What is a Promise?":** A Promise is JavaScript's way of saying "this thing is not done yet, but I will let you know when it is." `app.whenReady()` returns a Promise because Electron startup takes time (loading Chromium, initializing GPU, etc.). You `await` it or use `.then()` to pause your code until Electron is actually ready. If you try to create a window before this Promise resolves, Electron will throw an error because its window-creation machinery is not yet assembled.

**Blind spot — "What is a callback?":** A callback is a function you provide that gets called later when something happens. `app.on('window-all-closed', () => { console.log('all windows closed!'); })` means "Electron, please remember this function, and call it when all windows have been closed." The function does not run immediately — it waits for the event.

---

## 1. Why This File Runs Early

`src/main/index.ts` imports `./bootstrap` before the rest of the app. The comment at the top is important: some initialization must happen before Electron is fully ready, especially anything that affects app data paths or early environment setup.

The main rule is:
- Do OS-level or process-level setup before `app.whenReady()`
- Do `BrowserWindow`, tray, and most Electron UI work inside `app.whenReady()`

### Why This Rule Exists (Simple Principle)

Electron is built on top of Chromium, and Chromium has its own startup sequence. Think of it like starting a car engine:

1. You configure things that affect HOW the engine starts (fuel mixture, ignition timing) → **BEFORE turning the key** → This is the "before `app.whenReady()`" phase
2. You turn the key and wait for the engine to stabilize → **`await app.whenReady()`**
3. You can now drive: steer, accelerate, use the radio → **This is the "after ready" phase**

Specifically, these things MUST happen before Chromium initializes:
- **Command-line switches** (`app.commandLine.appendSwitch`) — These configure the Chromium engine itself. Once Chromium has started, switches have no effect.
- **Protocol registration** (`app.setAsDefaultProtocolClient`) — The OS needs to know about your protocol before the app is fully alive.
- **Crash reporter setup** — You want crash recording active before anything can crash.
- **Single-instance lock** — You want to check if another copy is running before doing expensive startup work.

### Universal Reuse — Your Own Electron Project

In ANY Electron app you build, the entry file (`main/index.ts` or `main.js`) should follow this same structure:

```typescript
// 1. Pre-ready: switches, protocol, crash reporter, single-instance lock
app.commandLine.appendSwitch('some-switch');
crashReporter.start({ uploadToServer: false });
const gotLock = app.requestSingleInstanceLock();
if (!gotLock) { app.quit(); return; }

// 2. Wait for ready
app.whenReady().then(() => {
  // 3. Create windows, tray, menus here
  const mainWindow = new BrowserWindow({...});
  mainWindow.loadFile('index.html');
});
```

This three-part structure is the universal skeleton of virtually every Electron main process.

---

## 2. Pre-Ready Runtime Setup

Before `app.whenReady()`, the project configures several Electron and process behaviors:

- `crashReporter.start(...)` enables local crash reports
- `app.disableHardwareAcceleration()` is applied if the config says so
- Windows gets `wm-window-animations-disabled` to avoid transparent-window flashing
- Linux Wayland gets `enable-features=GlobalShortcutsPortal` so global shortcuts work better
- Linux also gets WM class/name switches so the window manager identifies the app correctly
- All platforms get `DocumentPolicyIncludeJSCallStacksInCrashReports,EarlyEstablishGpuChannel,EstablishGpuChannelAsync`

The interesting Electron knowledge point is that many app-wide Chromium switches only work if added before windows are created.

### Plain-Language Explanation of Each Pre-Ready Action

#### `crashReporter.start(...)` — Crash Recording

**What it does:** Tells Electron "if anything in this app crashes (main process or any renderer), write a crash report to disk so developers can diagnose what went wrong."

**Why we do this:** Without it, when the app crashes, there is zero information about why. The user just sees "the app disappeared." Crash reports are like an airplane's black box — they record the state of the program at the moment of failure.

**Simple principle:** Chromium has a built-in crash-reporting subsystem. `crashReporter.start()` activates it and tells it where to save reports. These reports contain call stacks (the list of function calls that led to the crash) and system information.

**Universal reuse:** In production, you would configure `uploadToServer: true` and provide a server URL to collect crash reports from users. In development or local-only mode, `uploadToServer: false` with a local directory is sufficient. Every production Electron app should enable crash reporting.

#### `app.disableHardwareAcceleration()` — GPU Disable

**What it does:** Tells Chromium "do not use the graphics card (GPU) for rendering — use the CPU instead."

**Why we do this (conditional):** Some computers have buggy GPU drivers that cause visual glitches, flickering, or crashes. This is an escape hatch: if a user reports rendering problems, they can enable a setting that disables GPU rendering. The tradeoff is that CPU rendering is slower, but it is reliable.

**Simple principle:** Normally, Chromium offloads graphics work to the GPU because GPUs are optimized for drawing. But GPU drivers vary widely in quality, especially on Linux. Disabling hardware acceleration makes rendering slower but eliminates a whole category of GPU-driver-related bugs.

**Blind spot — "What is hardware acceleration?":** In normal operation, your CPU (general-purpose processor) delegates drawing tasks to your GPU (specialized graphics processor) because the GPU is hundreds of times faster at graphics math. "Disabling hardware acceleration" means "CPU, you do all the graphics math yourself." This is slower but more compatible.

**Universal reuse:** This is a common option in Electron app settings. Provide a checkbox like "Disable hardware acceleration (may fix visual glitches)" and restart the app with this flag when enabled.

#### Windows: `wm-window-animations-disabled`

**What it does:** Disables a specific Windows feature that animates windows when they appear.

**Why we do this:** On Windows, transparent (frameless) Electron windows can flash or flicker during the animation that plays when a window first appears. Disabling this animation prevents the flashing.

**Simple principle:** Windows has a built-in animation system for window appearance ("window manager animations"). When Electron creates a transparent frameless window (no standard Windows title bar), the animation system does not know how to handle the transparency correctly, causing a visible flash of white or black. Disabling the animation avoids this.

**Universal reuse:** If your Electron app uses frameless or transparent windows on Windows, you will likely encounter this flashing. Adding this switch is a standard fix.

#### Linux Wayland: `enable-features=GlobalShortcutsPortal`

**What it does:** Enables a newer, more secure way for apps to register global keyboard shortcuts on Linux systems running Wayland.

**Why we do this:** Traditional global shortcut APIs (like `globalShortcuts.register`) do not work well on Wayland because Wayland's security model prevents apps from snooping on each other's keyboard input. The "Portal" API provides a user-consent-based alternative.

**Simple principle:** On X11 (the older Linux display system), any app could listen to all keyboard input system-wide — convenient but insecure. Wayland changed this so apps cannot eavesdrop on each other. The GlobalShortcutsPortal is a bridge: the app asks the desktop environment "please give me Ctrl+Space," and the desktop environment (not the app) handles the registration.

**Blind spot — "What is Wayland?":** Wayland is a newer display protocol for Linux that replaces X11. It is more secure and modern but has different rules. Ubuntu, Fedora, and other major distributions now default to Wayland. If your Electron app has global shortcuts, you need to handle both X11 and Wayland.

#### Linux: WM Class/Name Switches

**What it does:** Sets the "window manager class" and "window manager name" — identifiers that the Linux desktop uses to recognize your application.

**Why we do this:** On Linux, the window manager (the system software that draws window borders and manages the taskbar/dock) identifies apps by their WM_CLASS property. Without setting these correctly:
- The app icon might not appear in the dock/taskbar
- The app might be grouped incorrectly in the task switcher
- The `.desktop` file association might not work

**Simple principle:** On Linux, when a window appears, the window manager asks "who are you?" The app must respond with its WM_CLASS. If this is wrong, the desktop environment cannot match the window to the correct application icon or launcher entry.

**Universal reuse:** Any Electron app targeting Linux should set the WM_CLASS. The standard approach is:
```typescript
app.commandLine.appendSwitch('wm-class', 'YourAppName');
app.commandLine.appendSwitch('wm-name', 'YourAppName');
```

#### All Platforms: `DocumentPolicyIncludeJSCallStacksInCrashReports,EarlyEstablishGpuChannel,EstablishGpuChannelAsync`

Three switches applied everywhere:

1. **`DocumentPolicyIncludeJSCallStacksInCrashReports`** — "When the renderer crashes, include the JavaScript call stack (which JS functions were running) in the crash report." Without this, renderer crash reports show only C++ stack frames, which are useless for debugging JavaScript code.

2. **`EarlyEstablishGpuChannel` + `EstablishGpuChannelAsync`** — "Start setting up the GPU communication channel early and do it asynchronously." This is a performance optimization: by establishing the GPU channel earlier in the startup sequence, the first window can appear slightly faster.

**Universal reuse:** These three switches are beneficial for virtually all Electron apps. The call-stack switch is especially valuable for debugging renderer crashes.

---

## 3. Renderer Diagnostics

Cherry Studio uses `app.on('web-contents-created', ...)` as a global hook for every renderer and guest content instance.

Inside that hook it:
- Injects the `Document-Policy: include-js-call-stacks-in-crash-reports` response header
- Listens for `unresponsive`
- Calls `webContents.mainFrame.collectJavaScriptCallStack()` when a renderer freezes

This is a useful Electron pattern: attach process-wide monitoring once instead of repeating it on every window.

### Plain-Language Explanation

**"web-contents-created" event:** Every time Electron creates a new "web contents" (the backing object for any window, webview, or guest page), it fires this event. It is a global catch-all — instead of attaching diagnostics to each window individually, you listen to this event once and it covers every renderer process in your app.

**"webContents" (refresher):** A `webContents` is Electron's object that manages a web page. Every `BrowserWindow` has a `webContents` inside it. Every `<webview>` tag creates a `webContents`. It is the thing that loads URLs, handles navigation, and runs JavaScript.

**Why this pattern is useful:** If you create windows in 5 different places, and each needs crash monitoring, you would otherwise need to remember to add the monitoring code in all 5 places. The global hook guarantees every renderer gets monitored automatically — even ones created by future code you have not written yet.

**"Unresponsive" event:** Chromium detects when a renderer process has stopped responding (frozen). When this fires, the code calls `collectJavaScriptCallStack()` to snapshot what JavaScript was doing at the moment of the freeze. This is invaluable for debugging "the app froze and I don't know why" reports.

**Blind spot — "What is a response header?":** When a browser (or Electron renderer) loads a web page, the server sends back two things: the page content (HTML) and "headers" (metadata about the response). One such header is `Document-Policy`, which controls browser behavior for that page. Injecting `include-js-call-stacks-in-crash-reports` into every response header ensures that even if the page does not declare it, crash reports will include JavaScript stack information.

### Universal Reuse — Your Own Electron Project

This global-monitoring pattern should be in every Electron app:

```typescript
app.on('web-contents-created', (event, contents) => {
  // Monitor for freezes
  contents.on('unresponsive', async () => {
    const stack = await contents.mainFrame.collectJavaScriptCallStack();
    logger.error(`Renderer unresponsive. JS stack: ${stack}`);
  });

  // Inject crash-report-enhancing headers into every response
  contents.session.webRequest.onHeadersReceived((details, callback) => {
    callback({
      responseHeaders: {
        ...details.responseHeaders,
        'Document-Policy': 'include-js-call-stacks-in-crash-reports'
      }
    });
  });
});
```

---

## 4. Production Error Guarding

When not in dev mode, the main process subscribes to:
- `process.on('uncaughtException', ...)`
- `process.on('unhandledRejection', ...)`

This is not Electron-specific by itself, but in Electron main process it matters because an unhandled Node error can take down the entire desktop shell.

### Plain-Language Explanation

**"uncaughtException":** In JavaScript, when an error is thrown and nothing catches it (no try/catch), it becomes an "uncaught exception." By default, Node.js prints the error and exits the process. In an Electron main process, this means your app just disappears with no warning.

**"unhandledRejection":** The Promise equivalent of uncaught exceptions. When a Promise fails (rejects) and nothing handles that rejection (no `.catch()` or `try/catch` around an `await`), it becomes an "unhandled rejection."

**Why this matters especially in Electron:** In a regular Node.js server, an unhandled error crashing the process is bad but recoverable (a process manager restarts the server). In an Electron desktop app, the user sees "the app vanished." There is no process manager to restart it. So you MUST catch these errors and at minimum log them, and ideally show the user a graceful error message.

**Simple principle:** These handlers are the "safety net" at the bottom of your app. Errors that slip through every other layer of error handling land here. Without this net, those errors would kill the main process silently.

### Universal Reuse — Your Own Electron Project

Every Electron main process should have these two handlers in production:

```typescript
if (!isDev) {
  process.on('uncaughtException', (error) => {
    logger.error('Uncaught exception in main process', error);
    // Optionally: show a dialog to the user
    // dialog.showErrorBox('Unexpected Error', error.message);
  });

  process.on('unhandledRejection', (reason) => {
    logger.error('Unhandled rejection in main process', reason);
  });
}
```

---

## 5. Single-Instance Control

Cherry Studio calls `app.requestSingleInstanceLock()`.

If the lock fails:
- the new process quits immediately

If the lock succeeds:
- the app continues booting
- later, `app.on('second-instance', ...)` brings the main window forward and consumes any deep-link URL from the second launch

This is the standard Electron pattern when protocol URLs or notification launches must reopen an existing app instead of creating duplicate instances.

### Plain-Language Explanation

**What `requestSingleInstanceLock()` does:** It asks the operating system "is there already a copy of this app running?" If yes, it returns `false` (lock failed — someone else holds it). If no, it returns `true` (lock acquired — you are the only instance).

**How it works (simple principle):** Electron uses an OS-level mutex (mutual exclusion lock) — think of it as a flag in the operating system's memory that says "Cherry Studio is running." When the first instance starts, it creates this flag. When a second instance starts, it sees the flag already exists and knows to quit. When the first instance exits, the OS automatically removes the flag.

**Why not just let two instances run?** If the user clicks a `cherrystudio://` link in their browser, the OS launches the app to handle it. If a second instance launches independently, it cannot forward the link to the already-running instance. You would have two copies of the app, one of which received the link but has no windows, and one which has windows but never received the link.

**The `second-instance` event:** When instance #2 starts and finds the lock taken, Electron tells instance #1 "someone tried to launch you again, here are the arguments they passed." Instance #1 can then react — typically by focusing its main window and processing the command-line arguments (which may contain a deep-link URL).

**Blind spot — "What are command-line arguments (process.argv)?":** When any program is launched, the OS passes it an array of strings called "arguments." The first argument is always the path to the executable. Additional arguments come from whatever launched the program. When the OS launches an app to handle a `cherrystudio://` URL, that URL appears as an argument. `process.argv` is how Node.js code reads these arguments.

**Simple mental model:**
```
User clicks cherrystudio://settings in browser
  → OS: "Cherry Studio, open this: cherrystudio://settings"
  → Instance #2 starts, sees lock is taken, tells Instance #1
  → Instance #1: "Ah, someone wants settings" → focuses window → navigates to settings
  → Instance #2 quits
```

### Universal Reuse — Your Own Electron Project

This pattern is essential for any Electron app that:
- Registers a custom protocol (like `myapp://`)
- Can be launched from the browser (OAuth callbacks)
- Has notification-click actions
- Wants to prevent users from accidentally opening duplicate instances

```typescript
const gotTheLock = app.requestSingleInstanceLock();

if (!gotTheLock) {
  app.quit();  // Another instance is already running
} else {
  app.on('second-instance', (event, commandLine, workingDirectory) => {
    // Someone tried to launch a second copy.
    // Focus the main window and process their arguments.
    if (mainWindow) {
      if (mainWindow.isMinimized()) mainWindow.restore();
      mainWindow.focus();
    }
    // Extract deep-link URL from commandLine if present
    const deepLink = commandLine.find(arg => arg.startsWith('myapp://'));
    if (deepLink) handleDeepLink(deepLink);
  });
}
```

---

## 6. `app.whenReady()` Boot Order

After Electron is ready, Cherry Studio performs desktop-shell startup in a deliberate order:

1. Record current app version
2. Initialize webview hotkeys
3. Set Windows AppUserModelID with `electronApp.setAppUserModelId(...)`
4. Hide the macOS dock if launch-to-tray is enabled
5. Finish any backup restore flow before creating windows
6. Create the main window through `windowService.createMainWindow()`
7. Create the tray service
8. Setup the macOS application menu
9. Initialize tracing, power monitoring, and analytics
10. Extract bundled runtime binaries if needed
11. Register global shortcuts
12. Register IPC handlers
13. Start LAN discovery
14. Patch devtools font on Windows dev builds
15. Setup Linux AppImage deep-link support
16. Install React/Redux devtools in development
17. Initialize the selection assistant
18. Start longer-running async services such as agents, API server, schedulers, and channels

The main lesson is that Electron startup order is not cosmetic. Window creation, protocol registration, IPC availability, and long-running services depend on each other.

### Why Each Step Has Its Place (Dependency Rationale)

**Steps 1-2 (version + webview hotkeys):** These have no dependencies — they can run immediately when Electron is ready. Version recording is informational; webview hotkeys need to be initialized before any webview content loads (which happens when windows are created).

**Step 3 (AppUserModelID):** On Windows, the AppUserModelID must be set before creating any windows. It tells Windows "this window belongs to this app" — without it, the taskbar icon grouping and notifications break.

**Blind spot — "What is AppUserModelID?":** On Windows 7+, every application has a unique string ID called the "Application User Model ID." This ID controls how Windows groups taskbar buttons (windows with the same ID stack together) and which app icon is shown. If you don't set it, Windows uses the executable path, which can cause your app to inherit a wrong icon or not group correctly with its own windows.

**Steps 4-5 (dock + backup restore):** These must happen before window creation because:
- If launch-to-tray is enabled, the dock icon should be hidden BEFORE the first window appears (otherwise it flashes briefly on screen)
- Backup restoration might affect app state that windows depend on

**Step 6 (main window):** The window is created here, before tray/menu, because the tray and menu often need a reference to the main window to control it.

**Step 7 (tray service):** After the main window exists (so the tray can show/hide it), but before IPC handlers are registered (the tray might need IPC).

**Step 8 (macOS menu):** After the window exists (menu actions navigate the window), but the menu is a native OS component, not dependent on IPC.

**Steps 9-18 (services):** These are ordered by dependency:
- Power monitoring and analytics have no dependencies → start early
- Runtime binary extraction must happen before any code that uses those binaries
- Global shortcuts must be registered before IPC handlers that let the renderer configure them
- IPC handlers must be registered before the renderer loads (or the renderer's IPC calls will fail)
- LAN discovery has no dependencies
- Devtools are development-only, so they go late in the boot order
- Long-running async services (agents, API server, schedulers) are started last because they are the most resource-intensive and depend on everything else being ready

**Simple principle:** When ordering startup steps, ask "does B need A to exist first?" If yes, A must come before B. If there is no dependency, place lighter/simpler steps earlier and heavy/resource-intensive steps later.

### Universal Reuse — Your Own Electron Project

The specific 18 steps are Cherry Studio's needs, but the categories are universal:

1. **Configuration that affects window appearance** (AppUserModelID, dock behavior)
2. **Window creation** (main window first, secondary windows after)
3. **Native OS integration** (tray, menu, shortcuts)
4. **IPC registration** (must happen before renderer loads)
5. **Background services** (resource-intensive, start last)

When adding a new startup step to your own Electron app, ask: "Does this depend on windows existing? Does this need IPC? Does this consume significant resources?" Place it accordingly.

---

## 7. App Activation And Reopen

Cherry Studio uses `app.on('activate', ...)` to match normal macOS desktop behavior:
- if no main window exists, recreate it
- otherwise, show the existing main window

That keeps dock clicks and app switching consistent with native macOS expectations.

### Plain-Language Explanation

**The `activate` event:** On macOS, this fires when the user clicks the app's dock icon while the app is already running. It is macOS-specific behavior — Windows and Linux apps typically do not need an activate handler because their windows are always visible or accessible from the taskbar.

**Why it matters:** On macOS, it is normal to close all an app's windows but keep the app running (the app remains in the dock with a dot underneath). When the user clicks the dock icon, they expect a new window to appear. Without handling `activate`, clicking the dock icon would do nothing — the app is running but has no windows to show.

**The "recreate if no windows" pattern:** This is the standard macOS behavior. If the user closed all windows (but did not quit the app), clicking the dock icon should create a fresh main window. If a window already exists (perhaps minimized or behind other windows), clicking the dock icon should bring it forward.

**Simple mental model:**
```
macOS user closes last window → app stays running (dot under dock icon)
User clicks dock icon → activate event fires
  → "Is there a window?" → No → Create one
  → "Is there a window?" → Yes → Show it
```

### Universal Reuse — Your Own Electron Project

Every Electron app targeting macOS should handle the `activate` event:

```typescript
app.on('activate', () => {
  if (BrowserWindow.getAllWindows().length === 0) {
    createMainWindow();
  } else {
    const mainWindow = BrowserWindow.getAllWindows()[0];
    mainWindow.show();
    mainWindow.focus();
  }
});
```

On Windows and Linux, this event typically does not fire because those platforms have different app lifecycle conventions (closing the last window usually quits the app).

---

## 8. Protocol Registration And Secondary Launch Routing

Outside `whenReady()`, `registerProtocolClient(app)` is called immediately, then three entry points can feed URLs back into the running app:

- `app.on('open-url', ...)` for macOS
- startup `process.argv`
- `app.on('second-instance', ...)` for Windows/Linux relaunches

This is why single-instance locking and deep-link handling are tightly connected.

### Plain-Language Explanation

**The three entry points — why three?**

Different operating systems deliver protocol URLs to your app through different mechanisms:

1. **macOS — `open-url` event:** When an app is already running, macOS uses a special event called `open-url` to deliver protocol URLs to the running instance. This is elegant: the OS just tells your app "here, handle this."

2. **Windows/Linux — `process.argv`:** On these platforms, protocol URLs are delivered as command-line arguments to a new process. The OS literally launches a second copy of your app with the URL as an argument. If your app has single-instance locking, this second copy quits immediately after forwarding the URL to the first instance.

3. **All platforms — `second-instance` event:** When single-instance locking prevents a second copy from running, the first instance receives the `second-instance` event with the command-line arguments that were passed to the would-be second instance. This is how Windows/Linux deep links ultimately reach the running app: OS → new process → single-instance lock blocks it → `second-instance` event fires on existing instance → URL is extracted from arguments.

**Blind spot — "Why doesn't macOS use `second-instance` for deep links?":** macOS handles protocol URLs differently at the OS level. When you register a custom protocol on macOS, the OS delivers URLs to your running app via the `open-url` event. It does not launch a second process. Windows and Linux, by contrast, launch a new process and pass the URL as a command-line argument. This OS-level difference is the reason your Electron app must handle all three entry points for cross-platform deep-link support.

**Why protocol registration is called outside `whenReady()`:** `setAsDefaultProtocolClient()` tells the OS "when a `cherrystudio://` link is clicked, launch this app." This registration should happen as early as possible — even before Electron is fully ready — so that if a protocol URL arrives during startup, the app is prepared to handle it.

### Universal Reuse — Your Own Electron Project

This three-entry-point pattern is mandatory for any Electron app with a custom protocol. Here is the minimal cross-platform recipe:

```typescript
// 1. Register early (before whenReady)
app.setAsDefaultProtocolClient('myapp');

// 2. Handle all three entry points
let deepLinkUrl: string | null = null;

// macOS: running app receives open-url
app.on('open-url', (event, url) => {
  event.preventDefault();
  deepLinkUrl = url;
  handleDeepLink(url);
});

// Windows/Linux: second instance forwards the URL
app.on('second-instance', (event, commandLine) => {
  const url = commandLine.find(arg => arg.startsWith('myapp://'));
  if (url) handleDeepLink(url);
});

// All platforms: app launched with URL as argument
// Check process.argv after whenReady
app.whenReady().then(() => {
  const url = process.argv.find(arg => arg.startsWith('myapp://'));
  if (url) handleDeepLink(url);
});
```

---

## 9. Quit Lifecycle

Cherry Studio uses two different quit phases:

### `before-quit`

Used to mark `app.isQuitting = true` and dispose lightweight interactive services:
- selection service
- LAN transfer client
- local transfer discovery

### `will-quit`

Used for broader async cleanup:
- stop OVMS if supported
- stop schedulers
- stop channel adapters
- destroy analytics service
- stop OpenClaw gateway
- cleanup MCP service
- stop API server
- finish logger output

The Electron knowledge point here is that shutdown is often split into a synchronous "transition to quitting" phase and a later resource cleanup phase.

### Plain-Language Explanation

**Why two phases instead of one?**

Electron's quit process has a specific sequence:

```
User clicks Quit (or Cmd+Q / Alt+F4)
  ↓
All windows emit 'close' event
  ↓
If all windows allow closing, 'before-quit' fires
  ↓ (transition: app.isQuitting = true)
Windows are destroyed
  ↓
'will-quit' fires
  ↓ (final cleanup: stop services, close connections)
App process exits
```

**The `before-quit` phase** is for the "decision" part of quitting:
- Set `app.isQuitting = true` — This flag is checked elsewhere in the code. For example, the close-to-tray logic checks this flag: if the user closed the window (not quitting), hide to tray; if the app is quitting, allow the close.
- Stop lightweight services that affect window behavior — These should stop promptly so they don't interfere with window destruction.

**The `will-quit` phase** is for the "cleanup" part:
- By this point, all windows are already destroyed. No user interaction is possible.
- This is where you stop long-running background services (API server, schedulers, agents).
- The order of cleanup matters: stop services that depend on each other in reverse dependency order. For example, the API server might use the database, so stop the API server before closing the database connection.

**Blind spot — "What happens if `before-quit` is not handled?":** Without setting `app.isQuitting = true` in `before-quit`, the close-to-tray logic cannot distinguish between "user clicked the X button" (should hide to tray) and "app is actually quitting" (should close the window for real). This causes the classic bug where Cmd+Q or "Quit" from the menu hides the app to tray instead of quitting.

**Blind spot — "What is `app.isQuitting`?":** This is not a built-in Electron property — it is a custom flag that the app sets on the `app` object. It is a simple boolean variable that other parts of the code read to determine whether the app is in the process of quitting. This is necessary because Electron does not provide a built-in way to check "am I currently quitting?" during the window close event.

### Universal Reuse — Your Own Electron Project

Every Electron app should structure its quit logic this way:

```typescript
let isQuitting = false;

app.on('before-quit', () => {
  isQuitting = true;
  // Stop lightweight services immediately
  // (anything that affects window behavior)
});

app.on('will-quit', async (event) => {
  // Stop heavyweight background services
  // (API server, database connections, scheduled tasks)
  // Order: stop dependents before dependencies
  await apiServer.stop();
  await database.close();
  await logger.flush();
});

// In window close handler:
mainWindow.on('close', (event) => {
  if (!isQuitting && trayEnabled) {
    event.preventDefault();  // Don't close, just hide
    mainWindow.hide();
  }
  // If isQuitting is true, allow normal close
});
```

---

## 10. Related Files

- `src/main/index.ts`
- `src/main/services/PowerMonitorService.ts`
- `src/main/services/WindowService.ts`
- `src/main/services/ProtocolClient.ts`
- `src/main/services/AppUpdater.ts`

---

## 11. Practical Takeaways

- Add Chromium switches before creating windows
- Treat `web-contents-created` as a global instrumentation hook
- Use `requestSingleInstanceLock()` if deep links or OS relaunches matter
- Keep Electron UI startup in `app.whenReady()`
- Split quit logic across `before-quit` and `will-quit`

---

## 12. How To Apply This Knowledge In Development

Use this document as a decision guide when you touch `src/main/index.ts` or add a new main-process service.

Choose the right place for new code:
- Put process-wide switches, protocol registration, and single-instance guards before `app.whenReady()`.
- Put `BrowserWindow`, tray, menu, and Electron UI construction inside `app.whenReady()` or lifecycle services started after readiness.
- Put long-lived resources into lifecycle services instead of growing `index.ts`.
- Put final cleanup into quit handlers only if it truly belongs to app shutdown rather than service stop logic.

Practical usage pattern:
1. Decide whether the behavior is pre-ready, ready-time, or shutdown-time.
2. Decide whether it is shell bootstrap code or a reusable service responsibility.
3. Verify whether it depends on an existing window, tray, protocol client, or IPC channel.
4. Place it in the earliest safe phase, not the latest convenient phase.

Common mistakes this avoids:
- Registering protocol or Chromium switches too late.
- Creating windows before required configuration is applied.
- Starting background services before their dependencies exist.
- Leaving quit cleanup split across unrelated files with no clear ownership.

---

## 13. Typical Application Scenarios

- You add a new startup-only Chromium flag for a rendering bug. This belongs before `app.whenReady()`.
- You add a new service that listens for system power events. This belongs in a lifecycle service started after readiness, not inline in `index.ts`.
- You add a new deep-link action. This requires both single-instance flow and protocol routing to stay consistent during startup and relaunch.
- You add a new resource that must be disposed on exit. First prefer lifecycle-managed cleanup; only use `before-quit` or `will-quit` for truly app-level shutdown coordination.

---

## 14. Relationship To The Other Electron Records

- This document is the foundation for `electron_window_tray_menu_architecture.md` because windows, tray, and menus are created during the ready phase.
- It directly supports `electron_protocol_oauth_and_deep_linking.md` because protocol registration, startup args, and `second-instance` routing are lifecycle concerns first.
- It indirectly supports `electron_webview_session_management.md` because webview session setup and guest hooks depend on the shell already being booted correctly.
- It affects `electron_build_packaging_and_update_pipeline.md` because packaged protocol handlers, startup arguments, and updater behavior must align with the runtime lifecycle.
