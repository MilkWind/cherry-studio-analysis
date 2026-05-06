# window.api Reference

## Overview

**window.api** is the main bridge between the Electron **renderer process** (React frontend) and the **main process** (Node.js backend) in Cherry Studio. It is defined in `src/preload/index.ts` and exposed via:

```typescript
contextBridge.exposeInMainWorld('api', api)
// Fallback for non-isolated mode:
window.api = api
```

All methods use `ipcRenderer.invoke()` (promise-based bidirectional IPC). A helper `tracedInvoke()` adds OpenTelemetry span context where needed (knowledge base, MCP tools).

**Key files:**
- Definition: `src/preload/index.ts` (lines 100-853)
- Type declaration: `src/preload/preload.d.ts`
- IPC channels: `packages/shared/IpcChannel.ts`

There are **592 references** to `window.api` across the renderer codebase.

---

## Architecture

**Exposure mechanism:** Uses `contextBridge.exposeInMainWorld()` (context-isolated mode) with a fallback for non-isolated mode.

**IPC Communication:** All methods use `ipcRenderer.invoke()` (promise-based bidirectional IPC). A special `tracedInvoke()` helper adds OpenTelemetry SpanContext to certain calls (`knowledgeBase.create/search/rerank`, `mcp.listTools/callTool`).

**Event Listeners:** Several namespaces return cleanup functions for IPC event listeners (e.g., `file.onFileChange`, `mcp.onServerLog`, `protocol.onReceiveData`, `agentSessionStream.onChunk`, `windowControls.onMaximizedChange`).

**Security:** The `shell.openExternal()` function includes protocol validation in the preload (sandboxed), only allowing `http:`, `https:`, `mailto:`, and `obsidian:` protocols.

---

## Complete API Reference

### A. Top-Level Functions (App/System Core)

| Function | Signature | Purpose |
|---|---|---|
| `getAppInfo()` | `() => Promise<AppInfo>` | Get application info |
| `getDiskInfo(directoryPath)` | `(string) => Promise<{free,size}\|null>` | Get disk space info for a directory |
| `reload()` | `() => Promise<void>` | Reload the application |
| `quit()` | `() => Promise<void>` | Quit the application |
| `setProxy(proxy, bypassRules?)` | `(string\|undefined, string?) => Promise<void>` | Set proxy configuration |
| `checkForUpdate()` | `() => Promise<void>` | Check for application updates |
| `quitAndInstall()` | `() => Promise<void>` | Quit and install update |
| `setLanguage(lang)` | `(string) => Promise<void>` | Set UI language |
| `setEnableSpellCheck(isEnable)` | `(boolean) => Promise<void>` | Enable/disable spell check |
| `setSpellCheckLanguages(languages)` | `(string[]) => Promise<void>` | Set spell check languages |
| `setLaunchOnBoot(isActive)` | `(boolean) => Promise<void>` | Set launch on system boot |
| `setLaunchToTray(isActive)` | `(boolean) => Promise<void>` | Set minimize to tray on launch |
| `setTray(isActive)` | `(boolean) => Promise<void>` | Enable/disable system tray |
| `setTrayOnClose(isActive)` | `(boolean) => Promise<void>` | Set tray on window close |
| `setTestPlan(isActive)` | `(boolean) => Promise<void>` | Set test plan mode |
| `setTestChannel(channel)` | `(UpgradeChannel) => Promise<void>` | Set update test channel |
| `setTheme(theme)` | `(ThemeMode) => Promise<void>` | Set application theme |
| `handleZoomFactor(delta, reset?)` | `(number, boolean?) => Promise<void>` | Adjust zoom factor |
| `setAutoUpdate(isActive)` | `(boolean) => Promise<void>` | Enable/disable auto-update |
| `select(options)` | `(Electron.OpenDialogOptions) => Promise<?>` | Open file/folder selection dialog |
| `hasWritePermission(path)` | `(string) => Promise<boolean>` | Check write permissions |
| `resolvePath(path)` | `(string) => Promise<string>` | Resolve a file path |
| `isPathInside(childPath, parentPath)` | `(string, string) => Promise<boolean>` | Check if child path is inside parent |
| `setAppDataPath(path)` | `(string) => Promise<void>` | Set application data directory |
| `getDataPathFromArgs()` | `() => Promise<string>` | Get data path from CLI args |
| `copy(oldPath, newPath, occupiedDirs?)` | `(string, string, string[]?) => Promise<?>` | Copy files/directories |
| `setStopQuitApp(stop, reason)` | `(boolean, string) => Promise<void>` | Prevent/prepare app quit |
| `flushAppData()` | `() => Promise<void>` | Flush application data |
| `isNotEmptyDir(path)` | `(string) => Promise<boolean>` | Check if directory is non-empty |
| `relaunchApp(options?)` | `(Electron.RelaunchOptions?) => Promise<void>` | Relaunch the application |
| `resetData()` | `() => Promise<void>` | Reset all application data |
| `openWebsite(url)` | `(string) => Promise<void>` | Open URL in external browser |
| `getCacheSize()` | `() => Promise<number>` | Get cache size |
| `clearCache()` | `() => Promise<void>` | Clear application cache |
| `logToMain(source, level, message, data)` | `(LogSourceWithContext, LogLevel, string, any[]) => Promise<?>` | Send log to main process |
| `setFullScreen(value)` | `(boolean) => Promise<void>` | Toggle fullscreen mode |
| `isFullScreen()` | `() => Promise<boolean>` | Check fullscreen status |
| `getSystemFonts()` | `() => Promise<string[]>` | Get installed system fonts |
| `getIpCountry()` | `() => Promise<string>` | Get user's IP country |
| `mockCrashRenderProcess()` | `() => Promise<void>` | Mock renderer crash (testing) |
| `quoteToMainWindow(text)` | `(string) => Promise<void>` | Quote selected text to main window |
| `setDisableHardwareAcceleration(isDisable)` | `(boolean) => Promise<void>` | Toggle hardware acceleration |
| `setUseSystemTitleBar(isActive)` | `(boolean) => Promise<void>` | Toggle system title bar |
| `openPath(path)` | `(string) => Promise<void>` | Open file/folder at path |

