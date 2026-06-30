# window.api Reference

## Overview

**window.api** is the main bridge between the Electron **renderer process** (React frontend) and the **main process** (Node.js backend) in Cherry Studio. It is defined in `src/preload/index.ts` and exposed via:

```typescript
contextBridge.exposeInMainWorld('api', api)
// Fallback for non-isolated mode:
window.api = api
```

Most methods use `ipcRenderer.invoke()` (promise-based bidirectional IPC). A helper `tracedInvoke()` adds OpenTelemetry span context where needed (`mcp.refreshTools`/`callTool`). A few members use `ipcRenderer.on()` to subscribe to push events and return an unsubscribe function.

**Key files:**
- Definition: `src/preload/index.ts` (the `api` object literal, lines 98–799)
- Type declaration: `src/preload/preload.d.ts` (`WindowApiType = typeof api`)
- IPC channels: `packages/shared/IpcChannel.ts`

**Scope of this reference:** Only APIs that are **actually called** from the Cherry Studio renderer (`src/renderer/**`) are listed below. The previous version of this document contained many stale entries (renamed, removed, or never-wired APIs); it has been reconciled against the current `api` object and against real call sites. Definitions that exist in the preload but are not invoked anywhere in the renderer are intentionally omitted.

There are **~429 production references** to `window.api` across the renderer codebase (plus ~78 in tests).

---

## Architecture

**Exposure mechanism:** Uses `contextBridge.exposeInMainWorld()` (context-isolated mode) with a fallback for non-isolated mode.

**IPC Communication:** Most methods use `ipcRenderer.invoke()` (promise-based bidirectional IPC). `mcp.refreshTools`/`mcp.callTool` go through the `tracedInvoke()` helper to attach an OpenTelemetry SpanContext.

**Event Listeners:** Several namespaces subscribe to main-process push events via `ipcRenderer.on()` and return a **cleanup function** (e.g., `tree.onMutation`, `mcp.onServerLog`, `protocol.onReceiveData`, `cache.onSync`, `preference.onChanged`, `window.api.<channel>.on*`). Always call the returned unsubscribe function on unmount.

**Non-invoke members (notable):**
- `file.getPathForFile(file)` — direct `webUtils.getPathForFile()` call (synchronous, no IPC).
- `shell.openExternal(url)` — direct `electron.shell.openExternal()` after a protocol allowlist check (`http:`, `https:`, `mailto:`, `obsidian:`). Rejects other/invalid URLs.
- `cache.broadcastSync(message)` — fire-and-forget `ipcRenderer.send()` (not invoke).

**Raw-string channels** (not via the `IpcChannel` enum): `selectionMenu` is defined but unused; `protocol.onReceiveData` listens on the literal `'protocol-data'` channel.

**Migrated off `window.api` (now reached via the generic RPC facade `window.api.ipcApi.request(route)` / `ipcApi.on(event)`):** Binary manager (`binary.*`), and all AI capability IPC (`ai.*` — model ops, streaming chat, agent-session warm-up, tool approval, agent run-task). Knowledge-base and memory direct namespaces were removed; RAG/AI-memory is no longer exposed as a named `window.api.*` surface.

---

## Complete API Reference

### A. Top-Level Functions (App/System Core)

| Function | Signature | Purpose |
|---|---|---|
| `getAppInfo()` | `() => Promise<AppInfo>` | Get application info (version, paths, etc.) |
| `reload()` | `() => Promise<void>` | Reload the main window |
| `setEnableSpellCheck(isEnable)` | `(boolean) => Promise<void>` | Enable/disable spell check |
| `handleZoomFactor(delta, reset?)` | `(number, boolean?) => Promise<void>` | Adjust zoom factor |
| `select(options)` | `(Electron.OpenDialogOptions) => Promise<?>` | Open file/folder selection dialog |
| `hasWritePermission(path)` | `(string) => Promise<boolean>` | Check write permissions for a path |
| `resolvePath(path)` | `(string) => Promise<string>` | Resolve a file path |
| `isPathInside(childPath, parentPath)` | `(string, string) => Promise<boolean>` | Check if child path is inside parent |
| `setAppDataPath(path)` | `(string) => Promise<void>` | Set application data directory |
| `getDataPathFromArgs()` | `() => Promise<string>` | Get data path from CLI args |
| `copy(oldPath, newPath, occupiedDirs?)` | `(string, string, string[]?) => Promise<?>` | Copy files/directories |
| `flushAppData()` | `() => Promise<void>` | Flush application data |
| `isNotEmptyDir(path)` | `(string) => Promise<boolean>` | Check if directory is non-empty |
| `resetData()` | `() => Promise<void>` | Reset all application data |
| `openWebsite(url)` | `(string) => Promise<void>` | Open URL in external browser |
| `getCacheSize()` | `() => Promise<number>` | Get cache size |
| `clearCache()` | `() => Promise<void>` | Clear application cache |
| `getSystemFonts()` | `() => Promise<string[]>` | Get installed system fonts |
| `getIpCountry()` | `() => Promise<string>` | Get user's IP country |
| `quoteToMainWindow(text)` | `(string) => Promise<void>` | Quote selected text to main window |
| `openPath(path)` | `(string) => Promise<void>` | Open a file/folder at path |

