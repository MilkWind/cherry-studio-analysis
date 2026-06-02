# Cross-Platform Support Principles

This document explains the **underlying principles and design reasoning** behind the cross-platform measures in Cherry Studio. It answers *why* each approach is chosen, not just *what* is done.

---

## 1. Matrix CI Builds — Principle: **Shift Quality Left**

**Why not build on one platform and hope it works elsewhere?**

> Each OS has its own file system semantics, shell environment, DLL/SO loading paths, code signing requirements, and package format expectations. Building natively on each OS in CI catches platform-specific bugs *before* release, not after.

Matrix builds enforce that code changes must compile and pass on all three targets simultaneously — this is the same principle as running tests in CI: **fail fast, fail loud**.

**Trade-off:** Longer CI pipeline duration. Mitigated by running builds in parallel.

---

## 2. Electron-Builder Config — Principle: **Declarative Platform Contracts**

**Why a single config file for all targets?**

> Electron-builder's config is a *declaration* of the contract between your app and each OS. Each target (NSIS, DMG, AppImage) is a different set of OS integration expectations:
>
> - **NSIS:** Windows users expect `.exe` installers with custom install paths, VC++ redist auto-install, and Start Menu entries. NSIS is the de facto Windows installer standard.
> - **DMG:** macOS users expect drag-and-drop `.dmg` installers with code signing and notarization (Gatekeeper compliance is *mandatory* on modern macOS, not optional).
> - **AppImage/DEB/RPM:** Linux has no single package format, so you must support the three major families (AppImage for distro-agnostic portability, DEB for Debian/Ubuntu, RPM for Fedora/RHEL).

**Key insight:** A single config enforces *parity* — you cannot accidentally ship on one platform without the others.

---

## 3. Platform Detection Constants — Principle: **Single Source of Truth**

**Why not scatter `process.platform` checks everywhere?**

> Every inlined `process.platform` check is a maintenance liability — when a new platform appears (or a constant needs to change), each inline check must be hunted down individually.
>
> By centralizing detection into `constant.ts`, you create a **single source of truth** and make platform behavior **auditable**. The renderer duplicates this rather than crossing the IPC boundary because platform detection is synchronous and called on every keystroke in some UI paths — performance matters.

**Why the `os` CSS attribute on `<body>`?**

> Many platform-specific UI tweaks (scrollbar widths, font rendering, spacing) are purely visual. Putting the platform in a CSS attribute lets you handle these in stylesheets instead of JavaScript, keeping the renderer leaner.

---

## 4. Platform-Specific Code — Principle: **Adapt to OS Conventions, Don't Fight Them**

Each OS has deeply ingrained conventions that users expect:

| Convention | Principle |
|---|---|
| **Config paths** | XDG Base Directory (Linux), `~/Library` (macOS), `%APPDATA%` (Windows) — following these means backups, permissions, and user expectations work naturally |
| **Binary naming** | `.exe` on Windows is not decoration — the OS uses it for file type association, security prompts, and PATHEXT resolution |
| **File permissions** | Windows does not use Unix `chmod`; setting executable bits is meaningless and would throw errors |
| **Terminal integration** | Each OS has different default terminals with different CLI argument formats — hardcoding one breaks on the others |
| **Filename validation** | Windows forbids `< > : " / \ \| ? *` and reserved names because the NT kernel reserves them; macOS forbids `:` because HFS+ uses it as a separator; Unix only forbids `/` and null because it's the simplest design |

**Key insight:** The most portable code is not the code that abstracts away the OS, but the code that **correctly detects and adapts** to each OS's native behavior. Fighting OS conventions (e.g., trying to use Unix paths on Windows) creates subtle bugs and poor UX.

---

## 5. Native Module Support — Principle: **Isolate Platform Coupling at the Dependency Boundary**

**Why `optionalDependencies` instead of a single universal package?**

> Native Node modules are compiled against specific Node/Electron ABI versions and OS/arch combinations. A single cross-platform `dependencies` entry would either:
> - Bundle all platform binaries (bloating every installation with unneeded files), or
> - Require `postinstall` compilation (fragile — requires toolchains on the target machine).
>
> `optionalDependencies` lets npm/pnpm resolve only the relevant binary at install time, and `before-pack.js` can further prune unused packages per platform at build time.

**Why the cross-compilation dance in `before-pack.js`?**

> GitHub Actions runners for one platform cannot install native modules for another platform. The `before-pack.js` script temporarily adds the target platform to `supportedArchitectures`, re-runs `pnpm install` to fetch the correct prebuilt binaries, then restores the config. This is a **build-time cross-compilation pattern** — the principle is *download prebuilt binaries rather than compile from source*, which avoids needing full cross-compilation toolchains on CI.

