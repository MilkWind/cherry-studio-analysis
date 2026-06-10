# Electron Window, Tray, And Menu Architecture In Cherry Studio

This tutorial explains how Cherry Studio implements its native desktop shell around `BrowserWindow`, tray integration, and the macOS application menu.

Primary files:
- `src/main/services/WindowService.ts`
- `src/main/services/TrayService.ts`
- `src/main/services/AppMenuService.ts`
- `src/main/utils/windowUtil.ts`

---

## Beginner's Prerequisites: Concepts You Need Before Reading

### What Is a Window in Electron?

In a regular web app, a "window" is just a browser tab. In Electron, a `BrowserWindow` is a **real operating-system window** — the kind you can drag around, minimize to the taskbar, resize by pulling the edges, and close with an X button. Electron creates and controls these windows from the main process, and each one loads an HTML file from your renderer code.

**Simple mental model:** `new BrowserWindow({width: 800, height: 600})` is the programmatic equivalent of opening a new Chrome window sized to 800x600. But unlike Chrome, you control everything: whether it has a title bar, what happens when the user clicks X, whether it stays on top of other windows, etc.

### What Is a System Tray?

The "system tray" (also called "notification area" on Windows or "menu bar extras" on macOS) is the area near the clock where small app icons live. Apps like Discord, Spotify, Dropbox, and antivirus software put icons there. The tray lets an app:

- Stay running in the background when its main window is closed
- Show a quick-status indicator (like "syncing" or "new message")
- Provide a right-click menu for quick actions (Show/Hide, Quit, etc.)

**Electron's tray API:** `new Tray(iconPath)` creates a tray icon. You provide a path to an image file (PNG on Windows/Linux, "Template" PNG on macOS). The tray object lets you set a tooltip, attach a context menu, and handle click events.

### What Is a Native Menu?

A "native menu" is a menu drawn by the operating system, not by HTML/CSS. There are two kinds in Electron:

1. **The application menu** (macOS only) — The menu bar at the top of the screen: 🍎 App Name, File, Edit, View, Window, Help. This is standard on macOS and expected by users.
2. **Context menus** (all platforms) — The menu that appears when you right-click on something (a tray icon, a window, or an element inside a web page).

**Why "native" matters:** Native menus look and feel exactly like every other app on the platform. They support platform-specific features like the macOS search box in the Help menu, keyboard shortcut display, and system-wide services. HTML-based menus cannot match this.

### Key Electron Classes Referenced in This Document

| Class/API | What It Does | Platform |
|-----------|-------------|----------|
| `BrowserWindow` | Creates and controls a desktop window | All |
| `Tray` | Creates a system tray icon | All |
| `Menu` / `MenuItem` | Builds native menus | All |
| `screen` | Provides information about connected displays | All |
| `shell.openExternal()` | Opens a URL in the user's default web browser | All |

### Blind Spot — "What Is 'Window Chrome'?"

In UI terminology, "chrome" does not mean Google Chrome. It means the **non-content parts of a window** that the operating system draws: the title bar, the close/minimize/maximize buttons, the window borders. "Frameless window" means a window where Electron does NOT ask the OS to draw these parts — the app draws its own custom title bar using HTML/CSS. "Hidden title bar" (macOS) means the OS title bar is hidden but the traffic-light buttons (close/minimize/maximize) remain.

---

## 1. Why `WindowService` Exists

Cherry Studio does not create windows inline in `main/index.ts`. Instead, `WindowService` owns:
- main window creation
- mini window creation
- window state persistence
- platform-specific title bar rules
- close/minimize/show logic
- guest content handlers

That is a common Electron scaling pattern: keep the main entry focused on lifecycle, and put `BrowserWindow` policy into a service.

### Why Centralize Window Management (Simple Principle)

In a one-window app, `new BrowserWindow({...})` directly in `index.ts` works fine. But as the app grows, window-related code multiplies:

- "When the user clicks X, should we quit or hide to tray?"
- "Where on screen was the window last time?"
- "On macOS, the title bar should be hidden; on Windows, it should be standard."
- "The mini window should stay on top and skip the taskbar."

Without a centralized service, these rules get scattered across the codebase in event listeners, utility functions, and inline configuration objects. Changing behavior (like "make close-to-tray optional") requires finding and updating code in multiple places.