#### Binary Management

| Function | Signature | Purpose |
|---|---|---|
| `isBinaryExist(name)` | `(string) => Promise<boolean>` | Check if a bundled binary exists |
| `installOvmsBinary()` | `() => Promise<void>` | Install the OVMS (OpenVINO) binary |

### B. `window.api.application` (App Lifecycle)

Replaces the legacy top-level `quit` / `relaunchApp` / `setStopQuitApp`.

| Function | Signature | Purpose |
|---|---|---|
| `preventQuit(reason)` | `(string) => Promise<string>` | Prevent app quit, returning a hold id |
| `allowQuit(holdId)` | `(string) => Promise<void>` | Release a quit hold by id |
| `relaunch(options?)` | `(Electron.RelaunchOptions?) => Promise<void>` | Relaunch the application |

### C. `window.api.mac` (macOS-Specific)

| Function | Signature | Purpose |
|---|---|---|
| `isProcessTrusted()` | `() => Promise<boolean>` | Check macOS accessibility trust |
| `requestProcessTrust()` | `() => Promise<boolean>` | Request macOS accessibility trust |

### D. `window.api.notification`

| Function | Signature | Purpose |
|---|---|---|
| `send(notification)` | `(Notification) => Promise<void>` | Send a system notification |

### E. `window.api.system` (System Information)

| Function | Signature | Purpose |
|---|---|---|
| `getDeviceType()` | `() => Promise<string>` | Get OS device type |
| `getHostname()` | `() => Promise<string>` | Get system hostname |

### F. `window.api.devTools`

| Function | Signature | Purpose |
|---|---|---|
| `toggle()` | `() => Promise<void>` | Toggle DevTools |

### G. `window.api.zip`

| Function | Signature | Purpose |
|---|---|---|
| `decompress(text)` | `(Buffer) => Promise<string>` | Decompress data |

### H. `window.api.backup` (Backup/Restore)

| Function | Signature | Purpose |
|---|---|---|
| `restore(path)` | `(string) => Promise<void>` | Restore from backup file |
| `backup(fileName, destinationPath, skipBackupFile)` | `(string, string, boolean) => Promise<void>` | Create backup |
| `backupToWebdav(webdavConfig)` | `(WebDavConfig) => Promise<boolean>` | Backup to WebDAV |
| `restoreFromWebdav(webdavConfig)` | `(WebDavConfig) => Promise<string>` | Restore from WebDAV |
| `listWebdavFiles(webdavConfig)` | `(WebDavConfig) => Promise<File[]>` | List WebDAV backup files |
| `checkWebdavConnection(webdavConfig)` | `(WebDavConfig) => Promise<boolean>` | Check WebDAV connection |
| `createDirectory(webdavConfig, path, options?)` | `(WebDavConfig, string, CreateDirectoryOptions?) => Promise<void>` | Create WebDAV directory |
| `deleteWebdavFile(fileName, webdavConfig)` | `(string, WebDavConfig) => Promise<void>` | Delete WebDAV file |
| `backupToLocalDir(fileName, localConfig)` | `(string, {localBackupDir?,skipBackupFile?}) => Promise<void>` | Backup to local directory |
| `restoreFromLocalBackup(fileName, localBackupDir?)` | `(string, string?) => Promise<string>` | Restore from local backup |
| `listLocalBackupFiles(localBackupDir?)` | `(string?) => Promise<File[]>` | List local backup files |
| `deleteLocalBackupFile(fileName, localBackupDir?)` | `(string, string?) => Promise<void>` | Delete local backup file |
| `backupToS3(s3Config)` | `(S3Config) => Promise<void>` | Backup to S3 |
| `restoreFromS3(s3Config)` | `(S3Config) => Promise<void>` | Restore from S3 |
| `listS3Files(s3Config)` | `(S3Config) => Promise<File[]>` | List S3 backup files |
| `deleteS3File(fileName, s3Config)` | `(string, S3Config) => Promise<void>` | Delete S3 file |
| `createLanTransferBackup(data, destinationPath?)` | `(string, string?) => Promise<string>` | Create LAN transfer backup |
| `deleteLanTransferBackup(filePath)` | `(string) => Promise<boolean>` | Delete LAN transfer backup |

