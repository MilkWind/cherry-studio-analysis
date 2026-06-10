# Electron Build, Packaging, And Update Pipeline In Cherry Studio

Cherry Studio is not a single-window Electron shell. Its build and packaging setup is designed around multiple renderer entry points, preload bridging, platform packaging, and remote update channels.

Primary files:
- `electron.vite.config.ts`
- `electron-builder.yml`
- `src/main/services/AppUpdater.ts`

---

## Beginner's Prerequisites: Concepts You Need Before Reading

### The Big Picture: Source Code → Desktop App

You write TypeScript, JSX, CSS, and import npm packages. Users download a `.exe`, `.dmg`, or `.AppImage` and run it. The gap between "source code on my machine" and "installer on the user's machine" is what this document covers. Here is the pipeline:

```
Source Code (TypeScript/JSX/CSS)
  ↓  [Build] electron-vite compiles everything into plain JS/HTML/CSS
Built Files (bundled JavaScript + HTML + assets)
  ↓  [Package] electron-builder wraps everything into platform installers
Installers (.exe / .dmg / .AppImage)
  ↓  [Distribute] Users download and install
Installed App
  ↓  [Update] electron-updater checks for new versions and applies them
Updated App
```

### What Is a "Build"? (For Absolute Beginners)

Your source code is full of things that browsers and Node.js cannot run directly:
- **TypeScript** → browsers only understand JavaScript. TypeScript must be converted ("transpiled") to JavaScript.
- **JSX** (`<div>...</div>` in JavaScript) → browsers do not understand JSX. It must be converted to `React.createElement()` calls.
- **Import statements** (`import { thing } from './module'`) → browsers need all the code in one file (or a few files), not scattered across hundreds of files with imports.
- **CSS modules** (`import styles from './app.module.css'`) → browsers do not understand CSS modules. They must be converted to plain CSS.

**The build tool** (electron-vite, in this project) does all these conversions. It reads your source files, transforms them, and outputs plain JavaScript/HTML/CSS files that can actually run.

**Simple mental model:** A build tool is like a translator. You write in "developer language" (TypeScript + JSX + imports), and the build tool translates it into "computer language" (plain JavaScript + HTML + CSS) that browsers and Node.js understand natively.

### What Is "Packaging"? (For Absolute Beginners)

After building, you have a folder of plain JavaScript/HTML/CSS files. But users do not want a folder of files — they want an installer they can double-click. Packaging is the process of wrapping those files into a platform-specific distributable format:

| Platform | Packaging Format | What It Is |
|----------|-----------------|------------|
| Windows | `.exe` installer (NSIS) | A setup wizard like "Next → Next → Install" |
| Windows | `.exe` portable | A standalone executable, no installation needed |
| macOS | `.dmg` disk image | A virtual disk that mounts when double-clicked, drag the app to Applications |
| macOS | `.app` bundle | The actual application bundle (inside the `.dmg`) |
| Linux | `.AppImage` | A single portable executable file |
| Linux | `.deb` | Debian/Ubuntu package (installed via `dpkg -i`) |
| Linux | `.rpm` | Red Hat/Fedora package (installed via `rpm -i`) |

**The tool that does this:** `electron-builder` — the standard packaging tool for Electron apps. It reads a configuration file (`electron-builder.yml`) that describes what platforms to target, what files to include, what icons to use, and what protocols to register.

### What Is "asar"? (Electron's Archive Format)

**asar** stands for "Atom Shell Archive" (Electron was originally called "Atom Shell"). It is a simple archive format — like a `.zip` file but optimized for Electron:

- All your app's JavaScript, HTML, and CSS files are packed into a single file called `app.asar`
- Electron can read files from inside the asar as if they were regular files on disk
- This makes distribution cleaner (one file instead of thousands) and slightly faster

**Why not everything goes in the asar:** Some things need to be real files on the filesystem because:
- **Native Node.js modules** (`.node` files) → these are compiled binary code. The OS loader needs them as real files.
- **External executables** → if your app spawns a child process, the executable must be a real file.
- **Runtime resources** → if your code uses `fs.readFileSync()` with a relative path, the file must be accessible as a real path.

This is where `asarUnpack` comes in — it tells electron-builder "keep these specific files outside the asar, as regular files."

### What Is "Auto-Update"?