#### Binary Management

| Function | Signature | Purpose |
|---|---|---|
| `isBinaryExist(name)` | `(string) => Promise<boolean>` | Check if a binary exists |
| `getBinaryPath(name)` | `(string) => Promise<string>` | Get path to a binary |
| `installUVBinary()` | `() => Promise<void>` | Install UV binary (Python) |
| `installBunBinary()` | `() => Promise<void>` | Install Bun binary (JS) |
| `installOvmsBinary()` | `() => Promise<void>` | Install OVMS binary (OpenVINO) |

### B. `window.api.mac` (macOS-Specific)

| Function | Signature | Purpose |
|---|---|---|
| `isProcessTrusted()` | `() => Promise<boolean>` | Check macOS accessibility trust |
| `requestProcessTrust()` | `() => Promise<boolean>` | Request macOS accessibility trust |

### C. `window.api.notification`

| Function | Signature | Purpose |
|---|---|---|
| `send(notification)` | `(Notification) => Promise<void>` | Send a system notification |

### D. `window.api.system` (System Information)

| Function | Signature | Purpose |
|---|---|---|
| `getDeviceType()` | `() => Promise<string>` | Get OS device type |
| `getHostname()` | `() => Promise<string>` | Get system hostname |
| `getCpuName()` | `() => Promise<string>` | Get CPU name |
| `checkGitBash()` | `() => Promise<boolean>` | Check if Git Bash is installed |
| `getGitBashPath()` | `() => Promise<string\|null>` | Get Git Bash path |
| `getGitBashPathInfo()` | `() => Promise<GitBashPathInfo>` | Get Git Bash path info |
| `setGitBashPath(newPath)` | `(string\|null) => Promise<boolean>` | Set Git Bash path |

### E. `window.api.devTools`

| Function | Signature | Purpose |
|---|---|---|
| `toggle()` | `() => Promise<void>` | Toggle DevTools |

### F. `window.api.zip`

| Function | Signature | Purpose |
|---|---|---|
| `compress(text)` | `(string) => Promise<Buffer>` | Compress text |
| `decompress(text)` | `(Buffer) => Promise<string>` | Decompress data |

### G. `window.api.backup` (Backup/Restore)

| Function | Signature | Purpose |
|---|---|---|
| `restore(path)` | `(string) => Promise<void>` | Restore from backup file |
| `backup(fileName, destinationPath, skipBackupFile)` | `(string, string, boolean) => Promise<void>` | Create backup |
| `backupToWebdav(webdavConfig)` | `(WebDavConfig) => Promise<boolean>` | Backup to WebDAV |
| `restoreFromWebdav(webdavConfig)` | `(WebDavConfig) => Promise<string>` | Restore from WebDAV |
| `listWebdavFiles(webdavConfig)` | `(WebDavConfig) => Promise<File[]>` | List WebDAV backup files |
| `checkConnection(webdavConfig)` | `(WebDavConfig) => Promise<boolean>` | Check WebDAV connection |
| `checkWebdavConnection(webdavConfig)` | `(WebDavConfig) => Promise<boolean>` | Alias for checkConnection |
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
| `checkS3Connection(s3Config)` | `(S3Config) => Promise<boolean>` | Check S3 connection |
| `createLanTransferBackup(data, destinationPath?)` | `(string, string?) => Promise<string>` | Create LAN transfer backup |
| `deleteLanTransferBackup(filePath)` | `(string) => Promise<boolean>` | Delete LAN transfer backup |

### H. `window.api.file` (File Operations)

The most heavily-used namespace (~150+ references). Handles file I/O, metadata, dialogs, and file watching.

