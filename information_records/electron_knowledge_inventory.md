# Electron Knowledge Inventory Missing From Existing `information_records`

This file enumerates Electron-development knowledge that is implemented in Cherry Studio but was not already covered as a dedicated tutorial in the existing `information_records` set. The older records already explain the preload API surface, selection assistant internals, and some feature behavior. The missing part was the Electron shell implementation behind those features.

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

Developer usage:
- Read this before moving code into or out of `src/main/index.ts`.
- Use it when deciding whether new logic belongs before `app.whenReady()`, inside lifecycle services, or during quit cleanup.
- Use it to avoid subtle startup bugs such as late protocol registration, duplicated app instances, or windows created before required switches are applied.

Application scenarios:
- Add a new global Chromium switch.
- Add startup diagnostics or crash monitoring.
- Register a new main-process service that must start after Electron is ready.
- Add cleanup for a long-lived resource on quit.

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

Developer usage:
- Read this before creating a new window type or changing main-window behavior.
- Use it when deciding whether a desktop action should be a tray interaction, native menu action, or renderer-only action.
- Use it to keep platform-specific window chrome and focus behavior inside the window layer instead of scattering it.

Application scenarios:
- Add a settings window or another utility window.
- Change launch-to-tray or close-to-tray rules.
- Add a tray action that must toggle renderer state through IPC.
- Add a macOS-only menu item that navigates the SPA.

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

Developer usage:
- Read this before introducing any OS-to-app callback path.
- Use it when a browser, installer, CLI tool, or another app must reopen Cherry Studio with structured data.
- Use it to decide what should be parsed in main process versus what can safely be forwarded to renderer.

Application scenarios:
- Add a new OAuth provider callback.
- Add `cherrystudio://feature/...` links for a new workflow.
- Support protocol-based installation or import actions from the website.
- Debug why a deep link works on macOS but not from a second launch on Windows.

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

Developer usage:
- Read this before enabling new embedded content or relaxing guest permissions.
- Use it when working on user-agent shaping, cookie/session persistence, popup policy, or guest keyboard behavior.
- Use it to keep privileged Electron APIs in main process even when the renderer owns the `<webview>` element.

Application scenarios:
- Embed a new third-party mini app.
- Export a guest page to PDF or HTML.
- Let some guest links stay in-app while pushing others to the system browser.
- Forward host shortcuts into or out of guest content.

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

Developer usage:
- Read this before adding a new renderer entry, packaging asset, installer target, or updater rule.
- Use it when deciding whether a file belongs in `asar`, `asarUnpack`, or an external resource path.
- Use it to understand why a feature works in dev but fails after packaging.

Application scenarios:
- Add a new standalone window HTML entry.
- Ship a native binary or runtime asset.
- Change update channels or mirror-selection logic.
- Add a new installer hook or platform packaging target.

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
