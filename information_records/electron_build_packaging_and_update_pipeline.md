# Electron Build, Packaging, And Update Pipeline In Cherry Studio

Cherry Studio is not a single-window Electron shell. Its build and packaging setup is designed around multiple renderer entry points, preload bridging, platform packaging, and remote update channels.

Primary files:
- `electron.vite.config.ts`
- `electron-builder.yml`
- `src/main/services/AppUpdater.ts`

## 1. Multi-Target Build With `electron-vite`

The project defines three build targets:
- `main`
- `preload`
- `renderer`

This matters because Electron apps usually ship different code under different trust models:
- main process has Node + Electron authority
- preload is the controlled bridge
- renderer is the UI app

## 2. Multi-Entry Renderer Windows

The renderer build has multiple HTML entry points:

- `index.html`
- `miniWindow.html`
- `selectionToolbar.html`
- `selectionAction.html`
- `traceWindow.html`

This is how Cherry Studio supports multiple native windows with separate React entry files while still shipping one desktop application.

Electron knowledge point:
- extra `BrowserWindow` instances often deserve their own renderer bundle instead of conditional routing inside one SPA entry

## 3. Main-Process Bundle Strategy

The main build:
- marks `electron` and many dependencies as external
- disables manual chunks
- enables `inlineDynamicImports`

That reduces main-process bundling complexity and makes the Electron entry easier to package as a single runtime-oriented bundle.

## 4. Packaging Rules In `electron-builder.yml`

`electron-builder.yml` defines:
- `appId`
- `productName`
- protocol registration for `cherrystudio`
- target formats for Windows, macOS, and Linux
- aggressive file exclusion rules
- `asarUnpack` exceptions for runtime assets and native dependencies
- hook scripts such as `beforePack`, `afterPack`, `afterSign`, and `artifactBuildCompleted`

Important project-specific details:
- Windows builds include both NSIS installer and portable target
- Linux builds include AppImage, deb, and rpm
- `resources/**/*` is explicitly included and unpacked
- mini runtime binaries and some native modules are kept outside packed asar where necessary

## 5. Why `asarUnpack` Matters Here

Cherry Studio uses native modules, runtime resources, and external binaries that may need real filesystem access at runtime. Packing everything into `app.asar` would break some of those use cases.

So the packaging config selectively unpacks:
- `resources/**`
- runtime binaries
- some native-library artifacts

That is a standard Electron packaging tradeoff: tighter packaging versus runtime accessibility.

## 6. Auto-Update Flow

`AppUpdater.ts` builds a more advanced updater than a simple `checkForUpdates()`.

It configures `electron-updater` with:
- app-specific request headers
- environment-aware dev behavior
- auto-download controlled by config
- `autoInstallOnAppQuit = false`

The last setting is important. Cherry Studio intentionally requires an explicit user install action instead of silently installing on quit.

## 7. Feed URL, Mirror, And Channel Selection

Before checking for updates, Cherry Studio:
- reads the current app version
- determines whether test-plan mode is enabled
- derives the requested channel (`latest`, `rc`, or `beta`)
- checks the user's IP country
- chooses GitCode for China and GitHub-based feeds otherwise
- downloads a remote update-config JSON
- finds the newest compatible channel allowed by `minCompatibleVersion`

If remote config fails, it falls back to a default production feed URL.

This is more sophisticated than a fixed Electron update URL and is useful when a desktop app needs region-aware distribution and phased channels.

## 8. Renderer Notification Of Update State

Updater events are pushed back into the UI with IPC:
- `UpdateError`
- `UpdateAvailable`
- `UpdateNotAvailable`
- `DownloadProgress`
- `UpdateDownloaded`

This keeps update policy in main process and update presentation in renderer.

## 9. Localized Release Notes

Cherry Studio stores multilingual release notes in one string using markers like:

```text
<!--LANG:en-->
...
<!--LANG:zh-CN-->
...
<!--LANG:END-->
```

`AppUpdater.processReleaseInfo()` extracts the correct language block according to app language before the UI shows it.

That is not an Electron API feature, but it is a project-specific desktop update pattern worth knowing.

## 10. Practical Takeaways

- Split Electron builds by trust boundary: main, preload, renderer
- Use separate renderer entries for separate native windows
- Keep runtime-required assets out of packed asar when necessary
- Let main process own updater policy
- Choose update feeds dynamically when region, compatibility, and release channel matter

## 11. How To Apply This Knowledge In Development

Use this document when a feature crosses the boundary from source code into a distributable desktop product.

Choose the right integration point:
- Change `electron.vite.config.ts` when you add or reshape main, preload, or renderer entry points.
- Change `electron-builder.yml` when the packaged app needs new assets, binaries, protocol metadata, or platform targets.
- Change `AppUpdater.ts` when update policy, feed selection, or renderer notifications change.
- Verify runtime file access assumptions before deciding whether something can live inside `app.asar`.

Practical usage pattern:
1. Identify which process owns the new code: main, preload, renderer, or multiple.
2. If a new native window exists, add a distinct renderer entry rather than overloading an unrelated one.
3. If the feature needs runtime files, native modules, or child processes, confirm packaging and `asarUnpack` rules.
4. Test both development execution and packaged behavior because Electron packaging changes path, protocol, and updater assumptions.

Common mistakes this avoids:
- Adding a new window in code without adding its HTML entry to the build.
- Packaging a runtime dependency into `app.asar` when it needs filesystem access.
- Assuming update behavior is a static URL rather than channel and region aware.
- Fixing only dev mode while leaving the packaged app broken.

## 12. Typical Application Scenarios

- Add a new renderer entry for a utility window, onboarding flow, or diagnostics surface.
- Ship an extra executable, model file, or runtime asset needed by the main process.
- Change Windows or Linux packaging targets.
- Add a new beta or regional update rule and surface the result in renderer.

## 13. Relationship To The Other Electron Records

- This document packages the runtime behaviors described in `electron_main_process_lifecycle.md`; startup assumptions only matter if the packaged app preserves them.
- It packages the window entry points described in `electron_window_tray_menu_architecture.md`.
- It must include protocol metadata required by `electron_protocol_oauth_and_deep_linking.md`.
- It can preserve or break the preload and asset assumptions used by `electron_webview_session_management.md`.