| Function | Signature | Purpose |
|---|---|---|
| `select(options?)` | `(OpenDialogOptions?) => Promise<FileMetadata[]\|null>` | Select files via dialog |
| `upload(file)` | `(FileMetadata) => Promise<?>` | Upload a file |
| `delete(fileId)` | `(string) => Promise<void>` | Delete a file by ID |
| `deleteDir(dirPath)` | `(string) => Promise<void>` | Delete a directory |
| `deleteExternalFile(filePath)` | `(string) => Promise<void>` | Delete external file |
| `deleteExternalDir(dirPath)` | `(string) => Promise<void>` | Delete external directory |
| `move(path, newPath)` | `(string, string) => Promise<void>` | Move a file |
| `moveDir(dirPath, newDirPath)` | `(string, string) => Promise<void>` | Move a directory |
| `rename(path, newName)` | `(string, string) => Promise<void>` | Rename a file |
| `renameDir(dirPath, newName)` | `(string, string) => Promise<void>` | Rename a directory |
| `read(fileId, detectEncoding?)` | `(string, boolean?) => Promise<string>` | Read file content by ID |
| `readExternal(filePath, detectEncoding?)` | `(string, boolean?) => Promise<string>` | Read external file content |
| `clear(spanContext?)` | `(SpanContext?) => Promise<void>` | Clear file cache |
| `get(filePath)` | `(string) => Promise<FileMetadata\|null>` | Get file metadata by path |
| `createTempFile(fileName)` | `(string) => Promise<string>` | Create a temp file |
| `mkdir(dirPath)` | `(string) => Promise<void>` | Create a directory |
| `write(filePath, data)` | `(string, Uint8Array\|string) => Promise<void>` | Write data to file |
| `writeWithId(id, content)` | `(string, string) => Promise<void>` | Write file by ID |
| `open(options?)` | `(OpenDialogOptions?) => Promise<?>` | Open file dialog |
| `openPath(path)` | `(string) => Promise<void>` | Open file at path |
| `save(path, content, options?)` | `(string, string\|ArrayBufferView, any?) => Promise<?>` | Save file |
| `selectFolder(options?)` | `(OpenDialogOptions?) => Promise<string\|null>` | Select folder via dialog |
| `saveImage(name, data)` | `(string, string) => Promise<boolean>` | Save an image file |
| `binaryImage(fileId)` | `(string) => Promise<Buffer>` | Get image as binary |
| `base64Image(fileId)` | `(string) => Promise<{mime,base64,data}>` | Get image as base64 |
| `saveBase64Image(data)` | `(string) => Promise<?>` | Save a base64 image |
| `savePastedImage(imageData, extension?)` | `(Uint8Array, string?) => Promise<?>` | Save pasted image data |
| `download(url, isUseContentType?)` | `(string, boolean?) => Promise<?>` | Download a file from URL |
| `copy(fileId, destPath)` | `(string, string) => Promise<void>` | Copy a file |
| `base64File(fileId)` | `(string) => Promise<string>` | Get file as base64 |
| `pdfInfo(fileId)` | `(string) => Promise<?>` | Get PDF info |
| `getPathForFile(file)` | `(File) => string` | Get path for a File object (synchronous) |
| `openFileWithRelativePath(file)` | `(FileMetadata) => Promise<void>` | Open file using relative path |
| `isTextFile(filePath)` | `(string) => Promise<boolean>` | Check if file is text |
| `isDirectory(filePath)` | `(string) => Promise<boolean>` | Check if path is directory |
| `getDirectoryStructure(dirPath)` | `(string) => Promise<?>` | Get directory tree |
| `listDirectory(dirPath, options?)` | `(string, DirectoryListOptions?) => Promise<?>` | List directory contents |
| `checkFileName(dirPath, fileName, isFile)` | `(string, string, boolean) => Promise<{safeName}>` | Validate file/dir name |
| `validateNotesDirectory(dirPath)` | `(string) => Promise<void>` | Validate notes directory |
| `startFileWatcher(dirPath, config?)` | `(string, any?) => Promise<void>` | Watch directory for changes |
| `stopFileWatcher()` | `() => Promise<void>` | Stop file watcher |
| `pauseFileWatcher()` | `() => Promise<void>` | Pause file watcher |
| `resumeFileWatcher()` | `() => Promise<void>` | Resume file watcher |
| `batchUploadMarkdown(filePaths, targetPath)` | `(string[], string) => Promise<void>` | Batch upload markdown files |
| `onFileChange(callback)` | `((FileChangeEvent) => void) => () => void` | Subscribe to file change events |
| `showInFolder(path)` | `(string) => Promise<void>` | Show file in system file manager |

### I. `window.api.fs` (Low-Level File System)

| Function | Signature | Purpose |
|---|---|---|
| `read(pathOrUrl, encoding?)` | `(string, BufferEncoding?) => Promise<Buffer\|string>` | Read file from filesystem |
| `readText(pathOrUrl)` | `(string) => Promise<string>` | Read file as text |

### J. `window.api.pdf`

| Function | Signature | Purpose |
|---|---|---|
| `extractText(data)` | `(Uint8Array\|ArrayBuffer\|string) => Promise<string>` | Extract text from PDF data |

### K. `window.api.export`

| Function | Signature | Purpose |
|---|---|---|
| `toWord(markdown, fileName)` | `(string, string) => Promise<void>` | Export markdown to Word document |

