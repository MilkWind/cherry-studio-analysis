# Cross-Platform Support Measures in Cherry Studio

Cherry Studio is a desktop client that supports **Windows, macOS, and Linux**. Below is a comprehensive breakdown of the measures taken to achieve cross-platform compatibility.

---

## 1. CI/CD — GitHub Actions Matrix Builds

The project uses **GitHub Actions** with a matrix strategy to build, sign, and package on all three platforms simultaneously.

| File | Purpose |
|------|---------|
| `.github/workflows/release.yml` | Main release pipeline — builds on `macos-latest`, `windows-latest`, `ubuntu-latest` |
| `.github/workflows/nightly-build.yml` | Nightly builds with the same matrix |
| `.github/workflows/v2-daily-preview-build.yml` | Preview builds for v2 |

Each workflow:
- Installs platform-specific dependencies (e.g., `brew install python-setuptools` on macOS; `rpm` + dev libraries on Linux)
- Runs a platform-specific build script (`build:win`, `build:mac`, or `build:linux`)
- Signs artifacts where applicable (Apple notarization on macOS, `signtool` on Windows)
- Collects platform-specific artifacts (`.exe`, `.dmg`, `.AppImage`, `.deb`, `.rpm`, etc.)
- Computes SHA256 checksums using platform-appropriate tools (`Get-FileHash` on Windows, `shasum`/`sha256sum` on Unix)

---

## 2. Electron-Builder Configuration

**File:** `electron-builder.yml`

| Platform | Targets | Key Details |
|----------|---------|-------------|
| **Windows** | NSIS installer, Portable | Custom sign script (`scripts/win-sign.js`), one-click off, custom install directory allowed, VC++ redist check via `build/nsis-installer.nsh` |
| **macOS** | DMG, ZIP | Hardened runtime entitlements (`build/entitlements.mac.plist`), notarization via `@electron/notarize` (`scripts/notarize.js`) |
| **Linux** | AppImage, DEB, RPM | Desktop entry with `StartupWMClass`, MIME protocol handler for `x-scheme-handler/cherrystudio`, RPM build ID workaround |

Build lifecycle hooks: `before-pack.js`, `after-pack.js`, `afterSign` (notarize), `artifactBuildCompleted`.

---

## 3. Platform Detection Constants

Central constants in both main and renderer processes:

**`src/main/constant.ts`** (main process):
```ts
export const isMac = process.platform === 'darwin'
export const isWin = process.platform === 'win32'
export const isLinux = process.platform === 'linux'
export const isPortable = isWin && 'PORTABLE_EXECUTABLE_DIR' in process.env
```

**`src/renderer/src/config/constant.ts`** (renderer process):
```ts
export const platform = window.electron?.process?.platform
export const isMac = platform === 'darwin'
export const isWin = platform === 'win32' || platform === 'win64'
export const isLinux = platform === 'linux'
```

The renderer also sets an `os` attribute on `document.body` (`'mac' | 'windows' | 'linux'`) for CSS-level platform theming (`ThemeProvider.tsx`).

---

## 4. Platform-Specific Code Patterns

### System Paths
Different platforms use different base paths for config/data storage:

| Purpose | macOS | Windows | Linux |
|---------|-------|---------|-------|
| App data | `~/Library/Application Support/<app>` | `%APPDATA%/<app>` | `$XDG_CONFIG_HOME/<app>` |
| Obsidian vault config | `~/Library/Application Support/obsidian/obsidian.json` | `%APPDATA%\obsidian\obsidian.json` | Custom resolution |
| System roots | `/`, `/usr`, `/etc` | Drive letters (C:\, D:\) | `/`, `/usr`, `/etc` |

### Binary Naming
- Windows executables get `.exe` suffix (e.g., `rg.exe`, `rtk.exe`)
- macOS/Linux use bare names (e.g., `rg`, `rtk`)

### Command Execution
- **Environment variables:** `set VAR=value` on Windows vs `export VAR="value"` on Unix
- **Command checking:** `where` (Windows) vs `command -v` (Unix/macOS)
- **File permissions:** `chmodSync(0o755)` only on non-Windows (`download-rtk-binaries.js`, `rtk.ts`)

### Terminal Integration (`CodeToolsService.ts`)
- **macOS:** Probes for Terminal.app, iTerm2, etc. via `terminalConfig.command()`
- **Windows:** Creates `.bat` files with `@echo off`, `chcp 65001`, `pushd`/`popd`, `pause`; uses Windows Terminal or selected terminal
- **Linux:** Probes for `gnome-terminal`, `konsole`, `deepin-terminal`, `xterm`, `x-terminal-emulator` with appropriate CLI flags

### File Name Validation (`file.ts`)
- **Windows:** Rejects chars `< > : " / \ | ? *` and reserved names (CON, PRN, etc.)
- **macOS:** Rejects `:`
- **Unix/Linux:** Rejects `/`