### I. `window.api.file` (File Operations)

The most heavily-used namespace. Handles file I/O, metadata, dialogs, and the file-entry abstraction.

| Function | Signature | Purpose |
|---|---|---|
| `select(options?)` | `(OpenDialogOptions?) => Promise<FileMetadata[]\|null>` | Select files via dialog |
| `createInternalEntry(params)` | `(CreateInternalEntryIpcParams) => Promise<FileEntry>` | Create an internal (managed) file entry |
| `getPhysicalPath(params)` | `(GetPhysicalPathIpcParams) => Promise<FilePath>` | Resolve the on-disk path for a file handle |
| `getMetadata(handle)` | `(FileHandle) => Promise<PhysicalFileMetadata>` | Get physical file metadata for a handle |
| `deleteExternalFile(filePath)` | `(string) => Promise<void>` | Delete an external file |
| `deleteExternalDir(dirPath)` | `(string) => Promise<void>` | Delete an external directory |
| `move(path, newPath)` | `(string, string) => Promise<void>` | Move a file |
| `moveDir(dirPath, newDirPath)` | `(string, string) => Promise<void>` | Move a directory |
| `rename(path, newName)` | `(string, string) => Promise<void>` | Rename a file |
| `renameDir(dirPath, newName)` | `(string, string) => Promise<void>` | Rename a directory |
| `readExternal(filePath, detectEncoding?)` | `(string, boolean?) => Promise<string>` | Read an external file's content |
| `get(filePath)` | `(string) => Promise<FileMetadata\|null>` | Get file metadata by path |
| `createTempFile(fileName)` | `(string) => Promise<string>` | Create a temp file |
| `mkdir(dirPath)` | `(string) => Promise<void>` | Create a directory |
| `write(filePath, data)` | `(string, Uint8Array\|string) => Promise<void>` | Write data to a file |
| `open(options?)` | `(OpenDialogOptions?) => Promise<?>` | Open file dialog |
| `openPath(path)` | `(string) => Promise<void>` | Open a file at path |
| `save(path, content, options?)` | `(string, string\|ArrayBufferView, any?) => Promise<string\|null>` | Save a file |
| `selectFolder(options?)` | `(OpenDialogOptions?) => Promise<string\|null>` | Select a folder via dialog |
| `saveImage(name, data)` | `(string, string) => Promise<boolean>` | Save an image file |
| `binaryImage(fileId)` | `(string) => Promise<Buffer>` | Get image as binary |
| `savePastedImage(imageData, extension?)` | `(Uint8Array, string?) => Promise<?>` | Save pasted image data |
| `getPathForFile(file)` | `(File) => string` | Get path for a File object (synchronous, no IPC) |
| `isTextFile(filePath)` | `(string) => Promise<boolean>` | Check if a file is text |
| `isDirectory(filePath)` | `(string) => Promise<boolean>` | Check if a path is a directory |
| `listDirectory(dirPath, options?)` | `(string, DirectoryListOptions?) => Promise<?>` | List directory contents |
| `checkFileName(dirPath, fileName, isFile)` | `(string, string, boolean) => Promise<{safeName}>` | Validate/generate a safe file or dir name |
| `validateNotesDirectory(dirPath)` | `(string) => Promise<void>` | Validate a notes directory |
| `batchUploadMarkdown(filePaths, targetPath)` | `(string[], string) => Promise<void>` | Batch upload markdown files |
| `showInFolder(path)` | `(string) => Promise<void>` | Reveal a file in the system file manager |

> The legacy file-watcher methods (`startFileWatcher` / `stopFileWatcher` / `pauseFileWatcher` / `resumeFileWatcher` / `onFileChange`) and `getDirectoryStructure` were removed. Directory watching now goes through `window.api.tree` (DirectoryTreeBuilder).

### J. `window.api.fs` (Low-Level File System)

| Function | Signature | Purpose |
|---|---|---|
| `read(pathOrUrl, encoding?)` | `(string, BufferEncoding?) => Promise<Buffer\|string>` | Read a file from filesystem or URL |
| `readText(pathOrUrl)` | `(string) => Promise<string>` | Read a file as text |

### K. `window.api.tree` (Directory Tree Builder)