### L. `window.api.obsidian`

| Function | Signature | Purpose |
|---|---|---|
| `getVaults()` | `() => Promise<?>` | Get Obsidian vaults |
| `getFolders(vaultName)` | `(string) => Promise<?>` | Get Obsidian folders |
| `getFiles(vaultName)` | `(string) => Promise<?>` | Get Obsidian files |

### M. `window.api.shortcuts`

| Function | Signature | Purpose |
|---|---|---|
| `update(shortcuts)` | `(Shortcut[]) => Promise<void>` | Update keyboard shortcuts |

### N. `window.api.knowledgeBase` (RAG/Knowledge Base)

| Function | Signature | Purpose |
|---|---|---|
| `create(base, context?)` | `(KnowledgeBaseParams, SpanContext?) => Promise<?>` | Create knowledge base |
| `reset(base)` | `(KnowledgeBaseParams) => Promise<?>` | Reset knowledge base |
| `delete(id)` | `(string) => Promise<void>` | Delete knowledge base |
| `add({base, item, userId?, forceReload?})` | `({...}) => Promise<?>` | Add item to knowledge base |
| `remove({uniqueId, uniqueIds, base})` | `({...}) => Promise<?>` | Remove item from knowledge base |
| `search({search, base}, context?)` | `({...}, SpanContext?) => Promise<?>` | Search knowledge base |
| `rerank({search, base, results}, context?)` | `({...}, SpanContext?) => Promise<?>` | Re-rank search results |

### O. `window.api.memory` (AI Memory)

| Function | Signature | Purpose |
|---|---|---|
| `add(messages, options?)` | `(string\|AssistantMessage[], AddMemoryOptions?) => Promise<?>` | Add to AI memory |
| `search(query, options)` | `(string, MemorySearchOptions) => Promise<?>` | Search AI memory |
| `list(options?)` | `(MemoryListOptions?) => Promise<?>` | List memories |
| `delete(id)` | `(string) => Promise<void>` | Delete a memory |
| `update(id, memory, metadata?)` | `(string, string, Record<string,any>?) => Promise<?>` | Update a memory |
| `get(id)` | `(string) => Promise<?>` | Get a specific memory |
| `setConfig(config)` | `(MemoryConfig) => Promise<void>` | Set memory config |
| `deleteUser(userId)` | `(string) => Promise<void>` | Delete user's memories |
| `deleteAllMemoriesForUser(userId)` | `(string) => Promise<void>` | Delete all user memories |
| `getUsersList()` | `() => Promise<?>` | Get list of users |
| `migrateMemoryDb()` | `() => Promise<void>` | Migrate memory database |

### P. `window.api.window` (Window Management)

| Function | Signature | Purpose |
|---|---|---|
| `setMinimumSize(width, height)` | `(number, number) => Promise<void>` | Set minimum window size |
| `resetMinimumSize()` | `() => Promise<void>` | Reset minimum window size |
| `getSize()` | `() => Promise<[number, number]>` | Get window dimensions |

### Q. `window.api.fileService` (Cloud File Service)

| Function | Signature | Purpose |
|---|---|---|
| `upload(provider, file)` | `(Provider, FileMetadata) => Promise<FileUploadResponse>` | Upload file to provider |
| `list(provider)` | `(Provider) => Promise<FileListResponse>` | List files on provider |
| `delete(provider, fileId)` | `(Provider, string) => Promise<void>` | Delete file from provider |
| `retrieve(provider, fileId)` | `(Provider, string) => Promise<FileUploadResponse>` | Retrieve file from provider |

### R. `window.api.selectionMenu`

| Function | Signature | Purpose |
|---|---|---|
| `action(action)` | `(string) => Promise<void>` | Trigger a selection menu action |

### S. `window.api.vertexAI`

| Function | Signature | Purpose |
|---|---|---|
| `getAuthHeaders(params)` | `({projectId, serviceAccount?}) => Promise<?>` | Get VertexAI auth headers |
| `getAccessToken(params)` | `({projectId, serviceAccount?}) => Promise<string>` | Get VertexAI access token |
| `clearAuthCache(projectId, clientEmail?)` | `(string, string?) => Promise<void>` | Clear VertexAI auth cache |

### T. `window.api.ovms` (OpenVINO Model Server)

| Function | Signature | Purpose |
|---|---|---|
| `isSupported()` | `() => Promise<boolean>` | Check OVMS support |
| `addModel(modelName, modelId, modelSource, task)` | `(string, string, string, string) => Promise<?>` | Add an OVMS model |
| `stopAddModel()` | `() => Promise<void>` | Cancel adding model |
| `getModels()` | `() => Promise<?>` | Get installed models |
| `isRunning()` | `() => Promise<boolean>` | Check if OVMS is running |
| `getStatus()` | `() => Promise<?>` | Get OVMS status |
| `runOvms()` | `() => Promise<void>` | Start OVMS server |
| `stopOvms()` | `() => Promise<void>` | Stop OVMS server |

### U. `window.api.config`

