# GitHub Workflows Analysis for Cherry Studio

This document explains each GitHub Actions workflow in the repository and the key knowledge contained within them.

---

## 1. CI & Quality Assurance

### `ci.yml` — Continuous Integration

**Trigger:** Push to `main`, PR to `main`/`develop`/`v2`, manual dispatch.

**Jobs:**
- **changes** — Uses `dorny/paths-filter` to detect which parts of the codebase changed (main process, renderer, shared). This enables conditional test execution.
- **changeset-check** — Validates that changesets are present for package changes (via `pnpm changeset status`), ensuring version bumps follow the changeset workflow.
- **basic-checks** — Runs lint, format, type-check, i18n check, hardcoded strings check, OpenAPI spec check, and skills check. This is the gatekeeper for code quality.
- **general-test** — Runs main process, AI core, shared package, and script tests (conditional on changes).
- **render-test** — Runs renderer tests (conditional on changes).
- **notify** — Sends a Feishu notification when CI fails on `main` branch pushes.

**Key Knowledge:**
- Path-based change detection (`dorny/paths-filter`) to skip unnecessary test jobs.
- All checks must pass before merging — this is the project's primary quality gate.
- Feishu notifications integrated directly into CI failure reporting.

---

### `ci-rerun-on-base-change.yml` — Re-run CI on Base Branch Change

**Trigger:** PR `edited` event when the base branch changes.

**Purpose:** If a PR's target branch is changed (e.g., from `develop` to `main`), this workflow automatically re-runs the latest CI workflow for that PR's HEAD SHA.

**Key Knowledge:**
- Uses `actions/github-script` to find and re-run the latest completed CI run via the GitHub API.
- Ensures CI results remain valid when a PR is retargeted to a different base branch.

---

### `pr-description-check.yml` — PR Description Check

**Trigger:** Non-draft PRs against `main`/`develop`/`v2` on `opened`, `edited`, `synchronize`, `ready_for_review`.

**Jobs:**
- **check-description** — Validates the PR description has:
  - `Before this PR:` section (non-empty)
  - `After this PR:` section (non-empty)
  - `### Why we need it` section (not just template defaults)
  - `release-note` block (non-empty)
  - Posts a bot comment listing failures, or deletes the previous failure comment when fixed.
- **notify-sync** — When new commits are pushed (`synchronize`), reminds the author to review their PR description.

**Key Knowledge:**
- PR descriptions must follow a strict template with specific headers.
- Bilingual comments (Chinese/English) are used for user-facing messages.
- Uses `core.setFailed()` to mark the check as failed, blocking merge via branch protection.

---

## 2. Build & Release

### `release.yml` — Full Application Release

**Trigger:** Manual dispatch (with tag + platform selection), or when a PR from a `release/v*` branch is merged to `main`.

**Matrix:** Cross-platform build on macOS, Windows, Linux.

**Steps:**
- Checkout with full history (`fetch-depth: 0` for version detection).
- Determine tag from manual input or branch name, sync `package.json` version.
- Install deps (pnpm with caching).
- Build per platform with platform-specific env vars (Apple codesign, etc.).
- Publish via `ncipollo/release-action` as a **draft release** with all artifacts.

**Key Knowledge:**
- Release flow: PR from `release/v*` → merge to `main` → auto-trigger build → draft release.
- Uses `fromJSON()` to dynamically build the OS matrix based on input.
- Apple codesigning requires `CSC_LINK`, `CSC_KEY_PASSWORD`, `APPLE_ID`, etc.
- Cross-platform artifact glob pattern: `dist/*.exe,dist/*.zip,dist/*.dmg,dist/*.AppImage,...`

---

### `release-packages.yml` — NPM Package Release

**Trigger:** Push to `main` with changes in `.changeset/` or specific package dirs, or manual dispatch.

**Purpose:** Uses `changesets/action` to create version PRs or publish packages to npm.

**Monorepo packages:** `aiCore`, `ai-sdk-provider`, `extension-table-plus`.