---

## 6. Installer & System Integration — Principle: **Respect OS Security Models**

| Platform | Security Model | Implication |
|---|---|---|
| **macOS** | Gatekeeping + Hardened Runtime | App must be signed, notarized, and declare entitlements (JIT, camera, etc.) upfront. `com.apple.security.cs.disable-library-validation` is needed because Electron loads dynamic libraries at runtime. |
| **Windows** | User Account Control (UAC) + SmartScreen | NSIS must request elevation for per-machine install; VC++ redist check avoids the "#1 cause of Windows app crashes" — missing C++ runtime |
| **Linux** | Distro-specific package managers + FHS | Desktop entries, MIME handlers, and `StartupWMClass` are needed for proper WM integration. RPM build IDs must match the distro's expectations. |

**Key insight:** Each OS has a different trust model. You work *with* it, not against it. Skipping notarization makes your app unbootable on macOS. Skipping VC++ redist makes your app crash on a fresh Windows install.

---

## 7. Runtime Binary Downloads — Principle: **Responsibility Separation**

**Why download runtime binaries at setup, not bundle them?**

> Platform-specific native binaries (Bun, UV, RTK, etc.) are large and vary across 20+ platform/arch combinations. Bundling all of them would:
> - Bloat the installer 10x or more
> - Violate the principle of **thin distribution** — ship only what's needed for the current platform
> - Create maintenance burden for tracking binary versions
>
> Downloading at setup time follows the **Internet-Sourced Runtime** pattern: the app ships the downloader logic, not the binaries. This also allows updating runtimes independently of the app.

**Why use PowerShell on Windows for `download.js`?**

> Windows does not ship `curl` or `wget` by default (until recent Windows 10+). PowerShell's `Invoke-WebRequest` is the most reliable built-in download mechanism on Windows. On Unix, `curl` is universally available. This is **platform-appropriate tool selection** — use the tool each OS guarantees.

---

## 8. Platform-Specific UI — Principle: **Consistent UX, Not Identical UX**

A great cross-platform app does not look the same on every OS — it feels native on each:

| Feature | Principle |
|---|---|
| **macOS accessibility** | macOS requires explicit accessibility trust for global shortcuts. Blocking the feature without trust is better than silently failing. |
| **Linux Wayland detection** | Wayland restricts global hotkeys and input injection for security. Detecting the display server prevents "broken but not crashed" UX. |
| **Windows Mica** | Windows 11 22H2+ introduced Mica material for native-feeling window backgrounds. Supporting it makes the app feel like a first-class Windows app. |
| **Tiling WM detection** | Linux tiling WMs (Hyprland, i3, etc.) have different window behavior. Adjusting to them prevents broken fullscreen or focus issues. |
| **Shortcut mapping** | Ctrl vs Cmd is the most visible platform difference. `CommandOrControl` abstracts the physical key, but the *label* should reflect the platform convention. |

**Key insight:** The goal is **platform appropriateness**, not platform uniformity. A macOS user expects Cmd+Q to quit; a Windows user expects Ctrl+Shift+Q. Both are "consistent" in their own platform context.

---

## 9. MCP Platform Overrides — Principle: **Configuration Over Code**

**Why `platform_overrides` instead of if-else logic in the service?**

> MCP tools are defined by users/plugins as configurations, not code. The service layer has no business knowing the intricacies of every tool's OS-specific command paths.
>
> Pulling platform differences into config space means:
> - New tool integrations don't require code changes to `DxtService.ts`
> - Platform overrides are **explicit and visible** in the config, not hidden in conditional branches
> - Users can fix platform issues by editing configs without rebuilding

This follows the **Inversion of Control** principle — the framework provides the override mechanism, the config provides the specifics.

---

## Overall Design Philosophy

Cherry Studio's cross-platform strategy follows four high-level principles:

| Principle | Meaning |
|---|---|
| **1. Native parity** | Every platform gets equal build, test, and release treatment. No "Windows as an afterthought." |
| **2. Explicit detection** | Platform differences are handled by centralized detection constants, not scattered heuristics. |
| **3. OS-appropriate tools** | Use PowerShell on Windows, `curl` on Unix, NSIS on Windows, DMG on macOS. Don't fight the platform. |
| **4. Configuration over conditionals** | Platform-specific behavior is pushed to config layers (electron-builder.yml, MCP overrides) where possible. |

The result is an app that **behaves natively on each OS** while sharing ~90% of its codebase — the 10% of platform-specific code is deliberate, isolated, and principled.