| Function | Signature | Purpose |
|---|---|---|
| `set(key, value, isNotify?)` | `(string, any, boolean?) => Promise<void>` | Set config value |
| `get(key)` | `(string) => Promise<any>` | Get config value |

### V. `window.api.miniWindow`

| Function | Signature | Purpose |
|---|---|---|
| `show()` | `() => Promise<void>` | Show mini window |
| `hide()` | `() => Promise<void>` | Hide mini window |
| `close()` | `() => Promise<void>` | Close mini window |
| `toggle()` | `() => Promise<void>` | Toggle mini window visibility |
| `setPin(isPinned)` | `(boolean) => Promise<void>` | Pin/unpin mini window |

### W. `window.api.aes` (Encryption)

| Function | Signature | Purpose |
|---|---|---|
| `encrypt(text, secretKey, iv)` | `(string, string, string) => Promise<string>` | AES encrypt data |
| `decrypt(encryptedData, iv, secretKey)` | `(string, string, string) => Promise<string>` | AES decrypt data |

### X. `window.api.mcp` (Model Context Protocol)

| Function | Signature | Purpose |
|---|---|---|
| `removeServer(server)` | `(MCPServer) => Promise<void>` | Remove MCP server |
| `restartServer(server)` | `(MCPServer) => Promise<void>` | Restart MCP server |
| `stopServer(server)` | `(MCPServer) => Promise<void>` | Stop MCP server |
| `listTools(server, context?)` | `(MCPServer, SpanContext?) => Promise<?>` | List MCP server tools |
| `callTool({server, name, args, callId?}, context?)` | `({...}, SpanContext?) => Promise<?>` | Call an MCP tool |
| `listPrompts(server)` | `(MCPServer) => Promise<?>` | List MCP prompts |
| `getPrompt({server, name, args?})` | `({MCPServer, string, Record?}) => Promise<?>` | Get MCP prompt |
| `listResources(server)` | `(MCPServer) => Promise<?>` | List MCP resources |
| `getResource({server, uri})` | `({MCPServer, string}) => Promise<?>` | Get MCP resource |
| `getInstallInfo()` | `() => Promise<?>` | Get MCP install info |
| `checkMcpConnectivity(server)` | `(any) => Promise<?>` | Check MCP connectivity |
| `uploadDxt(file)` | `(File) => Promise<?>` | Upload DXT file |
| `abortTool(callId)` | `(string) => Promise<void>` | Abort a tool call |
| `resolveHubTool(nameOrId)` | `(string) => Promise<{serverId,toolName}\|null>` | Resolve hub tool by name/ID |
| `getServerVersion(server)` | `(MCPServer) => Promise<string\|null>` | Get MCP server version |
| `getServerLogs(server)` | `(MCPServer) => Promise<MCPServerLogEntry[]>` | Get MCP server logs |
| `onServerLog(callback)` | `((MCPServerLogEntry) => void) => () => void` | Subscribe to server log events |

### Y. `window.api.python`

| Function | Signature | Purpose |
|---|---|---|
| `execute(script, context?, timeout?)` | `(string, Record?, number?) => Promise<?>` | Execute Python script |

### Z. `window.api.shell`

| Function | Signature | Purpose |
|---|---|---|
| `openExternal(url, options?)` | `(string, OpenExternalOptions?) => Promise<void>` | Open URL in system browser (protocol-restricted) |

### AA. `window.api.copilot` (GitHub Copilot Auth)

| Function | Signature | Purpose |
|---|---|---|
| `getAuthMessage(headers?)` | `(Record<string,string>?) => Promise<?>` | Get Copilot auth message |
| `getCopilotToken(device_code, headers?)` | `(string, Record<string,string>?) => Promise<?>` | Get Copilot token |
| `saveCopilotToken(access_token)` | `(string) => Promise<void>` | Save Copilot token |
| `getToken(headers?)` | `(Record<string,string>?) => Promise<string>` | Get Copilot token |
| `logout()` | `() => Promise<void>` | Logout from Copilot |
| `getUser(token)` | `(string) => Promise<?>` | Get Copilot user info |

### AB. `window.api.cherryin` (CherryIN OAuth)

| Function | Signature | Purpose |
|---|---|---|
| `saveToken(accessToken, refreshToken?)` | `(string, string?) => Promise<void>` | Save CherryIN token |
| `hasToken()` | `() => Promise<boolean>` | Check if CherryIN token exists |
| `getBalance(apiHost)` | `(string) => Promise<?>` | Get CherryIN balance |
| `logout(apiHost)` | `(string) => Promise<void>` | CherryIN logout |
| `startOAuthFlow(oauthServer, apiHost?)` | `(string, string?) => Promise<{authUrl,state}>` | Start CherryIN OAuth flow |
| `exchangeToken(code, state)` | `(string, string) => Promise<{apiKeys}>` | Exchange OAuth code for token |

### AC. `window.api.protocol` (Custom Protocol Handler)

| Function | Signature | Purpose |
|---|---|---|
| `onReceiveData(callback)` | `(({url, params}) => void) => () => void` | Listen for custom protocol data (OAuth callbacks) |