A `WindowService` centralizes all window policy in one place. It is the single source of truth for "how do windows behave in this app."

### Universal Reuse — Your Own Electron Project

Any Electron app with more than one window type benefits from a WindowService. The minimum structure:

```typescript
class WindowService {
  private mainWindow: BrowserWindow | null = null;
  private secondaryWindows: Map<string, BrowserWindow> = new Map();

  createMainWindow(): BrowserWindow { /* ... */ }
  createSecondaryWindow(type: string): BrowserWindow { /* ... */ }
  getMainWindow(): BrowserWindow | null { /* ... */ }
  closeWindow(type: string): void { /* ... */ }

  // Platform-specific configuration
  private getTitleBarConfig(): TitleBarConfig {
    if (process.platform === 'darwin') return { titleBarStyle: 'hidden' };
    if (process.platform === 'win32') return { frame: true };
    return { frame: true };
  }
}
```

---

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

### Plain-Language Explanation of Each Window Setting

#### `show: false` + `ready-to-show`

**What it does:** The window is created invisibly. It only becomes visible when the `ready-to-show` event fires (meaning the HTML page has finished loading and is ready to display).

**Why we do this:** If you create a window with `show: true` (the default), the user sees a white flash while the HTML page loads. On a slow computer, this flash can last a noticeable moment and looks unprofessional. Creating the window hidden and showing it only when content is ready prevents this visual flash.

**Simple principle:**
```
show: true → Window appears immediately (empty white) → Page loads → Content appears
show: false → Window created invisibly → Page loads → Window appears (content already there)
```

**Universal reuse:** Every Electron app should use `show: false` + `ready-to-show` for its main window. The flash is universally considered bad UX.

#### `autoHideMenuBar: true`

**What it does:** Hides the built-in menu bar (File, Edit, View, etc.) until the user presses the Alt key.

**Why we do this:** Electron's default menu bar is generic and contains many actions that do not apply to a custom app (like "Reload" and "Toggle Developer Tools" in production). Hiding it gives a cleaner look while still letting power users access it via Alt.

**Universal reuse:** Most consumer-facing Electron apps set `autoHideMenuBar: true`. Developer tools and IDEs (like VS Code) typically keep the menu visible.

#### `vibrancy: 'sidebar'` + `visualEffectState: 'active'` (macOS)

**What it does:** Applies macOS's "vibrancy" effect, which makes the window background semi-transparent and blurred, showing a subtle hint of the wallpaper behind it. `'sidebar'` is the specific style (used in Finder sidebars and Mail.app). `visualEffectState: 'active'` keeps the effect visible even when the window is not focused.

**Why we do this:** It makes the app look like a native macOS application. Many Apple-built apps (Finder, Mail, Notes, App Store) use vibrancy for their sidebars and backgrounds. Users subconsciously expect high-quality macOS apps to use these visual effects.

**Blind spot — "What is vibrancy technically?":** Vibrancy is a macOS compositing effect. The window manager takes the content behind your window (other windows, the desktop wallpaper), blurs it, blends it with your window's background color, and uses the result as the window background. This is done entirely by the GPU — your app just declares "I want vibrancy" and the OS does the rest.

#### `webviewTag: true`

**What it does:** Enables the `<webview>` HTML tag in this window's renderer. Without this, `<webview>` elements are not recognized.

**Why we do this:** Cherry Studio embeds external web pages (mini apps) inside `<webview>` tags. See `electron_webview_session_management.md` for full detail.

**Important safety note:** `<webview>` is powerful — it runs external content in a separate process. Only enable it if your app actually uses webviews. Enabling it unnecessarily increases the attack surface.

#### `backgroundThrottling: false`

**What it does:** Prevents Chromium from reducing the renderer process's CPU priority when the window is in the background (minimized or behind other windows).

**Why we do this:** By default, Chromium deprioritizes background tabs/windows to save battery. For Cherry Studio, the main window might be running AI inference or other background work that should not be throttled. Disabling throttling ensures consistent performance even when the user switches to another app.

**Tradeoff:** Higher CPU usage when the app is in the background. Only disable this if your background work genuinely needs full speed.

