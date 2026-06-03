# Electron Main Process Lifecycle In Cherry Studio

This tutorial explains how Cherry Studio boots, guards, monitors, and shuts down its Electron main process. The core implementation lives in `src/main/index.ts`.

## 1. Why This File Runs Early

`src/main/index.ts` imports `./bootstrap` before the rest of the app. The comment at the top is important: some initialization must happen before Electron is fully ready, especially anything that affects app data paths or early environment setup.

The main rule is:
- Do OS-level or process-level setup before `app.whenReady()`
- Do `BrowserWindow`, tray, and most Electron UI work inside `app.whenReady()`

## 2. Pre-Ready Runtime Setup

Before `app.whenReady()`, the project configures several Electron and process behaviors:

- `crashReporter.start(...)` enables local crash reports
- `app.disableHardwareAcceleration()` is applied if the config says so
- Windows gets `wm-window-animations-disabled` to avoid transparent-window flashing
- Linux Wayland gets `enable-features=GlobalShortcutsPortal` so global shortcuts work better
- Linux also gets WM class/name switches so the window manager identifies the app correctly
- All platforms get `DocumentPolicyIncludeJSCallStacksInCrashReports,EarlyEstablishGpuChannel,EstablishGpuChannelAsync`

The interesting Electron knowledge point is that many app-wide Chromium switches only work if added before windows are created.

## 3. Renderer Diagnostics

Cherry Studio uses `app.on('web-contents-created', ...)` as a global hook for every renderer and guest content instance.

Inside that hook it:
- Injects the `Document-Policy: include-js-call-stacks-in-crash-reports` response header
- Listens for `unresponsive`
- Calls `webContents.mainFrame.collectJavaScriptCallStack()` when a renderer freezes

This is a useful Electron pattern: attach process-wide monitoring once instead of repeating it on every window.

## 4. Production Error Guarding

When not in dev mode, the main process subscribes to:
- `process.on('uncaughtException', ...)`
- `process.on('unhandledRejection', ...)`

This is not Electron-specific by itself, but in Electron main process it matters because an unhandled Node error can take down the entire desktop shell.

## 5. Single-Instance Control

Cherry Studio calls `app.requestSingleInstanceLock()`.

If the lock fails:
- the new process quits immediately

If the lock succeeds:
- the app continues booting
- later, `app.on('second-instance', ...)` brings the main window forward and consumes any deep-link URL from the second launch

This is the standard Electron pattern when protocol URLs or notification launches must reopen an existing app instead of creating duplicate instances.

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

## 7. App Activation And Reopen

Cherry Studio uses `app.on('activate', ...)` to match normal macOS desktop behavior:
- if no main window exists, recreate it
- otherwise, show the existing main window

That keeps dock clicks and app switching consistent with native macOS expectations.

## 8. Protocol Registration And Secondary Launch Routing

Outside `whenReady()`, `registerProtocolClient(app)` is called immediately, then three entry points can feed URLs back into the running app:

- `app.on('open-url', ...)` for macOS
- startup `process.argv`
- `app.on('second-instance', ...)` for Windows/Linux relaunches

This is why single-instance locking and deep-link handling are tightly connected.

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

## 10. Related Files

- `src/main/index.ts`
- `src/main/services/PowerMonitorService.ts`
- `src/main/services/WindowService.ts`
- `src/main/services/ProtocolClient.ts`
- `src/main/services/AppUpdater.ts`

## 11. Practical Takeaways

- Add Chromium switches before creating windows
- Treat `web-contents-created` as a global instrumentation hook
- Use `requestSingleInstanceLock()` if deep links or OS relaunches matter
- Keep Electron UI startup in `app.whenReady()`
- Split quit logic across `before-quit` and `will-quit`

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

## 13. Typical Application Scenarios

- You add a new startup-only Chromium flag for a rendering bug. This belongs before `app.whenReady()`.
- You add a new service that listens for system power events. This belongs in a lifecycle service started after readiness, not inline in `index.ts`.
- You add a new deep-link action. This requires both single-instance flow and protocol routing to stay consistent during startup and relaunch.
- You add a new resource that must be disposed on exit. First prefer lifecycle-managed cleanup; only use `before-quit` or `will-quit` for truly app-level shutdown coordination.

## 14. Relationship To The Other Electron Records

- This document is the foundation for `electron_window_tray_menu_architecture.md` because windows, tray, and menus are created during the ready phase.
- It directly supports `electron_protocol_oauth_and_deep_linking.md` because protocol registration, startup args, and `second-instance` routing are lifecycle concerns first.
- It indirectly supports `electron_webview_session_management.md` because webview session setup and guest hooks depend on the shell already being booted correctly.
- It affects `electron_build_packaging_and_update_pipeline.md` because packaged protocol handlers, startup arguments, and updater behavior must align with the runtime lifecycle.