### AD. `window.api.externalApps`

| Function | Signature | Purpose |
|---|---|---|
| `detectInstalled()` | `() => Promise<ExternalAppInfo[]>` | Detect installed external apps |

### AE. `window.api.nutstore`

| Function | Signature | Purpose |
|---|---|---|
| `getSSOUrl()` | `() => Promise<string>` | Get Nutstore SSO URL |
| `decryptToken(token)` | `(string) => Promise<?>` | Decrypt Nutstore token |
| `getDirectoryContents(token, path)` | `(string, string) => Promise<?>` | List Nutstore directory contents |

### AF. `window.api.searchService`

| Function | Signature | Purpose |
|---|---|---|
| `openSearchWindow(uid, show?)` | `(string, boolean?) => Promise<void>` | Open search window |
| `closeSearchWindow(uid)` | `(string) => Promise<void>` | Close search window |
| `openUrlInSearchWindow(uid, url)` | `(string, string) => Promise<void>` | Open URL in search window |

### AG. `window.api.webview`

| Function | Signature | Purpose |
|---|---|---|
| `setOpenLinkExternal(webviewId, isExternal)` | `(number, boolean) => Promise<void>` | Set external link handling |
| `setSpellCheckEnabled(webviewId, isEnable)` | `(number, boolean) => Promise<void>` | Enable spell check in webview |
| `printToPDF(webviewId)` | `(number) => Promise<void>` | Print webview to PDF |
| `saveAsHTML(webviewId)` | `(number) => Promise<void>` | Save webview as HTML |
| `onFindShortcut(callback)` | `((WebviewKeyEvent) => void) => () => void` | Listen for find/ctrl+F shortcuts |

### AH. `window.api.storeSync` (Multi-Window Store Sync)

| Function | Signature | Purpose |
|---|---|---|
| `subscribe()` | `() => Promise<void>` | Subscribe to store sync |
| `unsubscribe()` | `() => Promise<void>` | Unsubscribe from store sync |
| `onUpdate(action)` | `(any) => Promise<void>` | Notify store update across windows |

### AI. `window.api.selection` (Selection Toolbar/Menu)

| Function | Signature | Purpose |
|---|---|---|
| `hideToolbar()` | `() => Promise<void>` | Hide selection toolbar |
| `writeToClipboard(text)` | `(string) => Promise<void>` | Write text to clipboard |
| `determineToolbarSize(width, height)` | `(number, number) => Promise<void>` | Report toolbar dimensions |
| `setEnabled(enabled)` | `(boolean) => Promise<void>` | Enable/disable selection |
| `setTriggerMode(triggerMode)` | `(string) => Promise<void>` | Set selection trigger mode |
| `setFollowToolbar(isFollowToolbar)` | `(boolean) => Promise<void>` | Set toolbar follow mode |
| `setRemeberWinSize(isRemeberWinSize)` | `(boolean) => Promise<void>` | Set remember window size |
| `setFilterMode(filterMode)` | `(string) => Promise<void>` | Set filter mode |
| `setFilterList(filterList)` | `(string[]) => Promise<void>` | Set filter list |
| `processAction(actionItem, isFullScreen?)` | `(ActionItem, boolean?) => Promise<void>` | Process a selection action |
| `closeActionWindow()` | `() => Promise<void>` | Close action window |
| `minimizeActionWindow()` | `() => Promise<void>` | Minimize action window |
| `pinActionWindow(isPinned)` | `(boolean) => Promise<void>` | Pin/unpin action window |
| `getLinuxEnvInfo()` | `() => Promise<?>` | Get Linux environment info |

### AJ. `window.api.agentTools`

| Function | Signature | Purpose |
|---|---|---|
| `respondToPermission(payload)` | `({requestId, behavior, updatedInput?, message?, updatedPermissions?}) => Promise<?>` | Respond to AI agent tool permission request |

### AK. `window.api.agentSessionStream` (Agent Session Streaming)

| Function | Signature | Purpose |
|---|---|---|
| `subscribe(sessionId)` | `(string) => Promise<void>` | Subscribe to agent session stream |
| `unsubscribe(sessionId)` | `(string) => Promise<void>` | Unsubscribe from session stream |
| `abort(sessionId)` | `(string) => Promise<void>` | Abort an agent session |
| `onChunk(callback)` | `(callback) => cleanup function` | Listen for agent stream chunks |
| `onSessionChanged(callback)` | `(callback) => cleanup function` | Listen for session changes |

### AL. `window.api.wechat` (WeChat Channel)

| Function | Signature | Purpose |
|---|---|---|
| `onQrLogin(callback)` | `(callback) => cleanup function` | Listen for WeChat QR login events |
| `hasCredentials(channelId)` | `(string) => Promise<{exists, userId?}>` | Check WeChat credentials |

### AM. `window.api.feishu` (Feishu/Lark Channel)

| Function | Signature | Purpose |
|---|---|---|
| `onQrLogin(callback)` | `(callback) => cleanup function` | Listen for Feishu QR login events |

