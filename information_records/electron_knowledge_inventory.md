# Electron Knowledge Inventory Missing From Existing `information_records`

This file enumerates Electron-development knowledge that is implemented in Cherry Studio but was not already covered as a dedicated tutorial in the existing `information_records` set. The older records already explain the preload API surface, selection assistant internals, and some feature behavior. The missing part was the Electron shell implementation behind those features.

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

Tutorial:
- `information_records/electron_main_process_lifecycle.md`

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

Tutorial:
- `information_records/electron_window_tray_menu_architecture.md`

## Group 3. Custom Protocol, OAuth Callback Routing, And Deep Links

Completed function:
- Receive `cherrystudio://` links for OAuth callbacks and MCP/provider actions, including packaged Linux AppImage deep-link registration.

New knowledge points:
- How `setAsDefaultProtocolClient()` is registered differently in dev and packaged modes
- How protocol URLs are consumed from `open-url`, startup args, and `second-instance`
- How main-process routing splits `mcp`, `providers`, and generic callback payloads
- How the renderer receives deep-link payloads through IPC instead of parsing OS launch arguments directly
- How AppImage builds create a `.desktop` handler and refresh the desktop database

Tutorial:
- `information_records/electron_protocol_oauth_and_deep_linking.md`

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

Tutorial:
- `information_records/electron_webview_session_management.md`

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

Tutorial:
- `information_records/electron_build_packaging_and_update_pipeline.md`

## Coverage Notes

Existing records that already covered adjacent Electron topics:
- `information_records/window-api-reference.md`
- `information_records/selection_assistant_implementation.md`
- `information_records/login_methods_explanation.md`
- `information_records/package-scripts-explanation.md`

Those files remain useful, but they do not explain the main-process and packaging patterns above in enough project-specific detail.
