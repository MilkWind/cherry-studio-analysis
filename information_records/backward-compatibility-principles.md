# Backward Compatibility Principles in Cherry Studio

## 1. Versioning Strategy

The project follows **semantic versioning** (`major.minor.patch`) managed through two mechanisms:

- **Main app** (`package.json` v1.9.4): Bumped via `node scripts/version.js` (supports `patch`/`minor`/`major`), which creates a git tag with `v` prefix.
- **Internal packages** (`@cherrystudio/ai-core`, etc.): Managed via **Changesets** — PRs modifying packages must include a changeset; CI automates changelog generation and npm publishing.

Branching strategy (`docs/en/guides/branching-strategy.md`):
- **Major:** `v1.0.0`, `v2.0.0`
- **Feature:** `v1.1.0`, `v1.2.0`
- **Patch:** `v1.0.1`, `v1.0.2`
- **Hotfix:** `v1.0.1-hotfix`

Release branches accept only documentation updates and bug fixes — no breaking features.

---

## 2. Upgrade Path System

The project enforces **safe, multi-step upgrade paths** to prevent data loss and ensure compatibility.

### Core Configuration

**`app-upgrade-config.json`** — Central upgrade path definition:
- Each version entry specifies a `minCompatibleVersion` constraint
- Segments are classified as `legacy` or `breaking`
- Enforces intermediate version upgrades (e.g., v1.6.x → v1.7.x → v2.0.x)

**`config/app-upgrade-segments.json`** — Semver-range-based segment definitions:
- Segment types: `legacy`, `latest`, `breaking`
- Example: `gateway-v1.8.1` (legacy), `current-v1` (latest), `gateway-v2` (breaking), `current-v2` (latest)
- Each segment specifies its own `minCompatibleVersion` and channel feed URLs

### Enforcement in Code

`src/main/services/AppUpdater.ts` (lines 182–232) — `_findCompatibleChannel()`:
- Uses `semver.rcompare` to sort versions in descending order
- Checks if current version meets `minCompatibleVersion`: `if (!semver.gte(currentVersion, versionConfig.minCompatibleVersion))`
- Falls back to GitHub releases if the upgrade config fetch fails

### Documented Upgrade Paths

`docs/en/references/app-upgrade.md` documents 6 upgrade scenarios:
1. v1.6.5 → v2.x (requires intermediate v1.7.x)
2. v1.7.0 → v2.x (direct upgrade allowed)
3. v1.7.5 → v2.1.0 (direct upgrade allowed)
4. v1.6.3 → v1.6.7 (patch within same major)
5. v1.6.7 → v2.0.0 (blocked — must go through v1.7.x)
6. v1.6.3 → v1.7.5 → v2.0.0 → v2.1.6 (full safe path)

**Key commitment** (from docs):
> "Users below v1.7 must first upgrade to v1.7+; users v1.7+ can directly upgrade to v2.x."

The upgrade config is automatically synced on every release via `.github/workflows/update-app-upgrade-config.yml`.

---

## 3. Data Migration Infrastructure

### Redux Store Migrations (`src/renderer/src/store/migrate.ts` — 3,423 lines)

A comprehensive `redux-persist` migration system using `createMigrate`, keyed by numeric versions (201, 202, 203, …). Each migration mutates the Redux state shape:
- Removes deprecated fields (mini-app icons, old provider keys)
- Adds new providers (e.g., `cherryai` → `qwen` reassignment)
- Transforms data formats (language codes, web search config)
- Backfills missing fields (e.g., `mcpMode` default, `anthropicApiHost`)

### IndexedDB Schema Upgrades (`src/renderer/src/databases/upgrades.ts` — 415 lines)

Dexie database upgrades supporting 5 schema versions:
- **V5:** Migrates Date objects to ISO strings; migrates `tavily` metadata to `webSearch`
- **V7:** Major migration — converts legacy message format to block-based message architecture
- **V8:** Migrates language codes (e.g., `"english"` → `"en-us"`)

### SQLite/Drizzle Migrations (`src/main/services/agents/database/MigrationService.ts`)

Full migration engine for the agents subsystem:
- Tracks applied migrations via a `migrations` table
- Reads migration journal from `resources/database/drizzle/meta/_journal.json`
- Executes pending SQL migrations sequentially

### Principle: Every schema change must have a migration.

---

## 4. Deprecation Policy

The project has a **formalized deprecation pattern** across 80+ files:

```
@deprecated Scheduled for removal in v2.0.0
⚠️ NOTICE: V2 DATA&UI REFACTORING (by 0xfullex)
STOP: Feature PRs affecting this file are currently BLOCKED.
Only critical bug fixes are accepted during this migration phase.
Contribution Hold: https://github.com/CherryHQ/cherry-studio/issues/10954
v2 Refactor PR   : https://github.com/CherryHQ/cherry-studio/pull/10162
```

This applies to store files, service files, database files, and hooks that are slated for replacement in v2.0.0.

### Deprecated Type Definitions