Live directory watching/mutation bridge (replaces the old `file.*FileWatcher` APIs).

| Function | Signature | Purpose |
|---|---|---|
| `create(rootPath, options?)` | `(string, DirectoryTreeOptions?) => Promise<CreateTreeIpcResult>` | Create a watched directory tree |
| `dispose(treeId)` | `(string) => Promise<void>` | Dispose a tree and stop watching |
| `rename(treeId, oldPath, newPath)` | `(string, string, string) => Promise<boolean>` | Rename within a watched tree |
| `onMutation(callback)` | `((TreeMutationPushPayload) => void) => () => void` | Subscribe to tree mutation push events |

### L. `window.api.export`

| Function | Signature | Purpose |
|---|---|---|
| `toWord(markdown, fileName)` | `(string, string) => Promise<void>` | Export markdown to a Word document |

### M. `window.api.obsidian`

| Function | Signature | Purpose |
|---|---|---|
| `getVaults()` | `() => Promise<?>` | Get Obsidian vaults |
| `getFiles(vaultName)` | `(string) => Promise<?>` | Get files in an Obsidian vault |

### N. `window.api.window` (Window Management)

| Function | Signature | Purpose |
|---|---|---|
| `setMinimumSize(width, height)` | `(number, number) => Promise<void>` | Set minimum window size |
| `resetMinimumSize()` | `() => Promise<void>` | Reset minimum window size |
| `setAlwaysOnTop(pinned)` | `(boolean) => Promise<boolean>` | Pin/unpin the current sub-window (always-on-top) |

### O. `window.api.command` (Native Popup Menu)

| Function | Signature | Purpose |
|---|---|---|
| `showNativePopupMenu(model, anchor?)` | `(NativePopupMenuModel<CommandId>, MenuAnchor?) => Promise<NativePopupMenuResult<CommandId>\|undefined>` | Show a native context popup menu |

### P. `window.api.ovms` (OpenVINO Model Server)

| Function | Signature | Purpose |
|---|---|---|
| `isSupported()` | `() => Promise<boolean>` | Check OVMS support |
| `addModel(modelName, modelId, modelSource, task)` | `(string, string, string, string) => Promise<?>` | Add an OVMS model |
| `stopAddModel()` | `() => Promise<void>` | Cancel adding a model |
| `getStatus()` | `() => Promise<?>` | Get OVMS status |
| `runOvms()` | `() => Promise<void>` | Start the OVMS server |
| `stopOvms()` | `() => Promise<void>` | Stop the OVMS server |

### Q. `window.api.config`

| Function | Signature | Purpose |
|---|---|---|
| `set(key, value, isNotify?)` | `(string, any, boolean?) => Promise<void>` | Set a config value |

### R. `window.api.quickAssistant` (Quick Assistant Window)

Replaces the legacy `miniWindow` namespace.

| Function | Signature | Purpose |
|---|---|---|
| `hide()` | `() => Promise<void>` | Hide the quick assistant window |
| `close()` | `() => Promise<void>` | Close the quick assistant window |
| `setPin(isPinned)` | `(boolean) => Promise<void>` | Pin/unpin the quick assistant window |

### S. `window.api.aes` (Encryption)

| Function | Signature | Purpose |
|---|---|---|
| `decrypt(encryptedData, iv, secretKey)` | `(string, string, string) => Promise<string>` | AES decrypt data |

### T. `window.api.mcp` (Model Context Protocol)

| Function | Signature | Purpose |
|---|---|---|
| `removeServer(serverId)` | `(string) => Promise<void>` | Remove an MCP server |
| `restartServer(serverId)` | `(string) => Promise<void>` | Restart an MCP server |
| `stopServer(serverId)` | `(string) => Promise<void>` | Stop an MCP server |
| `refreshTools(serverId, context?)` | `(string, SpanContext?) => Promise<?>` | Refresh/list an MCP server's tools (traced) |
| `listPrompts(serverId)` | `(string) => Promise<?>` | List an MCP server's prompts |
| `listResources(serverId)` | `(string) => Promise<?>` | List an MCP server's resources |
| `checkMcpConnectivity(serverId)` | `(string) => Promise<?>` | Check connectivity to an MCP server |
| `uploadDxt(file)` | `(File) => Promise<?>` | Upload a `.dxt` MCP package |
| `uploadMcpb(file)` | `(File) => Promise<?>` | Upload a `.mcpb` MCP package |
| `abortTool(callId)` | `(string) => Promise<void>` | Abort an in-flight tool call |
| `getServerVersion(serverId)` | `(string) => Promise<string\|null>` | Get an MCP server's version |
| `getServerLogs(serverId)` | `(string) => Promise<MCPServerLogEntry[]>` | Get an MCP server's logs |
| `onServerLog(callback)` | `((MCPServerLogEntry) => void) => () => void` | Subscribe to server log events |