Auto-update lets your app download and install new versions of itself without the user manually downloading and reinstalling. In Electron, this is typically done with the `electron-updater` package:

1. App starts → checks a remote server: "Is there a newer version?"
2. If yes → downloads the update in the background
3. When download completes → notifies user: "Update ready, install on restart?"
4. User restarts the app → new version is applied

**Key concepts:**
- **Feed URL:** The web address where the app checks for updates. This is a JSON file listing available versions.
- **Release channel:** A tier of updates — `latest` (stable, for everyone), `beta` (for early testers), `rc` (release candidate, almost stable).
- **Auto-download:** Should the app automatically download the update when available, or wait for the user to click "Download"?
- **Auto-install on quit:** Should the update be applied automatically when the user quits, or require an explicit "Install" action?

---

## 1. Multi-Target Build With `electron-vite`

The project defines three build targets:
- `main`
- `preload`
- `renderer`

This matters because Electron apps usually ship different code under different trust models:
- main process has Node + Electron authority
- preload is the controlled bridge
- renderer is the UI app

### Plain-Language Explanation

**Why three separate builds instead of one:**

Each target runs in a fundamentally different environment with different capabilities:

| Target | Runs In | Available APIs | Build Needs |
|--------|---------|---------------|-------------|
| **main** | Node.js (full power) | All Node.js built-ins (`fs`, `path`, `os`, etc.), all Electron APIs | Node-compatible output, external dependencies kept as `require()` |
| **preload** | Isolated Node-like context | Limited subset: `contextBridge`, `ipcRenderer`, a few Node APIs | Special output that works in Electron's preload sandbox |
| **renderer** | Chromium browser (no Node) | Web APIs only (DOM, fetch, etc.), plus what preload exposes via `contextBridge` | Browser-compatible output, all Node built-ins excluded |

If you tried to build all three with the same configuration, at least one would be broken. The main process needs `fs` and `path` as Node modules; the renderer would crash if those were included.

**How `electron-vite` handles this:**

`electron-vite` is a build tool that wraps Vite and understands Electron's three-target architecture. Its configuration defines each target separately:

```typescript
// electron.vite.config.ts (simplified concept)
export default {
  main: {
    // Build config for main process
    // Output: dist/main/index.js (Node.js-compatible)
  },
  preload: {
    // Build config for preload scripts
    // Output: dist/preload/index.js (preload sandbox compatible)
  },
  renderer: {
    // Build config for renderer (React UI)
    // Output: dist/renderer/index.html, assets/
  }
};
```

**Blind spot — "What is Vite?":** Vite is a modern build tool that is much faster than older tools like Webpack. It uses native ES modules during development (so code changes appear instantly) and Rollup for production builds (optimized output). `electron-vite` is Vite with Electron-specific configuration — it knows where Electron's main/preload/renderer boundaries are.

**Blind spot — "Why are dependencies marked as 'external'?":** In a browser app build, ALL code is bundled into one file. In Electron's main process, Node.js built-in modules (`fs`, `path`, `electron`) and many npm packages must NOT be bundled because they contain native code or use Node.js features that bundling would break. Marking them as "external" tells the build tool "leave these as `require()` calls, do not try to bundle them."

### Universal Reuse — Your Own Electron Project

Every Electron project using Vite needs this three-target structure. The `electron.vite.config.ts` template:

```typescript
import { defineConfig } from 'electron-vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  main: {
    // Main process: Node.js environment
    build: {
      rollupOptions: {
        external: ['electron', 'better-sqlite3', ...otherNativeModules]
      }
    }
  },
  preload: {
    // Preload: restricted Node-like environment
    build: {
      rollupOptions: {
        external: ['electron']
      }
    }
  },
  renderer: {
    // Renderer: browser environment
    plugins: [react()],
    build: {
      rollupOptions: {
        input: {
          main: 'src/renderer/index.html',
          settings: 'src/renderer/settings.html'
        }
      }
    }
  }
});
```

---

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

### Plain-Language Explanation

**Why separate HTML entries instead of one SPA for all windows:**

A single-page application (SPA) works well when your app has ONE window. All routes live in the same React app, and React Router switches between them. But Cherry Studio's windows are genuinely separate:

- The **main window** (`index.html`) is a full app with sidebar navigation, settings, chat interface
- The **mini window** (`miniWindow.html`) is a compact quick-assistant popup — completely different layout, different purpose
- The **selection toolbar** (`selectionToolbar.html`) is a floating tooltip near selected text — tiny, no navigation, just action buttons
- The **trace window** (`traceWindow.html`) is a debugging/monitoring view — completely different audience and purpose

**Why not just use one SPA and conditionally render?**

1. **Bundle size:** Each window would load ALL the React components for ALL windows, even though most are never used. A separate entry means each window loads only the code it needs.

2. **Performance:** The mini window needs to appear instantly. It should not load the main window's massive chat interface code just to show a small assistant popup.

3. **Isolation:** If the selection toolbar crashes, it should not take down the main window. Separate entries run in separate renderer processes.

4. **Memory:** Unused code in each window wastes RAM. Cherry Studio might have 5 windows open — each should only have the code it actually executes.

**Simple mental model:**
```
One SPA approach:
  Window 1 loads: ALL code (main + mini + toolbar + trace) → waste
  Window 2 loads: ALL code (main + mini + toolbar + trace) → waste

Multi-entry approach:
  Window 1 loads: main code only → efficient
  Window 2 loads: mini code only → efficient
```

### Universal Reuse — Your Own Electron Project

When adding a new window to your Electron app, create a new HTML entry instead of adding routes to an existing SPA:

```typescript
// electron.vite.config.ts renderer section
renderer: {
  build: {
    rollupOptions: {
      input: {
        main: resolve('src/renderer/main/index.html'),
        settings: resolve('src/renderer/settings/index.html'),
        onboarding: resolve('src/renderer/onboarding/index.html'),
      }
    }
  }
}
```

Each entry gets its own `index.html`, its own React root, and its own set of components.

---

## 3. Main-Process Bundle Strategy

The main build:
- marks `electron` and many dependencies as external
- disables manual chunks
- enables `inlineDynamicImports`

That reduces main-process bundling complexity and makes the Electron entry easier to package as a single runtime-oriented bundle.

### Plain-Language Explanation

**Why mark `electron` as external:**

The `electron` package is the entire Electron framework. It is not a normal npm package — it includes native binaries, the Chromium engine, and Node.js integration. You cannot bundle it into a single JavaScript file (it is hundreds of megabytes of native code). It must be kept as an external dependency that the Electron runtime provides.

**Why disable manual chunks:**

"Code splitting" or "chunks" is when the build tool splits your output into multiple files that load on demand. This is great for browser apps (faster initial page load). But in Electron's main process:
- There is no "page load" — the code runs once at startup
- Multiple files add complexity without benefit
- Dynamic imports in the main process can cause issues with Electron's module resolution

So main process builds should output a single file (or as few files as possible).

**Why `inlineDynamicImports`:**

Any `import()` expressions (dynamic imports) in the main process code are converted to inline code instead of creating separate chunk files. This keeps the main process as a single self-contained bundle.

**Blind spot — "What is a dynamic import?":** `import('./module')` (with parentheses, not a static `import` statement) is a "dynamic import." It loads code on demand at runtime instead of at startup. In browser apps, this is used for code splitting ("only load the settings page code when the user clicks Settings"). In the main process, dynamic imports are less useful and can complicate the build, so they are inlined.

### Universal Reuse — Your Own Electron Project

Standard main-process build config for electron-vite:

```typescript
main: {
  build: {
    rollupOptions: {
      external: [
        'electron',
        // Native modules that cannot be bundled
        'better-sqlite3',
        'sharp',
        // Any module with native .node files
      ]
    },
    // Disable code splitting for main process
    chunkSizeWarningLimit: 1000
  }
}
```

---

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

### Plain-Language Explanation of Each Configuration Section

#### `appId` and `productName`

- **`appId`:** A unique, reverse-domain identifier like `com.cherrystudio.app`. This is used by the OS to identify your app uniquely (for protocol registration, notifications, taskbar grouping). It must never change after the first release — changing it makes the OS think the old app was uninstalled and a new app installed.
- **`productName`:** The human-readable name shown to users in the installer, Start menu, and About dialog.

#### Protocol Registration

```yaml
protocols:
  - name: cherrystudio
    schemes:
      - cherrystudio
```

This tells electron-builder to register the `cherrystudio://` protocol in the OS during installation. On Windows, this adds registry entries. On macOS, it adds URL scheme entries to the app's `Info.plist`. On Linux, it adds MIME type entries to the `.desktop` file.