TypeScript `@deprecated` JSDoc tags are used on:
- Provider type fields (`src/renderer/src/types/provider.ts`)
- Message fields (`src/renderer/src/types/newMessage.ts`)
- Agent configuration types (`src/renderer/src/types/agent.ts`)
- Protocol types kept for reference only (`packages/shared/config/types.ts`)

### Deprecated Functions

Functions are marked `@deprecated` and replaced by new implementations:
- `deleteFiles()` → `safeDeleteFiles()`
- Legacy import methods → `importConversations()`
- Old toast API → new stub wrapper

### Principle: Deprecate first, remove later. Always provide a migration path.

---

## 5. Backward-Compatible Re-exports

The codebase uses re-exports extensively to maintain API compatibility during refactoring:

```typescript
// Re-export from shared, for backward compatibility
export * from 'shared/path'
```

Found in: `api.ts`, `provider.ts`, `settings.ts`, `models.ts`, `LanTransferClientService.ts`, and more.

### Fallback Defaults

When new fields are added, the code provides sensible defaults for old data:

- **MCP Mode** (`src/renderer/src/types/index.ts`): If `assistant.mcpMode` is absent, falls back to `'manual'` if `mcpServers` exist, otherwise `'disabled'`
- **Toast API** (`src/renderer/src/components/TopView/toast.tsx`): Stub no-op functions for backward compatibility with the previous toast API
- **Config defaults** (`src/renderer/src/config/constant.ts`): `DEFAULT_TEMPERATURE = 1.0`, `DEFAULT_CONTEXTCOUNT = 5`, etc.
- **Update feeds** (`AppUpdater.ts`): Falls back to `FeedUrl.PRODUCTION` or `FeedUrl.GITHUB_LATEST` if upgrade config fetch fails

### Principle: Never break existing callers. Re-export, alias, or default rather than remove.

---

## 6. Feature Freeze & Code Freeze

During the v2 migration period:
- The `main` branch is under **code freeze** — only critical bug fixes via `hotfix/*` branches
- The `v2` branch accepts all new features, refactoring, and optimizations
- Files with the deprecation header accept **bug fixes only** — no new features
- Redux state shape changes and schema modifications are blocked until v2.0.0

### Enforcement

`CLAUDE.md` (lines 168, 175):
> "BLOCKED: Do not add new Redux slices or change existing state shape until v2.0.0."
> "BLOCKED: Do not modify schema until v2.0.0."

---

## 7. PR & Contribution Compatibility Checks

The **PR template** (`.github/pull_request_template.md`) requires:
- A `### Breaking changes` section describing impact on users
- A checklist item: _"Upgrade: Impact of this change on upgrade flows was considered and addressed if required"_
- If action is needed from users, the release note must contain `'action required'`

`CONTRIBUTING.md` also includes upgrade flow impact as a checklist item.

### Principle: Every PR must consider upgrade compatibility before merging.

---

## 8. Automated Testing of Compatibility

The project has explicit backward compatibility tests:

- **MCP Mode** (`src/renderer/src/services/__tests__/mcpMode.test.ts`): 3 tests verifying fallback behavior for legacy assistants without `mcpMode`
- **App Upgrade Path** (`src/main/services/__tests__/AppUpdater.test.ts`): Tests covering blocked direct upgrades, allowed direct upgrades, and full multi-step upgrade paths with semver compatibility
- **Provider Config** (`src/renderer/src/aiCore/provider/__tests__/providerConfig.test.ts`): "OpenAI-compatible fallback" test block
- **Options Conversion** (`src/renderer/src/aiCore/utils/__tests__/options.test.ts`): Tests for `reasoning_effort` to `reasoningEffort` auto-conversion (backward compat for #11987)
- **Store Migration** (`src/renderer/src/store/__tests__/mcp.test.ts`): Tests migration logic for v202 filesystem approval backfill
- **Provider Registry** (`src/renderer/src/aiCore/provider/__tests__/integratedRegistry.test.ts`): "should maintain compatibility with existing providers"

---

## Summary of Principles

| # | Principle | How It's Enforced |
|---|-----------|-------------------|
| 1 | **Every upgrade must be safe** | `app-upgrade-config.json` enforces `minCompatibleVersion` constraints; blocked direct upgrades to breaking versions |
| 2 | **Every schema change needs a migration** | Redux migrations (3,423 lines), IndexedDB upgrades (415 lines), SQLite migration service |
| 3 | **Deprecate before removing** | Standardized `@deprecated` header across 80+ files, with target removal version |
| 4 | **Never break existing callers** | Re-exports, aliases, default values, and fallback logic throughout the codebase |
| 5 | **Upgrade path is non-negotiable** | v1.x users must reach v1.7+ before upgrading to v2.x |
| 6 | **Every PR must consider compatibility** | PR template with breaking changes section and upgrade flow checklist item |
| 7 | **Test backward compatibility explicitly** | Upgrade path tests, MCP mode fallback tests, provider fallback tests, migration tests |
| 8 | **Feature freeze during major refactors** | `main` branch code freeze; deprecation headers block new features on files slated for removal |