**Key Knowledge:**
- Changesets-driven versioning — PRs must include `.changeset/*.md` files.
- `changeset publish` publishes to npm; `changeset version` bumps versions.
- Requires `NPM_TOKEN` and `NODE_AUTH_TOKEN` for npm publishing.

---

### `nightly-build.yml` — Nightly Unstable Build

**Trigger:** Scheduled 17:00 UTC (01:00 Beijing time) daily; also manual.

**Purpose:** Builds the latest `main` branch into preview artifacts with a `cherry-studio-nightly-{date}-{os}` naming scheme.

**Key details:**
- Generates SHA256 checksums per platform.
- Artifacts retained for 3 days only.
- Previously had cleanup of old nightly artifacts (now removed from the current version — only `cleanup-artifacts` remains).
- Bilingual summary report (Chinese + English) with warnings that this is an unstable build.

---

### `v2-daily-preview-build.yml` — v2 Branch Daily Preview Build

**Trigger:** Scheduled 11:00 UTC (19:00 Beijing time); manual dispatch.

**Restricted to:** Only runs in `CherryHQ/cherry-studio` (upstream repo).

**Purpose:** Builds the `v2` branch with:
- A `check-repository` guard that ensures only the upstream repo runs.
- Patching of app identity (appId, productName, notarization config) to use the `cherrystudio` prefix instead of `kangfenmao`.
- Cross-platform build with all artifact formats.
- Feishu notification on completion.
- Old preview artifacts automatically cleaned after 14 days.

**Key Knowledge:**
- `check-repository` pattern is used to prevent forks from wasting CI resources.
- App identity patching is critical: changes `appId`, `productName`, and references to the old `com.kangfenmao.CherryStudio` across multiple files.
- Uses `fail-fast: false` so one platform failure doesn't cancel others.
- `build-summary` job runs regardless of build result (`always()`).

---

### `snapshot.yml` — Snapshot NPM Release

**Trigger:** Manual dispatch with a branch parameter.

**Purpose:** Publishes snapshot (pre-release) versions of packages to npm with the `snapshot` dist-tag.

**Steps:**
- Branch checkout → build → `changeset version --snapshot snapshot` → `changeset publish --tag snapshot`.

**Key Knowledge:**
- Snapshot releases allow testing of in-development packages without polluting the `latest` tag.
- Install via `pnpm add @cherrystudio/ai-core@snapshot`.

---

### `prepare-release.yml` — AI-Driven Release Preparation

**Trigger:** Manual dispatch with version bump type.

**Purpose:** Uses Claude Code (via `anthropics/claude-code-action`) to execute the `prepare-release` skill from `.agents/skills/prepare-release/SKILL.md`.

**Key Knowledge:**
- The workflow itself is thin — it delegates to an AI agent that reads skill instructions and performs version bumping, changelog generation, branch creation, and PR creation.
- Requires `CLAUDE_TRANSLATOR_APIKEY` and `TOKEN_GITHUB_WRITE`.
- This is a key example of **AI-augmented CI/CD**: the AI reads local skill files, runs git commands, creates branches/PRs autonomously.

---

## 3. Code Review & AI Assistance

### `claude-code-review.yml` — Automated Code Review

**Trigger:** Non-draft PRs opened (only from same-repo branches due to OIDC constraints).

**Purpose:** Claude reviews the PR for code quality, bugs, performance, security, and test coverage, then posts a comment via `gh pr comment`.

**Key Knowledge:**
- Restricted to same-repo PRs (`github.event.pull_request.head.repo.full_name == github.repository`) due to upstream OIDC issues.
- Uses `anthropics/claude-code-action` with `claude_code_oauth_token`.
- The prompt tells Claude to use `CLAUDE.md` for style/convention guidance.

---

### `claude.yml` — On-Demand Claude Code

**Trigger:** When a collaborator/owner comments `@claude` on an issue, PR review, or PR review comment.