### U. `window.api.shell`

| Function | Signature | Purpose |
|---|---|---|
| `openExternal(url, options?)` | `(string, OpenExternalOptions?) => Promise<void>` | Open URL in system browser (protocol-restricted: `http`/`https`/`mailto`/`obsidian`) |

### V. `window.api.copilot` (GitHub Copilot Auth)

| Function | Signature | Purpose |
|---|---|---|
| `getAuthMessage(headers?)` | `(Record<string,string>?) => Promise<?>` | Get Copilot device-flow auth message |
| `getCopilotToken(device_code, headers?)` | `(string, Record<string,string>?) => Promise<?>` | Exchange device code for Copilot token |
| `saveCopilotToken(access_token)` | `(string) => Promise<void>` | Save Copilot token |
| `getToken(headers?)` | `(Record<string,string>?) => Promise<string>` | Get stored Copilot token |
| `logout()` | `() => Promise<void>` | Logout from Copilot |
| `getUser(token)` | `(string) => Promise<?>` | Get Copilot user info |

### W. `window.api.cherryin` (CherryIN OAuth)

| Function | Signature | Purpose |
|---|---|---|
| `getBalance(apiHost)` | `(string) => Promise<?>` | Get CherryIN balance |
| `logout(apiHost)` | `(string) => Promise<void>` | CherryIN logout |
| `startOAuthFlow(oauthServer, apiHost?)` | `(string, string?) => Promise<{authUrl,state}>` | Start CherryIN OAuth flow |
| `onOAuthResult(callback)` | `(({state,apiKeys}\|{state,error}) => void) => () => void` | Subscribe to OAuth result events |

### X. `window.api.protocol` (Custom Protocol Handler)

| Function | Signature | Purpose |
|---|---|---|
| `onReceiveData(callback)` | `(({url, params}) => void) => () => void` | Listen for custom-protocol data (OAuth callbacks) |

### Y. `window.api.externalApps`

| Function | Signature | Purpose |
|---|---|---|
| `detectInstalled()` | `() => Promise<ExternalAppInfo[]>` | Detect installed external apps |

### Z. `window.api.nutstore`

| Function | Signature | Purpose |
|---|---|---|
| `getSSOUrl()` | `() => Promise<string>` | Get Nutstore SSO URL |
| `decryptToken(token)` | `(string) => Promise<?>` | Decrypt Nutstore token |
| `getDirectoryContents(token, path)` | `(string, string) => Promise<?>` | List Nutstore directory contents |

### AA. `window.api.searchService`

| Function | Signature | Purpose |
|---|---|---|
| `openUrlInSearchWindow(uid, url)` | `(string, string) => Promise<void>` | Open a URL inside a search window |

### AB. `window.api.webview`

| Function | Signature | Purpose |
|---|---|---|
| `setOpenLinkExternal(webviewId, isExternal)` | `(number, boolean) => Promise<void>` | Set external link handling |
| `setSpellCheckEnabled(webviewId, isEnable)` | `(number, boolean) => Promise<void>` | Enable spell check in a webview |
| `printToPDF(webviewId)` | `(number) => Promise<void>` | Print a webview to PDF |
| `saveAsHTML(webviewId)` | `(number) => Promise<void>` | Save a webview as HTML |
| `onFindShortcut(callback)` | `((WebviewKeyEvent) => void) => () => void` | Listen for find (Ctrl+F) shortcuts in a webview |

### AC. `window.api.settings`

Interim home for opening the Settings window (the underlying `SettingsWindow_Open` IPC is legacy and slated for migration onto IpcApi).

| Function | Signature | Purpose |
|---|---|---|
| `openSettings(path?)` | `(SettingsPath?) => Promise<string>` | Open the settings window at a path (default `/settings/provider`) |

### AD. `window.api.wechat` (WeChat Channel)

| Function | Signature | Purpose |
|---|---|---|
| `onQrLogin(callback)` | `(({channelId,agentId,url,status,userId?}) => void) => () => void` | Listen for WeChat QR login events |
| `hasCredentials(channelId)` | `(string) => Promise<{exists,userId?}>` | Check WeChat credentials |

### AE. `window.api.feishu` (Feishu/Lark Channel)