#### Custom Zoom Factor

**What it does:** Sets the page zoom level (like pressing Ctrl+/Ctrl- in a browser) to a user-configured value.

**Why we do this:** Users have different display preferences (screen size, eyesight, etc.). Persisting the zoom factor as a user preference and applying it on window creation ensures consistent experience.

**Blind spot — "Electron resets zoom on navigation":** A known Electron/Chromium quirk is that in-page navigation (SPA route changes) can reset the zoom factor to 100%. Cherry Studio handles this by reapplying the zoom factor in the `did-navigate-in-page` event listener — a workaround for a Chromium behavior that is specifically mentioned in section 4 below.

### Platform-Specific Title Bar Strategies — Why Different Per OS?

| Platform | Strategy | Why |
|----------|---------|-----|
| **macOS** | `titleBarStyle: 'hidden'` with `titleBarOverlay` | macOS users expect modern apps to use the full window height (no thick title bar). The traffic-light buttons float over the content. |
| **Windows 11** | `backgroundMaterial: 'mica'` | Mica is Windows 11's signature look — the title bar samples the desktop wallpaper and creates a tinted, frosted-glass effect. It signals "this is a modern Windows 11 app." |
| **Windows 10** | Standard title bar | Mica is not available on Windows 10, so the standard OS title bar is used. |
| **Linux** | Config-dependent (hidden or system) | Linux desktop environments vary widely (GNOME, KDE, etc.). Letting users choose between a custom title bar and the system title bar accommodates different desktop setups. |

**Simple principle:** Users on each platform have subconscious expectations about how app windows should look. A "good" Electron app matches the native look of each platform rather than forcing one custom style everywhere. A Windows user should feel "this feels like a Windows app," not "this feels like a web page in a box."

### Universal Reuse — Your Own Electron Project

When creating a `BrowserWindow`, always ask these questions:
1. Should the window be hidden until content is ready? (almost always yes)
2. Should the menu bar be visible? (consumer apps: usually no; dev tools: yes)
3. What platform-specific title bar style makes sense? (check `process.platform`)
4. Should the window throttle in the background? (usually yes, unless doing background work)
5. Does this window need webview support? (only if actually embedding external content)

---

## 3. Window State And Geometry

Two separate persisted states are used:
- default main window state
- `miniWindow-state.json` for the quick assistant

This lets each window type remember its own geometry independently.

### Plain-Language Explanation

**"Window state" / "Geometry":** The window's position on screen (x, y coordinates) and its size (width, height), plus whether it was maximized or fullscreen. "Persisting state" means saving these numbers when the app closes and restoring them when it reopens, so the window appears exactly where the user left it.

**How `electron-window-state` works (simple principle):**
1. When the app starts and creates a window, the library reads the saved state from a JSON file (e.g., `window-state.json`).
2. It applies the saved position, size, and maximized/fullscreen state to the new window.
3. It listens for window move, resize, maximize, and fullscreen events.
4. When any of these change, it writes the new values back to the JSON file.
5. Next time the app starts, the window is exactly where it was.

**Why separate state per window type:** The main window and the mini assistant window serve different purposes and have different default sizes. If they shared state, closing the mini window in a small size might cause the main window to open small next time (or vice versa). Each window type needs its own memory.

**Blind spot — "What if the saved position is off-screen?":** This is a real bug. If the user had a dual-monitor setup and then disconnected one monitor, the saved position might be on the now-removed screen (at coordinates that no longer exist). Good window-state libraries (including `electron-window-state`) handle this: they check if the saved position falls within any currently connected display, and if not, fall back to a default position.

### Universal Reuse — Your Own Electron Project

Every Electron app with resizable windows should persist window state. Users expect apps to remember where they were:

```typescript
import windowStateKeeper from 'electron-window-state';

function createMainWindow() {
  const state = windowStateKeeper({
    defaultWidth: 1200,
    defaultHeight: 800,
    file: 'main-window-state.json'  // Separate file per window type
  });

  const win = new BrowserWindow({
    x: state.x,
    y: state.y,
    width: state.width,
    height: state.height,
    // ...
  });

  state.manage(win);  // Auto-save position/size changes
  return win;
}
```

---

## 4. Main Window Event Strategy