**Permission level:** Restricted to `COLLABORATOR`, `MEMBER`, or `OWNER` associations.

**Purpose:** Allows team members to summon Claude to perform tasks directly on GitHub via natural language commands in comments.

**Key Knowledge:**
- Uses OAuth token (`CLAUDE_CODE_OAUTH_TOKEN`) for authentication.
- `additional_permissions: actions: read` allows Claude to read CI results on PRs.

---

### `claude-translator.yml` — Automatic Translation Bot

**Trigger:** Issues opened, comments/reviews created or edited (excluding bot-authored).

**Purpose:** Automatically translates non-English issue/comment/review content to English, appending the original content in a collapsible `<details>` block.

**Key Knowledge:**
- Concurrency group prevents duplicate translation runs on the same item.
- Smart detection: skips if content is already English; re-translates if existing translation doesn't match original.
- Handles four event types: `issues`, `issue_comment`, `pull_request_review`, `pull_request_review_comment`.
- Uses a specific GitHub token (`TOKEN_GITHUB_WRITE`) with write permissions to update comments.
- Format: `[!NOTE]` prefix + translation + `---` separator + collapsible `<details>` with original.

---

## 4. Issue/PR Management

### `issue-management.yml` — Stale Issue Management

**Trigger:** Daily schedule + manual dispatch.

**Purpose:** Manages stale issues using `actions/stale`:
1. **`needs-more-info` labeled issues** — Marked stale after 30 days, closed immediately after.
2. **Inactive issues** — Marked stale after 30 days, closed after 10 more days.
3. PRs are completely excluded from this workflow.

**Key Knowledge:**
- Exempt labels: `pending`, `Dev Team`, `kind/enhancement`, and any milestone/assignee.
- Bilingual stale messages (English + Chinese).
- `operations-per-run: 1000` for batch processing.

---

### `github-issue-tracker.yml` — Issue Feishu Notifications

**Trigger:** New issue opened, daily schedule (08:30 Beijing time), manual dispatch.

**Purpose:** Sends issue notifications to Feishu (Lark) via Claude-generated summaries.

**Two jobs:**
- **process-new-issue** — When an issue is opened, checks if it's during quiet hours (00:00-08:30 Beijing time). If so, adds a `pending-feishu-notification` label. Otherwise, Claude summarizes and sends notification immediately.
- **process-pending-issues** — Runs on schedule, processes all issues with the pending label, sends summaries via Feishu, then removes the label.

**Key Knowledge:**
- Uses Claude to generate concise Chinese summaries.
- Integrates Feishu (Lark) webhook notifications with custom scripts (`scripts/feishu-notify.ts`).
- Quiet hours mechanism prevents notification spam during off-hours.

---

## 5. Sync & Configuration

### `sync-to-gitcode.yml` — Sync Release to GitCode Mirror

**Trigger:** Release published, or manual dispatch with tag.

**Purpose:** Builds Windows binaries with code signing and syncs the full release to GitCode (a Chinese code hosting platform).

**Key Knowledge:**
- Runs on a **self-hosted Windows signing runner** (`[self-hosted, windows-signing]`).
- Code signing uses `CHERRY_CERT_PATH`, `CHERRY_CERT_KEY`, `CHERRY_CERT_CSP`.
- Downloads GitHub release assets, replaces Windows `.exe` files with signed versions, then uploads everything to GitCode via its API.
- Upload retry logic (3 retries per file).
- Failure notification to Feishu on error/cancellation.
- Essential for Chinese users who cannot easily access GitHub releases.

---

### `update-app-upgrade-config.yml` — Update App Upgrade Configuration

**Trigger:** Release published/prereleased, or manual dispatch.

**Purpose:** Maintains an external `app-upgrade-config.json` on the `x-files/app-upgrade-config` branch, which controls in-app update prompts.

