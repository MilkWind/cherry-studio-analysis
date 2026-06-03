# Electron Custom Protocol, OAuth Callback, And Deep-Link Flow

Cherry Studio uses the custom protocol `cherrystudio://` to receive OAuth callbacks and other in-app actions from the operating system.

Primary file:
- `src/main/services/ProtocolClient.ts`

Related entry points:
- `src/main/index.ts`
- `src/preload/index.ts`

## 1. Registering The Protocol Client

The protocol name is:

```text
cherrystudio
```

Registration happens with `app.setAsDefaultProtocolClient(...)`.

Cherry Studio handles two cases:
- dev mode with `process.defaultApp`, passing the app entry script path
- normal packaged mode, registering the app directly

This distinction matters because Electron dev launches usually run through the Node/Electron executable plus your project entry, while packaged apps have a single executable.

## 2. Where Deep Links Arrive

Electron can deliver protocol URLs from multiple places:

- `app.on('open-url', ...)` on macOS when the app is already running
- `process.argv` when the app is launched by a protocol URL
- `app.on('second-instance', ...)` when a new launch is redirected to the already-running instance

Cherry Studio supports all three, which is the correct pattern for a cross-platform deep-link-capable Electron app.

## 3. Why Single-Instance Lock Matters

Because Cherry Studio uses `requestSingleInstanceLock()`, a second launch is not allowed to keep running as a separate app instance.

Instead:
- the first instance is brought to the foreground
- the deep-link URL from the second launch is parsed and handled there

Without this, protocol callbacks could open duplicate app processes or lose the callback payload.

## 4. Main-Process URL Routing

`handleProtocolUrl(url)` parses the incoming URL with the standard `URL` API and then routes by hostname:

- `cherrystudio://mcp/...` -> MCP install handling
- `cherrystudio://providers/...` -> provider/OAuth handling
- everything else -> generic payload broadcast to the renderer

The generic path sends:

```ts
mainWindow.webContents.send('protocol-data', {
  url,
  params: Object.fromEntries(params.entries())
})
```

So the renderer does not need direct access to OS launch arguments.

## 5. Renderer Consumption Through Preload

The preload layer exposes:

```ts
window.api.protocol.onReceiveData(callback)
```

That gives the renderer a clean subscription-based interface for protocol payloads. This is safer and cleaner than letting renderer code read from Node APIs directly.

## 6. Why This Matters For OAuth

The login flows described elsewhere rely on redirect URIs such as:

```text
cherrystudio://oauth/callback
```

The provider redirects the system browser to that URI, the OS reopens Cherry Studio, and the main process forwards the parsed callback data to whichever feature is waiting for it.

In Electron terms, the custom protocol acts as the bridge between external browser authentication and the running desktop app.

## 7. Linux AppImage Deep-Link Support

Packaged Linux AppImages do not automatically gain protocol registration in every environment. Cherry Studio handles this with `setupAppImageDeepLink()`.

When running as an AppImage on Linux it:
- finds the current executable path
- creates `~/.local/share/applications/cherrystudio-url-handler.desktop`
- writes an `Exec=... %U` line
- declares `MimeType=x-scheme-handler/cherrystudio;`
- runs `update-desktop-database`

This is a project-specific example of Electron packaging support that must be added outside the normal protocol registration API.

## 8. Practical Takeaways

- Register protocol handlers differently in dev and packaged modes
- Support `open-url`, startup args, and `second-instance`
- Keep protocol parsing in main process
- Forward parsed data to renderer through preload IPC
- Expect extra Linux work for AppImage deep-link support