Cherry Studio attaches several event groups to the main window:

- `ready-to-show`: decides whether launch-to-tray should suppress the initial show
- `enter-full-screen` / `leave-full-screen`: pushes fullscreen state to the renderer
- `will-resize`, `resize`, `maximize`, `unmaximize`, `restore`: resends window size and reapplies zoom factor
- `did-navigate-in-page`: fixes Electron zoom reset during in-page navigation
- `render-process-gone`: reload once, or exit if crashes repeat too quickly

This is a good example of Electron code compensating for platform bugs and renderer quirks rather than only responding to ideal events.

### Plain-Language Explanation of Each Event Group

#### `ready-to-show` → Launch-to-Tray Check

**What it does:** When the page is loaded and ready to display, this event fires. The handler checks: "Is the app configured to start hidden to tray?" If yes, it keeps the window hidden. If no, it shows the window normally.

**Why we need this:** This is the only safe moment to make the show/hide decision. Before this event, the page is not ready. After this event, the window would flash visible before hiding.

#### Fullscreen Events

**What it does:** When the user enters or leaves fullscreen (F11 or macOS green button), the main process sends an IPC message to the renderer: `IpcChannel.CurrentWindowStateChange`.

**Why we need this:** The renderer might need to adjust its layout for fullscreen mode (hide certain UI elements, adjust spacing, etc.). The renderer cannot detect fullscreen state on its own because the fullscreen transition is managed by the OS window manager, not by the web page.

#### Resize/Maximize Events → Reapply Zoom

**What it does:** On resize, maximize, unmaximize, and restore, the handler sends the new window size to the renderer AND reapplies the custom zoom factor.

**Why we need this:** This is a workaround for an Electron bug. Some window operations (especially maximize/restore on certain platforms) can cause the zoom factor to reset to 100%. By reapplying the zoom factor on every size change, the app ensures the user's preferred zoom stays active.

#### `did-navigate-in-page` → Fix Zoom Reset

**What it does:** When the renderer navigates within the same page (SPA route change), reapplies the zoom factor.