**Logic:**
- Determines if the release is the latest stable version or a prerelease.
- Only proceeds for: (a) the latest stable release, (b) beta/rc prereleases, or (c) manual dispatch with a valid tag.
- Checks out two branches: default (for scripts) and `x-files/app-upgrade-config` (for the config file).
- Runs `scripts/update-app-upgrade-config.ts` to update the JSON.
- Commits and pushes changes to `x-files/app-upgrade-config` if differences exist.

**Key Knowledge:**
- The upgrade config is stored on a separate orphan branch (`x-files/app-upgrade-config`), keeping it independent of the main codebase.
- Tag validation: prereleases must contain `-beta` or `-rc` suffix; non-latest stable releases are skipped.
- This enables the Electron app to check for updates from a lightweight JSON file rather than querying GitHub Releases directly.

---

### `dispatch-docs-update.yml` — Trigger Docs Site Update

**Trigger:** Release `released` event.

**Purpose:** Sends a repository dispatch event to `CherryHQ/cherry-studio-docs` with the release tag, triggering a docs site update for download version info.

**Key Knowledge:**
- Uses `peter-evans/repository-dispatch` with a cross-repo PAT (`REPO_DISPATCH_TOKEN`).
- Decouples the main app release from docs site updates.

---

## 6. Internationalization

### `auto-i18n.yml` — Daily Auto I18N Sync

**Trigger:** Daily schedule (00:00 UTC / 08:00 Beijing time) + manual dispatch.

**Purpose:** Automatically syncs and translates localization files daily.

**Steps:**
- Checkout → install deps → `pnpm i18n:sync && pnpm i18n:translate` → format.
- Resets `package.json` and `pnpm-lock.yaml` changes (these shouldn't be included).
- If changes exist, creates an automated PR (`auto-i18n-daily-{run_id}` branch) with title `"🤖 Daily Auto I18N Sync: {date}"`.
- Failure notification to Feishu.

**Key Knowledge:**
- Translation is powered by an external API (`TRANSLATE_API_KEY`), defaulting to `deepseek/deepseek-v3.1` model via `https://api.ppinfra.com/openai`.
- Base locale is `en-us` (i.e., English is the source of truth for translations).
- PRs are auto-generated but require human review before merging.
- `git reset -- package.json pnpm-lock.yaml` prevents unintended dependency changes from being committed.
- Uses environment variables from both `secrets` and `vars` for flexible configuration.

---

## Cross-Cutting Patterns & Insights

### Infrastructure Patterns
- **pnpm caching** is used consistently across all workflows for dependency install speed.
- **Feishu (Lark) notifications** are deeply integrated (CI failures, releases, issues, i18n failures).
- **Node.js version** is managed via `.node-version` file for consistency.
- **Cross-platform matrix builds** use `fail-fast: false` for maximum signal.

### Security & Access Control
- **`check-repository` guard** prevents forks from running resource-intensive workflows.
- **Token separation**: different GitHub tokens for different permission levels (`GITHUB_TOKEN`, `TOKEN_GITHUB_WRITE`, `REPO_DISPATCH_TOKEN`).
- **Self-hosted runner** for Windows code signing (certificate security).

### AI Integration
- Three workflows use Anthropic Claude (`claude.yml`, `claude-translator.yml`, `claude-code-review.yml`).
- `prepare-release.yml` uses Claude autonomously to execute release preparation.
- `github-issue-tracker.yml` uses Claude for issue summarization.
- Multiple AI workflows use the shared secret `CLAUDE_TRANSLATOR_APIKEY` and base URL `CLAUDE_TRANSLATOR_BASEURL`.

### Release Workflow Sequence
1. Developer runs `prepare-release` (manual dispatch) → Claude creates release branch + PR.
2. PR merges to `main` → `release.yml` builds and creates a draft GitHub Release.
3. Release published → `update-app-upgrade-config.yml` updates upgrade config.
4. Release published → `sync-to-gitcode.yml` signs Windows binaries and mirrors to GitCode.
5. Release published → `dispatch-docs-update.yml` triggers docs site update.