### Backup & Restore (`BackupManager.ts`)
- Stores `process.platform` in backup metadata
- On restore, logs a warning if backup platform differs from current
- Windows uses `.restore` suffix to avoid file lock issues; macOS/Linux do direct replacement

---

## 5. Native Module Support

**`package.json`** declares platform-specific `optionalDependencies` for all target platform/arch combinations:

| Package | Supported Platforms |
|---------|-------------------|
| `@img/sharp` | darwin (arm64, x64), linux (arm64, x64), linuxmusl (arm64, x64), win32 (arm64, x64) |
| `@libsql/*` | darwin (arm64, x64), linux (arm64-gnu/musl, x64-gnu/musl), win32 (x64-msvc) |
| `@napi-rs/canvas` | darwin (arm64, x64), linux (arm64-gnu/musl, x64-gnu/musl), win32 (arm64-msvc, x64-msvc) |
| `@napi-rs/system-ocr` | darwin (arm64, x64), win32 (arm64-msvc, x64-msvc) |
| `@strongtz/win32-arm64-msvc` | win32-arm64 only |

The `before-pack.js` script manages cross-compilation: when building for a different platform than the host, it temporarily modifies `pnpm-workspace.yaml` to add the target's `supportedArchitectures`, runs `pnpm install`, then restores the config. It also filters out irrelevant native packages per platform/arch (e.g., excludes macOS arm64 packages when building for Windows x64). Windows ARM64 builds keep x64 packages for emulation compatibility.

---

## 6. Installer & System Integration

### Windows (NSIS) — `build/nsis-installer.nsh`
- Architecture compatibility check (prevents x64 installer on ARM64-only systems)
- VC++ Redistributable detection and automatic download/install
- Per-machine install elevation

### macOS — `build/entitlements.mac.plist`
- Enables JIT compilation, unsigned executable memory, DYLD environment variables
- Disables library validation (needed for Electron)
- Usage descriptions for camera, microphone, documents, downloads

### Linux
- Desktop entry with `StartupWMClass: CherryStudio` for proper window manager integration
- MIME protocol handler registration: `x-scheme-handler/cherrystudio`
- RPM build ID workaround via fpm flags

---

## 7. Runtime Binary Downloads

The project downloads platform-specific binaries at setup time:

| Script | Binary | Platform Coverage |
|--------|--------|-----------------|
| `resources/scripts/install-bun.js` | Bun runtime | darwin (arm64, x64), win32 (x64, arm64), linux (x64, arm64, musl variants) |
| `resources/scripts/install-uv.js` | UV | 20+ platform/arch combos including darwin, win32 (arm64, ia32, x64), linux (arm64, ia32, ppc64, s390x, etc.) |
| `resources/scripts/download-rtk-binaries.js` | RTK | darwin (arm64, x64), linux (x64, arm64), win32 (x64) |
| `resources/scripts/install-openclaw.js` | OpenClaw | Uses `os.platform()` + `os.arch()` |
| `resources/scripts/install-ovms.js` | OVMS | Uses `os.platform()` |
| `resources/scripts/download.js` | General | Uses PowerShell download on Windows only |

---

## 8. Platform-Specific UI Features

### Selection Assistant (`SelectionAssistantSettings.tsx`)
- **macOS:** Checks accessibility trust (`isProcessTrusted()`); disables assistant if not trusted
- **Linux:** Wayland vs XWayland detection, input device access, compositor compatibility checks
- **Windows:** Shows `ctrlkey` trigger mode option

### Window Management (`windowUtil.ts`)
- **Linux:** Detects tiling window managers (Hyprland, i3, Sway, etc.) for adaptive behavior
- **Windows:** Mica material support on Windows 11 22H2+ (`isWindowsMicaSupported`)
- **macOS:** Always treated as non-tiling

### Shortcut Handling (`ShortcutSettings.tsx`)
- Windows/Linux: Ctrl maps to `CommandOrControl`
- macOS: Meta maps to `CommandOrControl`

### Protocol Handler (`ProtocolClient.ts`)
- **Linux AppImage:** Special handling for deep links via `.desktop` file creation
- Falls back to secondary instance protocol on all platforms

---

## 9. MCP Tool Platform Overrides (`DxtService.ts`)

MCP tool configurations support `platform_overrides` — allowing per-platform command, argument, and environment variable overrides, so the same tool config can work seamlessly across operating systems.

---

## Summary

Cherry Studio achieves cross-platform support through a multi-layered approach:

1. **Build layer:** Matrix CI builds on all three OS runners with platform-specific packaging
2. **Packaging layer:** Electron-builder with per-platform targets (NSIS/DMG/AppImage), code signing, and installer logic
3. **Runtime layer:** Central platform detection constants, native module management, and OS-specific code paths for filesystem, terminal, process, and UI
4. **Distribution layer:** Platform-specific artifacts with appropriate formats and checksums