### AN. `window.api.channel` (Channel Management)

| Function | Signature | Purpose |
|---|---|---|
| `onLog(callback)` | `(callback) => cleanup function` | Listen for channel logs |
| `onStatusChange(callback)` | `(callback) => cleanup function` | Listen for channel status changes |
| `getLogs(channelId)` | `(string) => Promise<Array<{timestamp,level,message,channelId}>>` | Get channel logs |
| `getStatuses()` | `() => Promise<Array<{channelId, connected, error?}>>` | Get all channel statuses |

### AO. `window.api.trace` (OpenTelemetry Tracing)

| Function | Signature | Purpose |
|---|---|---|
| `saveData(topicId)` | `(string) => Promise<void>` | Save trace data |
| `getData(topicId, traceId, modelName?)` | `(string, string, string?) => Promise<any[]>` | Get trace data |
| `saveEntity(entity)` | `(SpanEntity) => Promise<void>` | Save a span entity |
| `getEntity(spanId)` | `(string) => Promise<?>` | Get a span entity |
| `bindTopic(topicId, traceId)` | `(string, string) => Promise<void>` | Bind a trace to a topic |
| `tokenUsage(spanId, usage)` | `(string, TokenUsage) => Promise<void>` | Record token usage |
| `cleanHistory(topicId, traceId, modelName?)` | `(string, string, string?) => Promise<void>` | Clean trace history |
| `cleanTopic(topicId, traceId?)` | `(string, string?) => Promise<void>` | Clean a topic's traces |
| `openWindow(topicId, traceId, autoOpen?, modelName?)` | `(string, string, boolean?, string?) => Promise<void>` | Open trace window |
| `setTraceWindowTitle(title)` | `(string) => Promise<void>` | Set trace window title |
| `addEndMessage(spanId, modelName, context)` | `(string, string, string) => Promise<void>` | Add end message to trace |
| `cleanLocalData()` | `() => Promise<void>` | Clean local trace data |
| `addStreamMessage(spanId, modelName, context, message)` | `(string, string, string, any) => Promise<void>` | Add stream message to trace |

### AP. `window.api.anthropic_oauth`

| Function | Signature | Purpose |
|---|---|---|
| `startOAuthFlow()` | `() => Promise<void>` | Start Anthropic OAuth flow |
| `completeOAuthWithCode(code)` | `(string) => Promise<void>` | Complete OAuth with code |
| `cancelOAuthFlow()` | `() => Promise<void>` | Cancel OAuth flow |
| `getAccessToken()` | `() => Promise<string>` | Get Anthropic access token |
| `hasCredentials()` | `() => Promise<boolean>` | Check Anthropic credentials |
| `clearCredentials()` | `() => Promise<void>` | Clear Anthropic credentials |

### AQ. `window.api.codeTools`

| Function | Signature | Purpose |
|---|---|---|
| `run(cliTool, model, directory, env, options?)` | `(string, string, string, Record<string,string>, {autoUpdateToLatest?, terminal?}?) => Promise<CodeToolsRunResult>` | Run code tools (e.g., Claude Code) |
| `getAvailableTerminals()` | `() => Promise<TerminalConfig[]>` | Get available terminals |
| `setCustomTerminalPath(terminalId, path)` | `(string, string) => Promise<void>` | Set custom terminal path |
| `getCustomTerminalPath(terminalId)` | `(string) => Promise<string\|undefined>` | Get custom terminal path |
| `removeCustomTerminalPath(terminalId)` | `(string) => Promise<void>` | Remove custom terminal path |

### AR. `window.api.ocr`

| Function | Signature | Purpose |
|---|---|---|
| `ocr(file, provider)` | `(SupportedOcrFile, OcrProvider) => Promise<OcrResult>` | Perform OCR on a file |
| `listProviders()` | `() => Promise<string[]>` | List OCR providers |

### AS. `window.api.cherryai`

| Function | Signature | Purpose |
|---|---|---|
| `generateSignature(params)` | `({method, path, query, body}) => Promise<?>` | Generate CherryAI API signature |

### AT. `window.api.windowControls` (Native Window Controls)

| Function | Signature | Purpose |
|---|---|---|
| `minimize()` | `() => Promise<void>` | Minimize window |
| `maximize()` | `() => Promise<void>` | Maximize window |
| `unmaximize()` | `() => Promise<void>` | Restore window from maximized |
| `close()` | `() => Promise<void>` | Close window |
| `isMaximized()` | `() => Promise<boolean>` | Check if window is maximized |
| `onMaximizedChange(callback)` | `((boolean) => void) => () => void` | Listen for maximize state changes |

### AU. `window.api.apiServer`

| Function | Signature | Purpose |
|---|---|---|
| `getStatus()` | `() => Promise<GetApiServerStatusResult>` | Get API server status |
| `start()` | `() => Promise<StartApiServerStatusResult>` | Start API server |
| `restart()` | `() => Promise<RestartApiServerStatusResult>` | Restart API server |
| `stop()` | `() => Promise<StopApiServerStatusResult>` | Stop API server |
| `onReady(callback)` | `(() => void) => () => void` | Listen for server ready event |