| Function | Signature | Purpose |
|---|---|---|
| `onQrLogin(callback)` | `(({channelId,agentId,url,status,appId?,appSecret?}) => void) => () => void` | Listen for Feishu QR login events |

### AF. `window.api.channel` (Channel Management)

| Function | Signature | Purpose |
|---|---|---|
| `onLog(callback)` | `(({timestamp,level,message,channelId}) => void) => () => void` | Listen for channel logs |
| `onStatusChange(callback)` | `(({channelId,connected,error?}) => void) => () => void` | Listen for channel status changes |
| `getLogs(channelId)` | `(string) => Promise<Array<{timestamp,level,message,channelId}>>` | Get a channel's logs |
| `getStatuses()` | `() => Promise<Array<{channelId,connected,error?}>>` | Get all channel statuses |

### AG. `window.api.trace` (OpenTelemetry Tracing)

| Function | Signature | Purpose |
|---|---|---|
| `getData(topicId, traceId)` | `(string, string) => Promise<any[]>` | Get trace data for a topic/trace |
| `cleanLocalData()` | `() => Promise<void>` | Clean local trace data |

### AH. `window.api.codeCli` (CLI Code Tools)

Replaces the legacy `codeTools` namespace.

| Function | Signature | Purpose |
|---|---|---|
| `run(cliTool, model, directory, env, options?)` | `(string, string, string, Record<string,string>, {autoUpdateToLatest?,terminal?}?) => Promise<CodeToolsRunResult>` | Run a CLI code tool (e.g. Claude Code) |
| `getAvailableTerminals()` | `() => Promise<TerminalConfig[]>` | Get available terminals |
| `setCustomTerminalPath(terminalId, path)` | `(string, string) => Promise<void>` | Set a custom terminal path |

### AI. `window.api.shortcut`

Note: singular `shortcut`, not the legacy plural `shortcuts`.

| Function | Signature | Purpose |
|---|---|---|
| `onRegistrationConflict(callback)` | `((ShortcutRegistrationConflictPayload) => void) => () => void` | Listen for accelerator registration conflicts |

### AJ. `window.api.cache` (Multi-Window Cache Sync)

| Function | Signature | Purpose |
|---|---|---|
| `broadcastSync(message)` | `(CacheSyncMessage) => void` | Broadcast a cache sync message to other windows (fire-and-forget `send`) |
| `onSync(callback)` | `((CacheSyncMessage) => void) => () => void` | Listen for cache sync messages from other windows |
| `getAllShared()` | `() => Promise<Record<string, CacheEntry>>` | Get all shared cache entries (initialization sync) |

### AK. `window.api.storageMonitor` (Disk Space Watcher)

| Function | Signature | Purpose |
|---|---|---|
| `getHealth()` | `() => Promise<StorageHealth>` | Get current disk-space health |
| `onHealthChange(callback)` | `((StorageHealth) => void) => () => void` | Subscribe to health transitions pushed from main |

### AL. `window.api.preference` (Unified Preferences)

Marked `// DO NOT MODIFY THIS SECTION` in the preload.

| Function | Signature | Purpose |
|---|---|---|
| `get(key)` | `<K extends UnifiedPreferenceKeyType>(K) => Promise<UnifiedPreferenceType[K]>` | Get a preference value |
| `set(key, value)` | `<K extends UnifiedPreferenceKeyType>(K, UnifiedPreferenceType[K]) => Promise<void>` | Set a preference value |
| `getMultipleRaw(keys)` | `<K extends UnifiedPreferenceKeyType>(K[]) => Promise<UnifiedPreferenceMultipleResultType<K>>` | Get multiple preferences |
| `setMultiple(updates)` | `(Partial<UnifiedPreferenceType>) => Promise<void>` | Set multiple preferences |
| `getAll()` | `() => Promise<UnifiedPreferenceType>` | Get all preferences |
| `subscribe(keys)` | `(UnifiedPreferenceKeyType[]) => Promise<void>` | Subscribe to preference changes |
| `onChanged(callback)` | `((key, value) => void) => () => void` | Listen for preference changes |

### AM. `window.api.dataApi` (Data API Bridge)

| Function | Signature | Purpose |
|---|---|---|
| `request(req)` | `(any) => Promise<?>` | Issue a Data API request |
| `subscribe(path, callback)` | `(string, (data, event) => void) => () => void` | Subscribe to a Data API stream for a path |

### AN. `window.api.ipcApi` (Generic RPC Facade)

Imported from `src/preload/ipc.ts`. The typed facade consumed by the renderer lives in `src/renderer/ipc`. This is the forwarder for capability surfaces migrated off named `window.api.*` methods (e.g. `binary.*`, `ai.*`).