This is why deep links can work "out of the box" after installation — the installer set up the protocol association. However, AppImage users need the runtime setup described in the protocol document (because AppImages are not installed by a package manager).

#### Platform Targets

| Platform | Format | Description |
|----------|--------|-------------|
| Windows | NSIS | Standard installer wizard (`Setup.exe`) |
| Windows | portable | Standalone `.exe`, no installation |
| macOS | dmg | Apple Disk Image (drag to Applications) |
| Linux | AppImage | Single portable executable |
| Linux | deb | Debian/Ubuntu system package |
| Linux | rpm | Red Hat/Fedora system package |

#### File Exclusion Rules

electron-builder includes all files in your project by default. But many files should NOT ship to users:
- Source code (`.ts`, `.tsx`, `.jsx`) — only compiled output is needed
- Development config files (`.eslintrc`, `tsconfig.json`)
- Test files, documentation, build scripts
- `node_modules` that are only used during development

Aggressive exclusion rules (`!**/node_modules/*.md`, `!**/node_modules/*.ts`, etc.) keep the installer small and clean.

#### `asarUnpack`

```yaml
asarUnpack:
  - "resources/**"
  - "node_modules/some-native-module/**"
```

Files matching these patterns are kept as real files on disk, NOT packed into `app.asar`. This is needed for:
- Native `.node` modules (compiled C++ code)
- External executables that the app spawns
- Resource files accessed via `fs.readFileSync` with relative paths

#### Hooks

- **`beforePack`:** Script runs before packaging starts. Used for cleanup, file generation, etc.
- **`afterPack`:** Script runs after packaging. Used for post-processing the packaged app.
- **`afterSign`:** Script runs after code signing (macOS/Windows). Used for notarization (macOS).
- **`artifactBuildCompleted`:** Script runs when a build artifact is ready. Used for CI/CD notifications.

**Blind spot — "What is code signing?":** Code signing is a cryptographic process that verifies "this app was built by the real Cherry Studio team, not modified by malware." On macOS, unsigned apps trigger scary warnings and may be blocked entirely. On Windows, unsigned apps trigger SmartScreen warnings. Code signing requires an expensive certificate from Apple or Microsoft — it is one of the main costs of distributing a desktop app.

### Universal Reuse — Your Own Electron Project

Minimal `electron-builder.yml` for a new project:

```yaml
appId: com.mycompany.myapp
productName: MyApp

directories:
  output: dist

files:
  - "!**/*.{ts,tsx,jsx}"
  - "!**/node_modules/*/{test,tests,doc,docs,example,examples}/**"

win:
  target:
    - target: nsis
      arch: [x64]
    - target: portable
      arch: [x64]

mac:
  target:
    - target: dmg
      arch: [x64, arm64]

linux:
  target:
    - target: AppImage
      arch: [x64]
    - target: deb
      arch: [x64]

protocols:
  - name: myapp
    schemes:
      - myapp

asarUnpack:
  - "resources/**"
```

---

## 5. Why `asarUnpack` Matters Here

Cherry Studio uses native modules, runtime resources, and external binaries that may need real filesystem access at runtime. Packing everything into `app.asar` would break some of those use cases.

So the packaging config selectively unpacks:
- `resources/**`
- runtime binaries
- some native-library artifacts

That is a standard Electron packaging tradeoff: tighter packaging versus runtime accessibility.

### Plain-Language Explanation

**The tradeoff in simple terms:**

| Everything in asar | Selective asarUnpack |
|-------------------|---------------------|
| Clean distribution: 1 archive file | More files on disk |
| Slightly faster app start | Slightly more complex setup |
| But: native modules BREAK | Native modules WORK |
| But: child processes BREAK | Child processes WORK |
| But: some `fs` operations BREAK | All `fs` operations WORK |

**How to know if something needs asarUnpack:**

Ask these questions about each file/dependency:
1. Does it contain compiled native code (`.node` files, `.dll`, `.so`, `.dylib`)? → MUST unpack
2. Does the app launch it as a child process (`child_process.spawn()`)? → MUST unpack
3. Is it read using `fs.readFileSync()` with a relative path? → MUST unpack
4. Is it served over HTTP or loaded via Electron's `protocol` API? → Can stay in asar
5. Is it a JavaScript file loaded via `require()` or `import`? → Can stay in asar