### AV. `window.api.skill`

| Function | Signature | Purpose |
|---|---|---|
| `list(agentId?)` | `(string?) => Promise<SkillResult<InstalledSkill[]>>` | List installed skills |
| `install(options)` | `(SkillInstallOptions) => Promise<SkillResult<InstalledSkill>>` | Install a skill |
| `uninstall(skillId)` | `(string) => Promise<SkillResult<void>>` | Uninstall a skill |
| `toggle(options)` | `(SkillToggleOptions) => Promise<SkillResult<InstalledSkill\|null>>` | Toggle a skill on/off |
| `installFromZip(options)` | `(SkillInstallFromZipOptions) => Promise<SkillResult<InstalledSkill>>` | Install skill from ZIP |
| `installFromDirectory(options)` | `(SkillInstallFromDirectoryOptions) => Promise<SkillResult<InstalledSkill>>` | Install skill from directory |
| `readSkillFile(skillId, filename)` | `(string, string) => Promise<SkillResult<string\|null>>` | Read a skill file |
| `listFiles(skillId)` | `(string) => Promise<SkillResult<SkillFileNode[]>>` | List skill files |
| `listLocal(workdir)` | `(string) => Promise<SkillResult<LocalSkill[]>>` | List local skills |

### AW. `window.api.localTransfer` (LAN File Transfer)

| Function | Signature | Purpose |
|---|---|---|
| `getState()` | `() => Promise<LocalTransferState>` | Get local transfer state |
| `startScan()` | `() => Promise<LocalTransferState>` | Start scanning for peers |
| `stopScan()` | `() => Promise<LocalTransferState>` | Stop scanning for peers |
| `connect(payload)` | `(LocalTransferConnectPayload) => Promise<LanHandshakeAckMessage>` | Connect to a peer |
| `disconnect()` | `() => Promise<void>` | Disconnect from peer |
| `onServicesUpdated(callback)` | `(callback) => cleanup function` | Listen for service updates |
| `onClientEvent(callback)` | `(callback) => cleanup function` | Listen for client events |
| `sendFile(filePath)` | `(string) => Promise<LanFileCompleteMessage>` | Send file to peer |
| `cancelTransfer()` | `() => Promise<void>` | Cancel file transfer |

### AX. `window.api.openclaw` (OpenClaw Integration)

| Function | Signature | Purpose |
|---|---|---|
| `checkInstalled()` | `() => Promise<{installed, path?, needsMigration}>` | Check if OpenClaw is installed |
| `install()` | `() => Promise<OperationResult>` | Install OpenClaw |
| `uninstall()` | `() => Promise<OperationResult>` | Uninstall OpenClaw |
| `startGateway(port?)` | `(number?) => Promise<OperationResult>` | Start OpenClaw gateway |
| `stopGateway()` | `() => Promise<OperationResult>` | Stop OpenClaw gateway |
| `getStatus()` | `() => Promise<{status, port}>` | Get OpenClaw status |
| `checkHealth()` | `() => Promise<OpenClawHealthInfo>` | Check OpenClaw health |
| `getDashboardUrl()` | `() => Promise<string>` | Get OpenClaw dashboard URL |
| `syncConfig(provider, primaryModel)` | `(Provider, Model) => Promise<OperationResult>` | Sync config to OpenClaw |
| `getChannels()` | `() => Promise<OpenClawChannelInfo[]>` | Get OpenClaw channels |
| `checkUpdate()` | `() => Promise<{hasUpdate, currentVersion, latestVersion, message?}>` | Check OpenClaw updates |
| `performUpdate()` | `() => Promise<OperationResult>` | Perform OpenClaw update |

### AY. `window.api.analytics`

| Function | Signature | Purpose |
|---|---|---|
| `trackTokenUsage(data)` | `(TokenUsageData) => Promise<void>` | Track token usage analytics |

---

## Usage Summary by Namespace

| Namespace | Approx. Uses | Description |
|---|---|---|
| `window.api.file.*` | ~150+ | File I/O (read, write, get, save, etc.) |
| `window.api.trace.*` | ~80+ | OpenTelemetry tracing |
| `window.api.backup.*` | ~40+ | Backup/restore operations |
| `window.api.knowledgeBase.*` | ~30+ | Knowledge base / RAG |
| `window.api.selection.*` | ~30+ | Selection toolbar interactions |
| `window.api.mcp.*` | ~20+ | MCP tool/server management |
| `window.api.storeSync.*` | ~15+ | Multi-window state sync |
| `window.api.config.*` | ~10+ | Application configuration |

**Key files that use window.api most:**
- `src/renderer/src/services/SpanManagerService.ts` - trace operations
- `src/renderer/src/apps/notes-app/useNotes.ts` - file operations
- `src/renderer/src/services/NutstoreService.ts` - backup operations
- `src/renderer/src/services/WebSearchService.ts` - knowledge base operations
- `src/renderer/src/utils/export.ts` - file save/export operations