| Function | Signature | Purpose |
|---|---|---|
| `request(route, input?, meta?)` | `(string, unknown?, unknown?) => Promise<unknown>` | Generic RPC invoke by route name |
| `on(event, callback)` | `(string, (payload) => void) => () => void` | Subscribe to a named event (demultiplexed) |

### AO. `window.api.topic`

| Function | Signature | Purpose |
|---|---|---|
| `onAutoRenamed(callback)` | `(({topicId}) => void) => () => void` | Listen for topic auto-rename events |

### AP. `window.api.agentSession`

| Function | Signature | Purpose |
|---|---|---|
| `onAutoRenamed(callback)` | `(({sessionId}) => void) => () => void` | Listen for agent-session auto-rename events |

### AQ. `window.api.translate`

| Function | Signature | Purpose |
|---|---|---|
| `open(req)` | `({streamId,text,targetLangCode,messageId?,sourceLangCode?}) => Promise<{streamId}>` | Open a translation stream |

### AR. `window.api.apiGateway` (API Gateway Service)

Replaces the legacy `apiServer` namespace.

| Function | Signature | Purpose |
|---|---|---|
| `start()` | `() => Promise<ApiGatewayStatusResult>` | Start the API gateway |
| `restart()` | `() => Promise<ApiGatewayStatusResult>` | Restart the API gateway |
| `stop()` | `() => Promise<ApiGatewayStatusResult>` | Stop the API gateway |

### AS. `window.api.skill` (Skills)

| Function | Signature | Purpose |
|---|---|---|
| `install(options)` | `(SkillInstallOptions) => Promise<SkillResult<InstalledSkill>>` | Install a skill |
| `uninstall(skillId)` | `(string) => Promise<SkillResult<void>>` | Uninstall a skill |
| `toggle(options)` | `(SkillToggleOptions) => Promise<SkillResult<InstalledSkill\|null>>` | Toggle a skill on/off |
| `installFromZip(options)` | `(SkillInstallFromZipOptions) => Promise<SkillResult<InstalledSkill>>` | Install a skill from a ZIP |
| `installFromDirectory(options)` | `(SkillInstallFromDirectoryOptions) => Promise<SkillResult<InstalledSkill>>` | Install a skill from a directory |
| `listLocal(workdir)` | `(string) => Promise<SkillResult<LocalSkill[]>>` | List local skills in a working directory |

### AT. `window.api.lanTransfer` (LAN File Transfer)

Replaces the legacy `localTransfer` namespace.

| Function | Signature | Purpose |
|---|---|---|
| `startScan()` | `() => Promise<LanTransferState>` | Start scanning for peers |
| `stopScan()` | `() => Promise<LanTransferState>` | Stop scanning for peers |
| `connect(payload)` | `(LanTransferConnectPayload) => Promise<LanHandshakeAckMessage>` | Connect to a peer |
| `disconnect()` | `() => Promise<void>` | Disconnect from a peer |
| `onServicesUpdated(callback)` | `((LanTransferState) => void) => () => void` | Listen for service/peer updates |
| `onClientEvent(callback)` | `((LanClientEvent) => void) => () => void` | Listen for client events |
| `sendFile(filePath)` | `(string) => Promise<LanFileCompleteMessage>` | Send a file to the connected peer |
| `cancelTransfer()` | `() => Promise<void>` | Cancel an in-progress file transfer |

### AU. `window.api.openclaw` (OpenClaw Integration)

| Function | Signature | Purpose |
|---|---|---|
| `checkInstalled()` | `() => Promise<{installed,path?,needsMigration}>` | Check if OpenClaw is installed |
| `install()` | `() => Promise<OperationResult>` | Install OpenClaw |
| `uninstall()` | `() => Promise<OperationResult>` | Uninstall OpenClaw |
| `startGateway(port?)` | `(number?) => Promise<OperationResult>` | Start the OpenClaw gateway |
| `stopGateway()` | `() => Promise<OperationResult>` | Stop the OpenClaw gateway |
| `getStatus()` | `() => Promise<{status,port}>` | Get OpenClaw gateway status |
| `getDashboardUrl()` | `() => Promise<string>` | Get the OpenClaw dashboard URL |
| `syncConfig(uniqueModelId)` | `(string) => Promise<OperationResult>` | Sync a model config to OpenClaw |
| `checkUpdate()` | `() => Promise<{hasUpdate,currentVersion,latestVersion,message?}>` | Check for OpenClaw updates |
| `performUpdate()` | `() => Promise<OperationResult>` | Perform an OpenClaw update |

