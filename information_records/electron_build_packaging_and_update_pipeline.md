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
