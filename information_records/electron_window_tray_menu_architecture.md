# Electron Window, Tray, And Menu Architecture In Cherry Studio

This tutorial explains how Cherry Studio implements its native desktop shell around `BrowserWindow`, tray integration, and the macOS application menu.

Primary files:
- `src/main/services/WindowService.ts`
- `src/main/services/TrayService.ts`
- `src/main/services/AppMenuService.ts`
- `src/main/utils/windowUtil.ts`

## 1. Why `WindowService` Exists

Cherry Studio does not create windows inline in `main/index.ts`. Instead, `WindowService` owns:
- main window creation
- mini window creation
- window state persistence
- platform-specific title bar rules
- close/minimize/show logic
- guest content handlers

That is a common Electron scaling pattern: keep the main entry focused on lifecycle, and put `BrowserWindow` policy into a service.

## 2. Main Window Construction

The main window is created with `electron-window-state`, so size and position survive restarts.

Important project-specific settings:
- `show: false` and delayed display on `ready-to-show`
- `autoHideMenuBar: true`
- `vibrancy: 'sidebar'`
- `visualEffectState: 'active'`
- `webviewTag: true`
- `backgroundThrottling: false`
- custom zoom factor from config

Platform differences:
- macOS uses `titleBarStyle: 'hidden'`, `titleBarOverlay`, and traffic light positioning
- Linux may use a system title bar when config enables it
- Windows 11 may use Mica through `backgroundMaterial`

The key Electron lesson is that a cross-platform window service usually chooses different window chrome strategies per OS instead of forcing one configuration everywhere.

## 3. Window State And Geometry

Two separate persisted states are used:
- default main window state
- `miniWindow-state.json` for the quick assistant

This lets each window type remember its own geometry independently.

## 4. Main Window Event Strategy

Cherry Studio attaches several event groups to the main window:

- `ready-to-show`: decides whether launch-to-tray should suppress the initial show
- `enter-full-screen` / `leave-full-screen`: pushes fullscreen state to the renderer
- `will-resize`, `resize`, `maximize`, `unmaximize`, `restore`: resends window size and reapplies zoom factor
- `did-navigate-in-page`: fixes Electron zoom reset during in-page navigation
- `render-process-gone`: reload once, or exit if crashes repeat too quickly

This is a good example of Electron code compensating for platform bugs and renderer quirks rather than only responding to ideal events.

## 5. Close-To-Tray Behavior

Cherry Studio intercepts the main window `close` event.

Decision path:
- if the app is already quitting, allow normal quit
- if tray is disabled, quit directly on Windows/Linux
- if tray is enabled and tray-on-close is enabled, prevent close and hide the window
- on macOS, also hide the dock when closing to tray

The important Electron concept is that "close window" and "quit app" are separate behaviors, and desktop apps often redefine that boundary.

## 6. Mini Window Architecture

The quick assistant mini window is a second `BrowserWindow` with desktop-utility behavior:
- frameless
- always on top
- skip taskbar
- hidden until needed
- optionally preloaded during main-window creation

Cherry Studio also handles:
- moving the mini window to the cursor's display
- restoring focus behavior differently on Windows and macOS
- pinning vs auto-hide-on-blur
- fullscreen-safe visibility on macOS

This is more advanced than a normal popup because it behaves like a desktop utility window rather than a page route.

## 7. Tray Service

`TrayService` turns config state into native tray behavior.

It handles:
- icon selection per OS and theme
- tray destruction/recreation when tray config changes
- localized context menus
- click behavior that can open either the main window or mini window
- live toggling of selection assistant state from the tray

Platform differences:
- macOS uses resized template images
- Linux explicitly sets the context menu on the tray
- Windows resets the tray image directly

## 8. macOS Application Menu

`AppMenuService` only exists on macOS.

It builds a localized native menu with:
- app menu roles like hide/unhide/services/quit
- standard file/edit/view/window menus
- help links opened through `shell.openExternal`
- an About action that sends `IpcChannel.Windows_NavigateToAbout` into the renderer

This shows an important Electron pattern: native menu actions can drive SPA navigation by emitting IPC into the existing renderer.

## 9. Guest Content And External Navigation Policy

`WindowService` also owns several guest-content rules:
- `will-navigate` blocks unexpected navigation and opens safe URLs externally
- `setWindowOpenHandler()` allows a small OAuth popup allowlist
- `http://file/...` URLs are resolved safely and checked against path traversal
- CSP and X-Frame-Options headers are stripped from responses for embedded content

That mixes shell UX with embedded-content policy, which is common in Electron apps that host external sites or mini apps.

## 10. Practical Takeaways

- Centralize `BrowserWindow` policy in a service
- Treat tray, window close, and quit as distinct UX states
- Persist geometry per window type
- Expect platform-specific hacks for focus, fullscreen, and title bars
- Use native menus and tray items as IPC entry points back into the renderer