---

## Usage Summary by Namespace

Approximate production call-site counts (renderer only):

| Namespace | Approx. Uses | Description |
|---|---|---|
| `window.api.file.*` | ~150 | File I/O, metadata, dialogs, file-entry abstraction |
| `window.api.backup.*` | ~40 | Backup/restore (WebDAV, S3, local, LAN) |
| `window.api.system.*` | ~20 | Device type / hostname |
| `window.api.mcp.*` | ~20 | MCP server/tool management |
| `window.api.openclaw.*` | ~12 | OpenClaw gateway |
| `window.api.application.*` | ~13 | App lifecycle (relaunch, quit holds) |
| `window.api.preference.*` | ~10 | Unified preferences |
| `window.api.copilot.*` | ~6 | GitHub Copilot auth |
| `window.api.ovms.*` | ~11 | OpenVINO model server |
| `window.api.webview.*` | ~8 | Embedded webview control |
| `window.api.lanTransfer.*` | ~8 | LAN peer file transfer |

**Key files that use window.api most:**
- `src/renderer/src/services/BackupService.ts` — backup/restore, file/path helpers
- `src/renderer/src/services/NotesService.ts` — notes file operations
- `src/renderer/src/pages/notes/NotesPage.tsx` — file/tree operations
- `src/renderer/src/data/PreferenceService.ts` — preference bridge
- `src/renderer/src/settings/DataSettings/BasicDataSettings.tsx` — app data / cache / path helpers

---

## Removed / Out-of-Scope (previously listed)

The following were present in earlier versions of this document but are **not** part of the current `window.api` surface (renamed, removed, commented-out in the preload, or never invoked from the renderer). They are intentionally omitted above.

- **Entire namespaces removed/renamed:** `knowledgeBase`, `memory`, `fileService`, `selection`, `selectionMenu`, `storeSync`, `agentTools`, `agentSessionStream`, `anthropic_oauth`, `ocr`, `windowControls`, `pdf` (all unused), `vertexAI` (unused), `python` (unused), `analytics` (unused), `cherryai` (unused).
  - `miniWindow` → renamed to `quickAssistant`
  - `shortcuts` → renamed to `shortcut`
  - `codeTools` → renamed to `codeCli`
  - `localTransfer` → renamed to `lanTransfer`
  - `apiServer` → renamed to `apiGateway`
- **Top-level functions removed/renamed/commented-out:** `getDiskInfo`, `quit`, `setProxy`, `checkForUpdate`, `quitAndInstall`, `setLanguage`, `setSpellCheckLanguages`, `setLaunchOnBoot`, `setLaunchToTray`, `setTray`, `setTrayOnClose`, `setTestPlan`, `setTestChannel`, `setTheme`, `setAutoUpdate`, `setStopQuitApp`, `relaunchApp`, `logToMain`, `setFullScreen`, `isFullScreen`, `mockCrashRenderProcess`, `setDisableHardwareAcceleration`, `setUseSystemTitleBar`, `installUVBinary`, `installBunBinary`, `getBinaryPath`.
  - `quit` / `relaunchApp` / `setStopQuitApp` → consolidated under `application`
- **Unused methods within otherwise-used namespaces:** `zip.compress`, `aes.encrypt`, `config.get`, `system.getCpuName`, `obsidian.getFolders`, `backup.checkConnection`, `backup.checkS3Connection`, `searchService.openSearchWindow`, `searchService.closeSearchWindow`, `ovms.getModels`, `ovms.isRunning`, `mcp.callTool`, `mcp.getPrompt`, `mcp.getResource`, `mcp.getInstallInfo`, `cherryin.saveToken`, `cherryin.hasToken`, `cherryin.exchangeToken`, `trace.*` (all except `getData`/`cleanLocalData`), `codeCli.getCustomTerminalPath`, `codeCli.removeCustomTerminalPath`, `quickAssistant.show`, `quickAssistant.toggle`, `application.quit`, `window.getSize`, `skill.list`, `skill.readSkillFile`, `skill.listFiles`, `lanTransfer.getState`, `openclaw.checkHealth`, `openclaw.getChannels`, and the removed `file` watcher methods (`upload`, `delete`, `deleteDir`, `read`, `clear`, `writeWithId`, `base64Image`, `saveBase64Image`, `download`, `copy`, `base64File`, `pdfInfo`, `openFileWithRelativePath`, `startFileWatcher`, `stopFileWatcher`, `pauseFileWatcher`, `resumeFileWatcher`, `onFileChange`, `getDirectoryStructure`).