**Why we need this:** This is another workaround for a Chromium behavior. In single-page applications (like Cherry Studio's React UI), "navigation" happens without loading a new page (React Router changes the URL and renders different components). Chromium sometimes treats this as a new page load for zoom purposes, resetting to 100%.

**Simple principle:** Electron is built on Chromium, and Chromium has behaviors designed for traditional multi-page websites. Single-page apps trigger edge cases where Chromium's assumptions break. The `did-navigate-in-page` handler is a patch for one such mismatch.

#### `render-process-gone` → Crash Recovery

**What it does:** When the renderer process crashes or is killed (OOM, GPU crash, etc.), this event fires. The handler:
1. On the first crash: reload the page (give it another chance).
2. If it crashes again quickly: exit the app (something is fundamentally wrong — infinite reload loop would be worse).

**Why we need this:** Without this handler, a renderer crash leaves the user staring at a blank white window with no feedback. The reload-once-then-quit strategy balances "try to recover" with "don't get stuck in a crash loop."

**Blind spot — "Why would a renderer crash?":** Renderer crashes happen for many reasons: a GPU driver bug, an out-of-memory condition (loading a huge file or image), a bug in Chromium itself, or a JavaScript infinite loop that the engine cannot interrupt. They are rare but real, especially on systems with older GPU drivers.

### Universal Reuse — Your Own Electron Project

These event handlers are a good template for any Electron app's main window:

```typescript
let crashCount = 0;
let lastCrashTime = 0;

mainWindow.on('ready-to-show', () => {
  if (!launchToTray) mainWindow.show();
});

mainWindow.on('enter-full-screen', () => {
  mainWindow.webContents.send('fullscreen-changed', true);
});
mainWindow.on('leave-full-screen', () => {
  mainWindow.webContents.send('fullscreen-changed', false);
});

mainWindow.on('render-process-gone', (event, details) => {
  const now = Date.now();
  if (now - lastCrashTime < 10000) {
    crashCount++;
    if (crashCount > 2) {
      app.quit();  // Crash loop detected
      return;
    }
  } else {
    crashCount = 0;
  }
  lastCrashTime = now;
  mainWindow.reload();  // Try recovering
});
```

---

## 5. Close-To-Tray Behavior

Cherry Studio intercepts the main window `close` event.

Decision path:
- if the app is already quitting, allow normal quit
- if tray is disabled, quit directly on Windows/Linux
- if tray is enabled and tray-on-close is enabled, prevent close and hide the window
- on macOS, also hide the dock when closing to tray

The important Electron concept is that "close window" and "quit app" are separate behaviors, and desktop apps often redefine that boundary.

### Plain-Language Explanation

**The `close` event:** When the user clicks the X button, presses Alt+F4, or Cmd+W, the window emits a `close` event. By default, Electron closes the window, and if it was the last window, quits the app (on Windows/Linux) or keeps running with no windows (on macOS).

**Why intercept it:** Desktop apps with tray support need different behavior:
- The X button should HIDE the window (the app keeps running in the tray)
- "Quit" from the tray menu or Cmd+Q should ACTUALLY QUIT

The `close` event handler distinguishes these two cases using the `isQuitting` flag (set in `before-quit`, as described in the lifecycle document).

**Simple decision tree:**
```
User clicks X on main window
  → Is the app quitting? (isQuitting flag)
    → YES: allow window to close normally → app quits
    → NO: Is tray enabled AND close-to-tray setting on?
      → YES: prevent default close action, hide window instead
      → NO: allow window to close (quit app on Windows/Linux)
```

**MacOS dock hiding:** On macOS, when the window hides to tray, the app remains running with a dot under the dock icon. Hiding the dock icon (`app.dock.hide()`) makes the app disappear from the dock entirely, leaving only the tray icon — which is the expected behavior for a background/tray app.

**Blind spot — "What happens if you don't call `event.preventDefault()` in the close handler?":** The default close action proceeds: the window is destroyed, its renderer process is terminated, and the `BrowserWindow` object becomes unusable. Calling `event.preventDefault()` stops this — the window stays alive. You can then `.hide()` it (keeps the window alive but invisible) or `.minimize()` it.

### Universal Reuse — Your Own Electron Project

This is the standard close-to-tray recipe for any Electron app:

```typescript
let isQuitting = false;

app.on('before-quit', () => { isQuitting = true; });

mainWindow.on('close', (event) => {
  if (!isQuitting && trayEnabled && closeToTrayEnabled) {
    event.preventDefault();
    mainWindow.hide();
    if (process.platform === 'darwin') {
      app.dock.hide();  // macOS: hide dock icon too
    }
  }
  // If isQuitting, let the window close normally
});
```

---

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

### Plain-Language Explanation

**"Desktop utility window":** A window that behaves more like a tool or widget than a full application window. Think of Spotlight (macOS), the emoji picker, or a screen capture tool — they appear on demand, float above other windows, and disappear when you click away.

**Each mini-window setting explained:**

| Setting | What It Does | Why |
|---------|-------------|-----|
| `frame: false` (frameless) | No OS-drawn title bar or borders | The mini window has its own custom-designed header; OS chrome would ruin the look |
| `alwaysOnTop: true` | Stays above all other windows | This is an assistant tool — the user summoned it and expects it to be visible over whatever they were working on |
| `skipTaskbar: true` | Does not appear in the taskbar/dock | It is a transient tool, not a persistent application window. Having it in the taskbar would be distracting |
| `show: false` | Created hidden | It only appears when the user invokes it (hotkey or button click) |
| Preloading | Window is created during app startup but kept hidden | The first time the user summons it, it appears instantly instead of waiting for a new window to be created and load its page |

**Display-aware positioning:** When the user summons the mini window, it should appear on the same monitor as their mouse cursor. The code uses `screen.getCursorScreenPoint()` to find which display the cursor is on, then positions the mini window on that display. Without this, the mini window might appear on a different monitor than the one the user is looking at.

**Pin vs auto-hide:** "Pinning" means the mini window stays visible until the user explicitly closes it. "Auto-hide on blur" means it disappears when the user clicks somewhere else (loses focus). This is a user preference — some want it always available, others want it out of the way when not actively used.

**Fullscreen-safe visibility (macOS):** On macOS, `alwaysOnTop` windows do NOT appear above fullscreen apps by default. They are confined to the desktop space. Cherry Studio handles this by detecting fullscreen state and adjusting the window level accordingly so the mini window can appear over a fullscreen app when summoned.

**Blind spot — "What is 'window level' on macOS?":** macOS assigns every window a "level" (an integer) that determines its stacking order. Normal windows are at level 0. Floating windows (like the mini assistant) are at a higher level so they stay above normal windows. Fullscreen app windows are at an even higher level. To show something above a fullscreen app, you must set the window level higher than the fullscreen level. This is macOS-specific — Windows and Linux handle window stacking differently.

### Universal Reuse — Your Own Electron Project

The mini-window pattern is useful for any "quick access" tool in an Electron app:

```typescript
function createMiniWindow() {
  const mini = new BrowserWindow({
    width: 400,
    height: 500,
    frame: false,
    alwaysOnTop: true,
    skipTaskbar: true,
    show: false,
    webPreferences: { preload: '...' }
  });

  mini.loadFile('miniWindow.html');

  // Position near cursor when shown
  mini.on('show', () => {
    const cursor = screen.getCursorScreenPoint();
    const display = screen.getDisplayNearestPoint(cursor);
    const { x, y } = display.workArea;
    mini.setPosition(x + 50, y + 50);
  });

  // Auto-hide on blur (configurable)
  mini.on('blur', () => {
    if (!isPinned) mini.hide();
  });

  return mini;
}
```

---

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

### Plain-Language Explanation

**"Template images" (macOS):** On macOS, tray icons should be "template" images — simple monochrome shapes where the OS automatically adjusts the color for light mode (dark icon) and dark mode (light icon). The naming convention is to suffix the filename with `Template` (e.g., `iconTemplate.png`). The OS does the color adjustment; you just provide the shape.

**Why icon selection differs per OS:**
- **macOS**: Expects template images (PDF or PNG @2x), small size (~18x18 points), auto-colored for light/dark mode
- **Windows**: Expects ICO or PNG files, 16x16 or 32x32 pixels, with the actual colors you want (no auto-coloring)
- **Linux**: Varies by desktop environment; generally PNG files work, 22x22 or 24x24 pixels

**Localized context menus:** The tray menu text (Show Window, Quit, etc.) should display in the user's language. Since tray menus are built in the main process, the tray service needs access to localization data, and must rebuild the menu when the user changes the app language.

**Why destruction/recreation on config change:** Electron's `Tray` API does not support updating all properties of a live tray instance. Some changes (like switching the icon or fundamentally restructuring the menu) require destroying the old `Tray` and creating a new one. The service detects relevant config changes and performs this rebuild automatically.

**Click behavior routing:** A single click on the tray icon can be configured to:
- Show the main window (for full access)
- Show the mini window (for quick access to the assistant)
This is a user preference stored in config — the tray service reads it and dispatches accordingly.

**Blind spot — "Why does Linux need explicit `setContextMenu`?":** On macOS and Windows, `new Tray(icon)` + `tray.setContextMenu(menu)` automatically sets up the right-click behavior. On some Linux desktop environments, the context menu must be explicitly re-set after certain operations or it stops appearing. This is a known Electron-on-Linux quirk.

### Universal Reuse — Your Own Electron Project

A basic tray setup for any Electron app:

```typescript
import { Tray, Menu, nativeImage } from 'electron';
import path from 'path';

function createTray(mainWindow: BrowserWindow) {
  const iconPath = path.join(__dirname, 'assets/tray-icon.png');
  const tray = new Tray(nativeImage.createFromPath(iconPath));

  const contextMenu = Menu.buildFromTemplate([
    {
      label: 'Show Window',
      click: () => mainWindow.show()
    },
    {
      label: 'Quit',
      click: () => {
        app.isQuitting = true;
        app.quit();
      }
    }
  ]);

  tray.setToolTip('My Electron App');
  tray.setContextMenu(contextMenu);

  tray.on('click', () => {
    mainWindow.isVisible() ? mainWindow.hide() : mainWindow.show();
  });

  return tray;
}
```

---

## 8. macOS Application Menu

`AppMenuService` only exists on macOS.

It builds a localized native menu with:
- app menu roles like hide/unhide/services/quit
- standard file/edit/view/window menus
- help links opened through `shell.openExternal`
- an About action that sends `IpcChannel.Windows_NavigateToAbout` into the renderer

This shows an important Electron pattern: native menu actions can drive SPA navigation by emitting IPC into the existing renderer.

### Plain-Language Explanation

**"Roles" in Electron menus:** Electron provides predefined menu items called "roles." When you use a role like `role: 'quit'` or `role: 'hide'`, Electron automatically:
- Uses the correct label in the user's system language (no need to translate "Quit" or "Hide")
- Places the item in the correct position expected by the OS
- Binds the correct keyboard shortcut (Cmd+Q for Quit, Cmd+H for Hide)

Using roles is better than manually creating these items because the behavior matches what the OS expects.

**The About action pattern — why IPC instead of direct navigation:**

The About menu item is a native macOS menu item (in the 🍎 App Name menu). When clicked, it needs to navigate the React SPA to the About page. But the menu code runs in the main process — it cannot call React Router's `navigate('/about')` directly.

The pattern is:
1. User clicks "About Cherry Studio" in the native menu
2. Main process receives the click
3. Main process sends an IPC message: `mainWindow.webContents.send('navigate-to-about')`
4. Renderer process receives the message
5. React Router navigates to the About route

This pattern — native menu → IPC → SPA navigation — is useful for any scenario where OS-native UI needs to trigger changes inside the web UI.

**`shell.openExternal` for help links:** `shell.openExternal(url)` opens a URL in the user's default web browser (Chrome, Safari, Firefox), not inside the Electron app. This is appropriate for help/documentation links — users expect documentation to open in their browser.

**Blind spot — "Why does the app menu only exist on macOS?":** On macOS, the application menu is a system-wide concept — it appears at the very top of the screen, belongs to whichever app is currently active, and is always visible. On Windows and Linux, each window has its own menu bar (or no menu bar if `autoHideMenuBar` is set). Electron apps on Windows/Linux typically either show a per-window menu bar or hide it entirely and provide all navigation within the renderer UI.

### Universal Reuse — Your Own Electron Project

A macOS application menu for any Electron app:

```typescript
import { Menu, shell, app } from 'electron';

function createMacMenu(mainWindow: BrowserWindow) {
  const template: Electron.MenuItemConstructorOptions[] = [
    {
      label: app.name,
      submenu: [
        { role: 'about' },
        { type: 'separator' },
        { role: 'hide' },
        { role: 'hideOthers' },
        { role: 'unhide' },
        { type: 'separator' },
        { role: 'quit' }
      ]
    },
    {
      label: 'Edit',
      submenu: [
        { role: 'undo' },
        { role: 'redo' },
        { type: 'separator' },
        { role: 'cut' },
        { role: 'copy' },
        { role: 'paste' }
      ]
    },
    {
      label: 'Help',
      submenu: [
        {
          label: 'Documentation',
          click: () => shell.openExternal('https://myapp.com/docs')
        }
      ]
    }
  ];

  Menu.setApplicationMenu(Menu.buildFromTemplate(template));
}
```

Note: Only call this on macOS (`process.platform === 'darwin'`). On Windows/Linux, either set a per-window menu or call `Menu.setApplicationMenu(null)` to remove the default menu.

---

## 9. Guest Content And External Navigation Policy

`WindowService` also owns several guest-content rules:
- `will-navigate` blocks unexpected navigation and opens safe URLs externally
- `setWindowOpenHandler()` allows a small OAuth popup allowlist
- `http://file/...` URLs are resolved safely and checked against path traversal
- CSP and X-Frame-Options headers are stripped from responses for embedded content

That mixes shell UX with embedded-content policy, which is common in Electron apps that host external sites or mini apps.

### Plain-Language Explanation

**`will-navigate` event:** Fires when the renderer is about to navigate to a new URL (user clicks a link, JavaScript calls `window.location = ...`, etc.). The handler can inspect the URL and decide whether to allow or block the navigation.

**Why block unexpected navigation:** If your app embeds third-party web pages (mini apps), those pages might contain links to external websites. Without intervention, clicking such a link would navigate your entire app window away from your UI and into the external website. Blocking navigation and opening the link in the system browser instead keeps your app's window under your control.

**`setWindowOpenHandler()`:** Controls what happens when a web page tries to open a new window (via `window.open()` or clicking a link with `target="_blank"`). The handler:
- Checks if the URL is on the OAuth allowlist (some OAuth flows need popups)
- If allowed: creates a small Electron popup window
- If not allowed: opens the URL in the system browser

**`http://file/...` URLs:** Cherry Studio has a feature where it can load local files through a virtual `http://file/` URL scheme. The code validates these URLs to prevent "path traversal" attacks — where a malicious URL like `http://file/../../../etc/passwd` could escape the intended directory and read arbitrary files. The validation checks that the resolved path stays within the allowed directories.

**CSP and X-Frame-Options stripping:** These are HTTP response headers that websites use to control where their content can be embedded:

- **CSP (Content Security Policy):** "I only allow my page to be loaded in these specific ways and from these specific sources"
- **X-Frame-Options:** "Do not allow my page to be displayed inside an iframe or webview"

Cherry Studio strips these headers for embedded content because:
1. The content is intentionally being embedded in the app (it is a mini app the user added)
2. The original website's CSP might block loading resources inside Electron's environment
3. Without stripping, many legitimate embedded pages would fail to load or function

**Security note:** Stripping security headers should only be done for content the user explicitly chose to embed. Never strip CSP from arbitrary web pages — the headers exist for a reason.

### Universal Reuse — Your Own Electron Project

```typescript
// Block unexpected navigation, open in system browser
mainWindow.webContents.on('will-navigate', (event, url) => {
  const allowed = url.startsWith('file://') || url.startsWith(appUrl);
  if (!allowed) {
    event.preventDefault();
    shell.openExternal(url);
  }
});

// Control popup behavior
mainWindow.webContents.setWindowOpenHandler(({ url }) => {
  if (isOAuthUrl(url)) {
    return { action: 'allow' };  // Allow OAuth popup
  }
  shell.openExternal(url);
  return { action: 'deny' };    // Block, open in browser instead
});
```

---

## 10. Practical Takeaways

- Centralize `BrowserWindow` policy in a service
- Treat tray, window close, and quit as distinct UX states
- Persist geometry per window type
- Expect platform-specific hacks for focus, fullscreen, and title bars
- Use native menus and tray items as IPC entry points back into the renderer

---

## 11. How To Apply This Knowledge In Development

Use this document when you are deciding how a new desktop-facing feature should appear to the user.

Choose the right integration point:
- Use `WindowService` when the feature changes window creation, reuse, focus, geometry, or native shell policy.
- Use `TrayService` when the feature must remain available while the main window is hidden or closed to tray.
- Use `AppMenuService` for macOS-native entry points that should feel like part of the desktop app, not just the web UI.
- Use renderer navigation only after the native entry point has already routed into the correct window.

Practical usage pattern:
1. Define whether the new behavior belongs to the main window, a utility window, tray, or menu.
2. Decide whether it should reuse an existing window or create a distinct renderer entry.
3. Check platform-specific expectations before copying behavior across Windows, macOS, and Linux.
4. Keep shell policy in main process and UI state in renderer, connected by IPC.

Common mistakes this avoids:
- Attaching window listeners after a reused window is already alive.
- Treating close, hide, minimize, and quit as the same action.
- Adding renderer-only code for behavior that must still work when the window is hidden.
- Forgetting macOS-specific menu and dock expectations.

---

## 12. Typical Application Scenarios

- Add a new always-on-top utility window for a focused workflow.
- Change what happens when the user closes the main window while tray mode is enabled.
- Add a tray action for toggling a background feature without restoring the main window.
- Add a macOS menu item that opens a settings route or About screen in the current renderer.

---

## 13. Relationship To The Other Electron Records

- This document builds on `electron_main_process_lifecycle.md` because window and tray creation only make sense after the app boot phase is correct.
- It works closely with `electron_webview_session_management.md` because embedded guest content inherits window-level popup, navigation, and focus policy.
- It supports `electron_protocol_oauth_and_deep_linking.md` because a deep link usually needs to show, focus, or navigate an existing window.
- It connects to `electron_build_packaging_and_update_pipeline.md` when you add a new renderer entry file for another native window.