**Why native `.node` files break in asar:** `.node` files are compiled C++ code loaded by Node.js via `process.dlopen()`. The operating system's dynamic linker (`dlopen` on Unix, `LoadLibrary` on Windows) needs a real file path to load the library. It cannot read from inside an archive — it talks directly to the filesystem.

**Blind spot — "Why can't Electron just make asar look like a real filesystem?":** Electron DOES make asar look like a real filesystem for JavaScript-level operations. `fs.readFileSync('/path/to/app.asar/some/file.js')` works because Electron intercepts the call and reads from the archive. But it cannot intercept calls made by native code (C++ `.node` modules) or the operating system's dynamic linker. Those bypass JavaScript entirely and talk to the filesystem directly.

### Universal Reuse — Your Own Electron Project

When you add a new native dependency, always check if it needs unpacking:

```bash
# Check if a package contains native .node files
ls node_modules/some-package/build/Release/*.node

# If yes, add to electron-builder.yml:
# asarUnpack:
#   - "node_modules/some-package/**"
```

And test the packaged app (not just dev mode) to verify it works. Dev mode does not use asar, so native modules always work in dev — the failure only appears after packaging.

---

## 6. Auto-Update Flow

`AppUpdater.ts` builds a more advanced updater than a simple `checkForUpdates()`.

It configures `electron-updater` with:
- app-specific request headers
- environment-aware dev behavior
- auto-download controlled by config
- `autoInstallOnAppQuit = false`

The last setting is important. Cherry Studio intentionally requires an explicit user install action instead of silently installing on quit.

### Plain-Language Explanation

**How `electron-updater` works under the hood:**

1. You publish your app builds to a server (GitHub Releases, S3, or a custom server)
2. Alongside the builds, you publish a `latest.yml` (or `latest-mac.yml`, `latest-linux.yml`) file that describes the latest version
3. `electron-updater` downloads this YAML file, compares the version to the current app version
4. If a newer version exists, it downloads the appropriate installer/delta
5. On next app launch, the new version is applied

**App-specific request headers:** Cherry Studio adds custom HTTP headers to the update check request. This allows the update server to identify which app is checking and serve the correct build.

**Environment-aware dev behavior:** In development, auto-update is meaningless (you're running source code, not a packaged version). Cherry Studio disables it in dev mode. Some apps provide a dev-only "simulate update" feature for testing.

**Auto-download vs auto-install:**

- **Auto-download (enabled):** When an update is available, download it in the background without asking. The user continues working uninterrupted. When the download finishes, show a notification "Update ready — will install on restart."
- **Auto-install on quit (disabled):** The update is NOT applied automatically when the user quits. The app waits for the user to explicitly click "Install Update." This prevents surprises — imagine quitting your app to restart your computer and finding that the app silently changed.

**Why Cherry Studio disables auto-install:** User trust. Silently modifying installed software (even for legitimate updates) can feel like malware behavior. Explicit user consent for installation is the safer UX.

**Blind spot — "What is a delta update?":** Instead of downloading the entire new version (which could be 100MB+), a "delta" only downloads the differences between the old and new versions. If only a few files changed, the delta might be 5MB instead of 100MB. electron-updater supports delta updates on Windows via `nsis-web` target type.

### Universal Reuse — Your Own Electron Project

Basic electron-updater setup:

```typescript
import { autoUpdater } from 'electron-updater';
import { app } from 'electron';

function setupAutoUpdater() {
  // Base configuration
  autoUpdater.setFeedURL('https://updates.myapp.com/latest');

  // Request headers for server-side routing
  autoUpdater.requestHeaders = {
    'X-App-Version': app.getVersion(),
    'X-Platform': process.platform
  };

  // User-visible events
  autoUpdater.on('update-available', () => {
    mainWindow.webContents.send('update-status', 'available');
  });

  autoUpdater.on('download-progress', (progress) => {
    mainWindow.webContents.send('update-progress', progress.percent);
  });

  autoUpdater.on('update-downloaded', () => {
    mainWindow.webContents.send('update-status', 'downloaded');
  });

  // Check for updates (but don't install silently)
  autoUpdater.autoDownload = true;
  autoUpdater.autoInstallOnAppQuit = false;

  autoUpdater.checkForUpdates();
}

// When user clicks "Install Update":
function installUpdate() {
  autoUpdater.quitAndInstall();
}
```

---

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

### Plain-Language Explanation

**Why not just use a single static update URL?**

A single URL like `https://releases.cherrystudio.io/latest` works for simple cases but cannot handle:

1. **Regional mirrors:** Users in China might have slow or blocked access to GitHub. A GitCode mirror provides fast downloads in China.
2. **Release channels:** Beta users should get beta builds, not stable builds. Each channel needs a different feed.
3. **Compatibility gates:** If version 2.0 requires a new database migration, users on version 1.0 should only be offered version 1.9 (the last compatible update), not version 2.0.
4. **Phased rollouts:** You might want to release to 10% of users first, then 50%, then 100%. A dynamic config server can control this.

**How Cherry Studio's feed selection works:**

```
Step 1: Determine channel
  → Is test-plan mode? → Use test channel
  → User config says "beta"? → Use beta channel
  → User config says "rc"? → Use rc channel
  → Default → Use "latest" channel

Step 2: Determine mirror
  → Fetch user's IP country from geo-IP service
  → Country is China? → Use GitCode mirror
  → Otherwise → Use GitHub-based URL

Step 3: Fetch remote config
  → Download JSON from chosen mirror
  → Config lists: channels, versions, minCompatibleVersion

Step 4: Find matching channel
  → Look up the requested channel in the config
  → Check minCompatibleVersion: is the available version compatible?
  → Return the feed URL for the matching channel

Step 5: Fallback
  → If any step fails (network error, config parse error)
  → Use hardcoded default production feed URL
```

**Blind spot — "Why check the user's country by IP?":** GitHub is sometimes slow or blocked in China due to network policies. Cherry Studio maintains a mirror on GitCode (a Chinese code-hosting platform) so Chinese users can download updates at full speed. The IP check is a best-effort guess at the user's location — it is not perfectly accurate but is good enough for choosing between two CDN URLs.

**Blind spot — "What is `minCompatibleVersion`?":** This is a version number declared in the update config that says "users below this version should NOT receive this update." For example, if version 2.0.0 has `minCompatibleVersion: 1.5.0`, users on 1.4.0 will not be offered the 2.0.0 update. This prevents offering an update that would break the user's app because they need an intermediate migration step first.

### Universal Reuse — Your Own Electron Project

Channel-aware update feed selection:

```typescript
async function getUpdateFeedUrl(): Promise<string> {
  const channel = getUserChannel();  // 'latest', 'beta', 'rc'
  const appVersion = app.getVersion();

  // Fetch remote config
  try {
    const config = await fetch('https://updates.myapp.com/config.json').then(r => r.json());

    // Find channel entry compatible with current version
    const channelConfig = config.channels[channel];
    if (!channelConfig) throw new Error(`Channel ${channel} not found`);

    // Check minimum compatible version
    if (semver.lt(appVersion, channelConfig.minCompatibleVersion)) {
      // User needs an intermediate update
      return channelConfig.migrationFeedUrl;
    }

    return channelConfig.feedUrl;
  } catch {
    // Fallback to default production URL
    return 'https://releases.myapp.com/latest';
  }
}
```

---

## 8. Renderer Notification Of Update State

Updater events are pushed back into the UI with IPC:
- `UpdateError`
- `UpdateAvailable`
- `UpdateNotAvailable`
- `DownloadProgress`
- `UpdateDownloaded`

This keeps update policy in main process and update presentation in renderer.

### Plain-Language Explanation

**Why main process owns policy, renderer owns presentation:**

The main process:
- Decides WHEN to check for updates (on app start, periodically)
- Decides WHICH feed URL to use (channel selection, mirror logic)
- Controls the download process (pause, resume, cancel)
- Knows about app lifecycle (do not install update while user is in the middle of work)

The renderer:
- Shows the update status to the user ("Update available — version 1.5.0")
- Shows download progress ("Downloading... 45%")
- Provides action buttons ("Download Now", "Install on Quit", "Remind Later")
- Does NOT control the update process directly

This separation prevents UI bugs from breaking the update mechanism and keeps update logic centralized.

**Each IPC event explained:**

| Event | Meaning | Renderer Action |
|-------|---------|----------------|
| `UpdateError` | Something went wrong during update check/download | Show error message with retry option |
| `UpdateAvailable` | A newer version exists on the server | Show "Update available" notification with version info and release notes |
| `UpdateNotAvailable` | Current version is the latest | Usually silent, or show "You're up to date" in settings |
| `DownloadProgress` | Download is in progress (fires repeatedly) | Update progress bar (percentage, speed, remaining time) |
| `UpdateDownloaded` | Download complete, ready to install | Show "Ready to install — will apply on restart" with Install button |

**Blind spot — "What happens between 'downloaded' and 'installed'?":** The update is downloaded to a temporary location on disk. It is NOT applied yet — the currently running app is untouched. When the user clicks "Install" (or the app restarts), electron-updater launches the installer for the new version. The old app quits, the installer replaces the old files with new ones, and the new version launches. This is why updates require an app restart — you cannot replace running executable files on any operating system.

### Universal Reuse — Your Own Electron Project

Renderer-side update UI component (React example concept):

```typescript
// Main process: forward all updater events
autoUpdater.on('update-available', (info) => {
  mainWindow.webContents.send('update:available', info.version);
});
autoUpdater.on('download-progress', (progress) => {
  mainWindow.webContents.send('update:progress', {
    percent: progress.percent,
    transferred: progress.transferred,
    total: progress.total
  });
});
autoUpdater.on('update-downloaded', () => {
  mainWindow.webContents.send('update:downloaded');
});

// Renderer: listen and display
window.api.onUpdateAvailable((version) => {
  showNotification(`Version ${version} available!`);
});
window.api.onUpdateProgress(({ percent }) => {
  updateProgressBar(percent);
});
window.api.onUpdateDownloaded(() => {
  showInstallButton();
});
```

---

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

### Plain-Language Explanation

**The problem:** Release notes describe what changed in the new version. Cherry Studio has users who speak different languages. Instead of maintaining separate release note files per language (which easily get out of sync), all translations live in one string.

**The marker format:**
```
<!--LANG:en-->
- Added new feature X
- Fixed bug Y
<!--LANG:zh-CN-->
- 新增功能X
- 修复错误Y
<!--LANG:END-->
```

**How extraction works:**
1. Read the full release notes string
2. Find the marker for the user's language (`<!--LANG:zh-CN-->`)
3. Extract everything between that marker and the next `<!--LANG:` or `<!--LANG:END-->`
4. If the user's language is not found, fall back to English (`<!--LANG:en-->`)

**Why this is useful:** This is a lightweight multilingual pattern that works for any desktop app that shows release notes. It does not require a translation framework or external service — just a naming convention and some string parsing.

### Universal Reuse — Your Own Electron Project

```typescript
function extractLocalizedReleaseNotes(
  rawNotes: string,
  language: string
): string {
  // Try to find the requested language block
  const langMarker = `<!--LANG:${language}-->`;
  const startIdx = rawNotes.indexOf(langMarker);

  if (startIdx !== -1) {
    const contentStart = startIdx + langMarker.length;
    const nextMarker = rawNotes.indexOf('<!--LANG:', contentStart);

    if (nextMarker !== -1) {
      return rawNotes.slice(contentStart, nextMarker).trim();
    }
    return rawNotes.slice(contentStart).trim();
  }

  // Fallback to English
  return extractLocalizedReleaseNotes(rawNotes, 'en');
}
```

---

## 10. Practical Takeaways

- Split Electron builds by trust boundary: main, preload, renderer
- Use separate renderer entries for separate native windows
- Keep runtime-required assets out of packed asar when necessary
- Let main process own updater policy
- Choose update feeds dynamically when region, compatibility, and release channel matter

---

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

---

## 12. Typical Application Scenarios

- Add a new renderer entry for a utility window, onboarding flow, or diagnostics surface.
- Ship an extra executable, model file, or runtime asset needed by the main process.
- Change Windows or Linux packaging targets.
- Add a new beta or regional update rule and surface the result in renderer.

---

## 13. Relationship To The Other Electron Records

- This document packages the runtime behaviors described in `electron_main_process_lifecycle.md`; startup assumptions only matter if the packaged app preserves them.
- It packages the window entry points described in `electron_window_tray_menu_architecture.md`.
- It must include protocol metadata required by `electron_protocol_oauth_and_deep_linking.md`.
- It can preserve or break the preload and asset assumptions used by `electron_webview_session_management.md`.
