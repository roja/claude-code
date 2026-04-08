# Services Layer & Utilities Architecture

## Document Scope

This document provides a deep technical reference for every service module under `src/services/` and the key utility subsystems under `src/utils/`. It targets principal/senior engineers and AI scientists who need to understand internal architecture, API surfaces, data flows, error handling, and cross-service dependencies.

---

## Table of Contents

1. [Service Directory Overview](#1-service-directory-overview)
2. [API Service (`services/api/`)](#2-api-service)
3. [Compact Service (`services/compact/`)](#3-compact-service)
4. [LSP Service (`services/lsp/`)](#4-lsp-service)
5. [MCP Service (`services/mcp/`)](#5-mcp-service)
6. [Policy Limits Service (`services/policyLimits/`)](#6-policy-limits-service)
7. [Remote Managed Settings (`services/remoteManagedSettings/`)](#7-remote-managed-settings)
8. [OAuth Service (`services/oauth/`)](#8-oauth-service)
9. [Analytics Service (`services/analytics/`)](#9-analytics-service)
10. [Session Memory (`services/SessionMemory/`)](#10-session-memory)
11. [Tools Orchestration (`services/tools/`)](#11-tools-orchestration)
12. [Other Services](#12-other-services)
13. [Utility Subsystems (`utils/`)](#13-utility-subsystems)
14. [Service Dependency Diagram](#14-service-dependency-diagram)

---

## 1. Service Directory Overview

```
src/services/
  AgentSummary/          # Agent task summary generation
  analytics/             # Datadog, GrowthBook, first-party event logging
  api/                   # Anthropic API client, retry, streaming, errors, files
  autoDream/             # Auto-dream consolidation (memory consolidation)
  compact/               # Context compaction (auto, micro, session-memory)
  extractMemories/       # Memory extraction from conversations
  lsp/                   # Language Server Protocol integration
  MagicDocs/             # Magic docs generation
  mcp/                   # Model Context Protocol (documented separately)
  oauth/                 # OAuth 2.0 PKCE flow
  plugins/               # Plugin installation and CLI commands
  policyLimits/          # Organization policy restrictions
  PromptSuggestion/      # Prompt suggestion and speculation
  remoteManagedSettings/ # Enterprise remote settings sync
  SessionMemory/         # Session memory (background note-taking)
  settingsSync/          # Settings synchronization
  teamMemorySync/        # Team memory synchronization with secret scanning
  tips/                  # Tip scheduler and registry
  tools/                 # Tool execution, orchestration, streaming
  toolUseSummary/        # Tool use summary generation

  # Top-level service files
  awaySummary.ts         # Away-mode summary generation
  claudeAiLimits.ts      # Claude.ai rate limit tracking/display
  diagnosticTracking.ts  # Diagnostic event tracking
  internalLogging.ts     # Internal log sink
  mcpServerApproval.tsx  # MCP server approval UI
  mockRateLimits.ts      # Rate limit mocking (testing)
  notifier.ts            # Desktop/terminal notifications
  preventSleep.ts        # Prevent system sleep during long operations
  rateLimitMessages.ts   # Rate limit message formatting
  tokenEstimation.ts     # Token counting via API/Bedrock/Vertex
  vcr.ts                 # Request recording/playback for testing
  voice.ts               # Voice input support
  voiceKeyterms.ts       # Voice keyterm detection
  voiceStreamSTT.ts      # Voice streaming speech-to-text
```

---

## 2. API Service

**Path:** `src/services/api/`

### Purpose

The API service is the primary interface between Claude Code and the Anthropic Messages API. It manages client construction for multiple providers (Direct, Bedrock, Vertex, Foundry), retry logic with exponential backoff, streaming, error classification, rate limit handling, and session persistence.

### File Inventory

| File | Purpose |
|------|---------|
| `client.ts` | Client factory -- constructs `Anthropic` SDK instances for all providers |
| `claude.ts` | Core message-sending logic: `queryClaudeAPI()`, streaming, tool dispatch |
| `withRetry.ts` | Retry engine: exponential backoff, 529 handling, fast-mode fallback |
| `errors.ts` | Error classification, rate limit messages, prompt-too-long detection |
| `errorUtils.ts` | Low-level error detail extraction (connection errors) |
| `bootstrap.ts` | Bootstrap API: fetches client_data and additional model options at startup |
| `usage.ts` | Utilization API: fetches 5-hour/7-day rate limit utilization |
| `filesApi.ts` | Files API client: download/upload file attachments (BYOC mode) |
| `sessionIngress.ts` | Session transcript persistence via append-only log |
| `grove.ts` | Grove privacy settings (consumer terms notification) |
| `dumpPrompts.ts` | Prompt dumping for debugging |
| `logging.ts` | API request/response logging |
| `referral.ts` | Referral tracking |
| `adminRequests.ts` | Admin API requests |
| `overageCreditGrant.ts` | Overage credit grant handling |
| `ultrareviewQuota.ts` | Ultra review quota management |
| `metricsOptOut.ts` | Metrics opt-out handling |
| `promptCacheBreakDetection.ts` | Detects when prompt cache is broken |
| `firstTokenDate.ts` | Tracks first token date |
| `emptyUsage.ts` | Empty usage sentinel |

### Client Construction (`client.ts`)

The `getAnthropicClient()` function is an async factory that returns an `Anthropic` SDK instance configured for the current provider:

```
Provider selection (env vars):
  CLAUDE_CODE_USE_BEDROCK  -> AnthropicBedrock (AWS)
  CLAUDE_CODE_USE_FOUNDRY  -> AnthropicFoundry (Azure)
  CLAUDE_CODE_USE_VERTEX   -> AnthropicVertex (GCP)
  (default)                -> Anthropic (first-party)
```

**Authentication flow:**
1. Checks and refreshes OAuth token if needed
2. For non-Claude.ai subscribers: configures API key headers (env var or apiKeyHelper)
3. Injects custom headers from `ANTHROPIC_CUSTOM_HEADERS`
4. Injects session ID, container ID, remote session ID headers
5. Wraps fetch to inject `x-client-request-id` UUID for request correlation
6. Configures proxy settings via `getProxyFetchOptions()`

**Provider-specific auth:**
- **Bedrock:** AWS credentials (cached + refreshed), region override for small-fast model, `AWS_BEARER_TOKEN_BEDROCK` for API key auth, `CLAUDE_CODE_SKIP_BEDROCK_AUTH` for proxy scenarios
- **Vertex:** Google Auth with service account or ADC, project ID fallback to avoid 12s metadata server timeout, `CLAUDE_CODE_SKIP_VERTEX_AUTH` for mock auth
- **Foundry:** Azure AD token provider via `DefaultAzureCredential`, or `ANTHROPIC_FOUNDRY_API_KEY`

### Retry Engine (`withRetry.ts`)

The `withRetry<T>()` async generator yields `SystemAPIErrorMessage` events during waits, allowing the UI to show retry status.

**Retry strategy:**
- Default 10 retries (`CLAUDE_CODE_MAX_RETRIES` override)
- Exponential backoff: 500ms base, 2x growth, 32s max (configurable)
- Jitter: 25% of base delay added randomly
- Retry-After header honored when present

**Error classification for retry decisions:**
- **529 (Overloaded):** Retried for foreground sources only (main thread, SDK, compact, hooks, side questions, security classifiers). Background sources (summaries, titles, suggestions) bail immediately to avoid gateway amplification. Max 3 consecutive 529s before triggering model fallback.
- **429 (Rate Limited):** Retried for API key and Enterprise users. Consumer Claude.ai subscribers do not retry 429 (limits are hourly). `x-should-retry` header from server is respected.
- **401/403:** Trigger client re-creation with fresh credentials (OAuth refresh, AWS/GCP credential cache clear)
- **408/409:** Retried (timeout/lock)
- **5xx:** Retried
- **ECONNRESET/EPIPE:** Disable keep-alive and retry with fresh connection
- **400 (context overflow):** Adjust `max_tokens` downward and retry

**Fast mode fallback:**
When fast mode is active and a 429/529 occurs:
- Short retry-after (<20s): sleep and retry with fast mode still active (preserves prompt cache)
- Long retry-after (>=20s): enter cooldown (10min minimum), switch to standard speed
- Overage rejection: permanently disable fast mode

**Persistent retry mode (`CLAUDE_CODE_UNATTENDED_RETRY`):**
For unattended sessions (ant-only), 429/529 retries are infinite with 5min max backoff, 6hr reset cap, and 30s heartbeat yields to prevent idle timeouts.

### Session Ingress (`sessionIngress.ts`)

Provides durable session transcript persistence using optimistic concurrency control:

- **Append protocol:** PUT with `Last-Uuid` header for ordering. UUID chain ensures no gaps.
- **Conflict resolution (409):** Adopts server's `x-last-uuid` header and retries. Falls back to re-fetching the full log to discover the chain head.
- **Sequential execution:** Per-session `sequential()` wrapper prevents concurrent writes.
- **Teleport events:** Paginated GET from Sessions API (v2), replacing session-ingress. Handles Spanner and threadstore backends.

### Files API (`filesApi.ts`)

Manages file attachments for BYOC (Bring Your Own Cloud) mode:

- **Download:** `downloadFile()` with retry, `downloadSessionFiles()` with parallel concurrency (default 5)
- **Upload:** `uploadFile()` with multipart form data, 500MB limit, `uploadSessionFiles()` for batch
- **Path safety:** `buildDownloadPath()` normalizes paths and rejects traversal (`..`)
- **List:** `listFilesCreatedAfter()` with pagination via `after_id` cursor

### Bootstrap (`bootstrap.ts`)

Called at startup to fetch `client_data` and `additional_model_options` from `/api/claude_cli/bootstrap`. Supports both OAuth and API key auth. Results are persisted to disk cache and only written if changed (avoids unnecessary disk I/O).

### Error Handling Patterns

1. **Fail-open for non-critical services:** Bootstrap, usage, grove all swallow errors
2. **Fail-closed for auth:** 401/403 trigger credential refresh, then retry
3. **Context overflow recovery:** Automatically adjusts max_tokens when input exceeds context window
4. **Model fallback:** After 3 consecutive 529 errors, falls back to configured fallback model

---

## 3. Compact Service

**Path:** `src/services/compact/`

### Purpose

The compact service manages conversation context when it approaches the model's context window limit. It provides three tiers of compaction, from lightweight to heavy, to maintain conversation quality while staying within token limits.

### File Inventory

| File | Purpose |
|------|---------|
| `compact.ts` | Core compaction logic: `compactConversation()` |
| `autoCompact.ts` | Auto-compaction trigger logic and threshold calculations |
| `microCompact.ts` | Lightweight compaction: clears old tool results |
| `apiMicrocompact.ts` | API-level context management (cache editing) |
| `prompt.ts` | Compaction prompt templates and formatting |
| `grouping.ts` | Message grouping for partial compaction |
| `sessionMemoryCompact.ts` | Session memory-based compaction alternative |
| `postCompactCleanup.ts` | Post-compaction cleanup tasks |
| `compactWarningHook.ts` | Hook for context window warnings |
| `compactWarningState.ts` | Warning suppression state management |
| `timeBasedMCConfig.ts` | Time-based microcompact configuration |

### Three-Tier Compaction Architecture

```
Tier 1: Microcompact (lightweight, no API call for legacy; cache_edits for cached)
  |-- Time-based: Clears old tool results when gap > threshold minutes
  |-- Cached MC: Uses cache_edits API to delete tool results without breaking cache
  |-- Legacy MC: (removed) Direct content replacement
  |
Tier 2: Session Memory Compaction (medium weight)
  |-- Uses session memory's markdown notes to reconstruct context
  |-- Preserves recent messages verbatim
  |
Tier 3: Full Compaction (heavy, API call required)
  |-- Runs forked agent to summarize entire conversation
  |-- Produces structured summary with 9 sections
  |-- Replaces all messages with summary + recent messages
```

### Auto-Compact Trigger (`autoCompact.ts`)

**Threshold calculation:**
```
effectiveContextWindow = contextWindow - max(maxOutputTokens, 20K)
autoCompactThreshold   = effectiveContextWindow - 13K buffer
warningThreshold       = threshold - 20K
errorThreshold         = threshold - 20K
blockingLimit          = effectiveContextWindow - 3K
```

**Circuit breaker:** After 3 consecutive compaction failures, auto-compact stops retrying for the session. This prevents runaway API calls when context is irrecoverably over the limit.

**Guards:**
- Disabled for `session_memory` and `compact` query sources (prevents recursion)
- Disabled when context collapse is enabled (it owns headroom management)
- Disabled in reactive-only mode (let API's prompt-too-long trigger compact)
- Respects `DISABLE_COMPACT` and `DISABLE_AUTO_COMPACT` env vars
- `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` for testing with percentage thresholds

### Microcompact (`microCompact.ts`)

**Time-based microcompact:**
When the gap since the last assistant message exceeds a configurable threshold (cache is cold), content-clears all but the most recent N compactable tool results. Mutates message content directly since there's no cached prefix to preserve.

**Cached microcompact (cache editing):**
Uses the `cache_edits` API to delete old tool results without invalidating the cached prefix. Tracks tool results globally, groups them by user message, and queues `cache_edits` blocks for the API layer. Only runs for the main thread (sub-agents are excluded to prevent global state corruption).

**Compactable tools:** FileRead, Bash/Shell, Grep, Glob, WebSearch, WebFetch, FileEdit, FileWrite.

### Compaction Prompts (`prompt.ts`)

Three prompt variants:
- **Base compact:** Summarizes the entire conversation into 9 structured sections
- **Partial compact (from):** Summarizes only recent messages after a retained prefix
- **Partial compact (up_to):** Summarizes a prefix that will be followed by retained recent messages

All prompts include a `NO_TOOLS_PREAMBLE` that aggressively blocks tool use (the model sometimes attempts tool calls despite being told not to, especially on Sonnet 4.6+). An `<analysis>` scratchpad block improves summary quality and is stripped by `formatCompactSummary()` before the summary enters context.

### Dependencies
- `utils/forkedAgent.ts` - runs compaction as a forked sub-agent
- `utils/tokens.ts` - token counting
- `services/api/promptCacheBreakDetection.ts` - cache break tracking
- `services/SessionMemory/` - session memory compaction path

---

## 4. LSP Service

**Path:** `src/services/lsp/`

### Purpose

Integrates Language Server Protocol servers to provide diagnostic feedback (errors, warnings), hover information, go-to-definition, and other IDE-like features within Claude Code sessions. LSP servers are exclusively provided by plugins.

### File Inventory

| File | Purpose |
|------|---------|
| `LSPClient.ts` | LSP client wrapper over vscode-jsonrpc stdio transport |
| `LSPServerInstance.ts` | Manages a single LSP server lifecycle |
| `LSPServerManager.ts` | Manages multiple LSP server instances |
| `manager.ts` | Singleton lifecycle: init, reinit, shutdown |
| `config.ts` | Configuration loader (from plugins) |
| `LSPDiagnosticRegistry.ts` | Async diagnostic collection and deduplication |
| `passiveFeedback.ts` | Registers notification handlers on servers |

### Architecture

```
Plugins
  |
  v
config.ts (getAllLspServers)  -->  LSPServerManager (singleton via manager.ts)
                                    |
                                    +-- LSPServerInstance (per server)
                                    |     |
                                    |     +-- LSPClient (vscode-jsonrpc over stdio)
                                    |     |     |
                                    |     |     +-- child_process (LSP server binary)
                                    |     |
                                    |     +-- diagnostics --> LSPDiagnosticRegistry
                                    |
                                    +-- passiveFeedback.ts (notification handlers)
```

### LSP Client (`LSPClient.ts`)

The `createLSPClient()` factory returns an object that:
- Spawns the LSP server as a child process via `child_process.spawn()`
- Creates a `MessageConnection` using `StreamMessageReader/StreamMessageWriter` over stdio
- Supports handler registration before connection is ready (queued)
- Detects crashes and invokes `onCrash` callback for restart support
- Traces communication when debug logging is enabled

### Manager Singleton (`manager.ts`)

**Lifecycle states:** `not-started` -> `pending` -> `success` | `failed`

- `initializeLspServerManager()`: Called during startup. Creates manager, starts async initialization (non-blocking). Skipped in `--bare` mode.
- `reinitializeLspServerManager()`: Called on plugin refresh. Shuts down old instance, creates new one. Uses generation counter to invalidate stale init promises.
- `shutdownLspServerManager()`: Called during shutdown. State always cleared even if shutdown fails.
- `isLspConnected()`: Returns true if at least one server is healthy.

### Diagnostic Registry (`LSPDiagnosticRegistry.ts`)

Stores diagnostics received asynchronously from LSP servers via `textDocument/publishDiagnostics`:

1. `registerPendingLSPDiagnostic()` stores incoming diagnostics
2. `checkForLSPDiagnostics()` retrieves pending, deduplicates, volume-limits
3. Diagnostics are delivered as attachments in the next query

**Deduplication:** Both within-batch and cross-turn, using LRU cache (500 files max). Diagnostics are keyed by message + severity + range + source + code.

**Volume limiting:** Max 10 diagnostics per file, 30 total. Sorted by severity (errors first).

---

## 5. MCP Service

**Path:** `src/services/mcp/`

Cross-reference: The MCP service is documented in detail in a separate document. Key files for reference:

| File | Purpose |
|------|---------|
| `MCPConnectionManager.tsx` | Connection lifecycle management |
| `client.ts` | MCP client protocol implementation |
| `config.ts` | Configuration loading and merging |
| `InProcessTransport.ts` | In-process MCP server transport |
| `SdkControlTransport.ts` | SDK control protocol transport |
| `auth.ts` | MCP server authentication |
| `channelPermissions.ts` | Channel-based permission management |
| `channelAllowlist.ts` | Tool allowlisting per channel |
| `envExpansion.ts` | Environment variable expansion in MCP configs |
| `normalization.ts` | Config normalization |

---

## 6. Policy Limits Service

**Path:** `src/services/policyLimits/`

### Purpose

Fetches organization-level policy restrictions from the API and uses them to disable CLI features. This is the enforcement mechanism for enterprise admin controls (e.g., disabling remote sessions, product feedback).

### Public API

```typescript
// Check if a specific policy is allowed (sync, fail-open)
isPolicyAllowed(policy: string): boolean

// Load policy limits during initialization (async, non-blocking)
loadPolicyLimits(): Promise<void>

// Wait for initial loading to complete (with 30s timeout)
waitForPolicyLimitsToLoad(): Promise<void>

// Refresh after auth state changes
refreshPolicyLimits(): Promise<void>

// Check eligibility (no circular deps)
isPolicyLimitsEligible(): boolean
```

### Architecture

**Eligibility:**
- First-party API provider with first-party base URL
- Console users (API key): all eligible
- OAuth users: only Team and Enterprise subscribers

**Fetch with caching:**
1. Load cached restrictions from `~/.claude/policy-limits.json`
2. Compute SHA-256 checksum of cached restrictions
3. Fetch with `If-None-Match` header (ETag-style caching)
4. Handle 304 (cache valid), 404 (no restrictions), 200 (new data)
5. Save to file cache on change
6. Retry with exponential backoff (5 retries max)

**Fail-open design:**
- `isPolicyAllowed()` returns `true` if cache is unavailable
- Exception: `allow_product_feedback` fails closed in essential-traffic-only mode (HIPAA compliance)

**Background polling:** Hourly interval to pick up mid-session changes. Uses `registerCleanup()` for shutdown.

### Types (`types.ts`)

```typescript
type PolicyLimitsResponse = {
  restrictions: Record<string, { allowed: boolean }>
}
```

---

## 7. Remote Managed Settings

**Path:** `src/services/remoteManagedSettings/`

### Purpose

Manages enterprise-administered settings that override local configuration. Follows the same patterns as policy limits (fail-open, checksum caching, background polling) but delivers full `SettingsJson` objects instead of boolean restrictions.

### File Inventory

| File | Purpose |
|------|---------|
| `index.ts` | Main service: fetch, cache, poll, security check |
| `types.ts` | Response schema and fetch result types |
| `syncCache.ts` | Eligibility checks, cache reset |
| `syncCacheState.ts` | Session cache state, file I/O |
| `securityCheck.tsx` | Security validation of settings changes |

### Public API

```typescript
loadRemoteManagedSettings(): Promise<void>
waitForRemoteManagedSettingsToLoad(): Promise<void>
refreshRemoteManagedSettings(): Promise<void>
clearRemoteManagedSettingsCache(): Promise<void>
isEligibleForRemoteManagedSettings(): boolean
computeChecksumFromSettings(settings: SettingsJson): string
```

### Architecture

**Fetch protocol:**
1. Cache-first: if disk cache exists, apply immediately and unblock waiters
2. Fetch from `/api/claude_code/settings` with checksum-based ETag
3. Validate response against `RemoteManagedSettingsResponseSchema` (zod)
4. Validate settings structure against `SettingsSchema`
5. Security check: `checkManagedSettingsSecurity()` reviews dangerous changes (e.g., new allowed commands). User can reject.
6. Persist to `~/.claude/remote-managed-settings.json` with `fsync`
7. Trigger hot-reload via `settingsChangeDetector.notifyChange('policySettings')`

**Checksum computation:**
Must match Python server's `json.dumps(sort_keys=True, separators=(",", ":"))` followed by SHA-256. Keys are recursively sorted for consistency.

**Background polling:** Hourly. Compares serialized settings before/after to detect changes and trigger hot-reload.

---

## 8. OAuth Service

**Path:** `src/services/oauth/`

### Purpose

Implements OAuth 2.0 Authorization Code flow with PKCE for authenticating with Claude.ai and the Anthropic Console.

### File Inventory

| File | Purpose |
|------|---------|
| `index.ts` | `OAuthService` class: orchestrates the full flow |
| `client.ts` | URL building, token exchange, profile fetching, scope parsing |
| `crypto.ts` | PKCE code verifier/challenge generation, state generation |
| `auth-code-listener.ts` | Local HTTP server for OAuth callback |
| `getOauthProfile.ts` | Profile data fetching |

### Flow

```
1. OAuthService.startOAuthFlow()
   |
   +-- AuthCodeListener.start()    (bind localhost:random_port)
   +-- crypto.generateCodeVerifier() + generateCodeChallenge()
   +-- client.buildAuthUrl()        (manual + automatic variants)
   |
   +-- openBrowser(automaticFlowUrl)
   +-- authURLHandler(manualFlowUrl)  (display to user)
   |
   +-- waitForAuthorizationCode()
   |     |-- AuthCodeListener receives callback with code
   |     \-- OR user pastes code manually
   |
   +-- client.exchangeCodeForTokens()
   +-- client.fetchProfileInfo()
   |
   +-- Return OAuthTokens { accessToken, refreshToken, expiresAt, scopes,
   |                        subscriptionType, rateLimitTier, tokenAccount }
   |
   +-- AuthCodeListener.close()
```

**SDK integration:** `skipBrowserOpen` option lets SDK consumers handle URL opening themselves via the control protocol.

**Token expiry check:** `isOAuthTokenExpired()` is used throughout the codebase to skip API calls when tokens are known to be expired.

---

## 9. Analytics Service

**Path:** `src/services/analytics/`

### Purpose

Provides event logging infrastructure routing to Datadog and first-party event storage. Zero-dependency design prevents import cycles.

### File Inventory

| File | Purpose |
|------|---------|
| `index.ts` | Public API: `logEvent()`, marker types for PII verification |
| `sink.ts` | Event routing: queue, dequeue, fan-out to backends |
| `sinkKillswitch.ts` | Emergency killswitch for analytics |
| `datadog.ts` | Datadog StatsD integration |
| `firstPartyEventLogger.ts` | First-party event logger |
| `firstPartyEventLoggingExporter.ts` | Exports events to first-party storage |
| `metadata.ts` | Event metadata enrichment (tool details, file extensions) |
| `config.ts` | Analytics configuration |
| `growthbook.ts` | GrowthBook feature flag integration |

### Design Principles

1. **No dependencies:** `index.ts` has zero imports to avoid circular dependency issues
2. **Queue-first:** Events are queued until `attachAnalyticsSink()` is called during init
3. **PII safety:** Two marker types enforce compile-time verification:
   - `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` for general storage
   - `AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED` for PII-tagged proto columns
4. **`_PROTO_*` keys:** Special payload prefix routed only to first-party storage (stripped before Datadog)

---

## 10. Session Memory

**Path:** `src/services/SessionMemory/`

### Purpose

Automatically maintains a markdown file with notes about the current conversation. Runs periodically in the background using a forked sub-agent to extract key information without interrupting the main conversation flow.

### File Inventory

| File | Purpose |
|------|---------|
| `sessionMemory.ts` | Core service: scheduling, extraction, file management |
| `prompts.ts` | Prompt templates for memory extraction and updates |
| `sessionMemoryUtils.ts` | Threshold tracking, configuration, initialization state |

### Architecture

Session Memory is a post-sampling hook that fires after each model response:

1. Check if initialization threshold is met (minimum tool calls)
2. Check if update threshold is met (tool calls since last update)
3. Run forked agent with `FileRead` + `FileEdit` tools to read current memory file and update it
4. Agent uses structured prompts to extract: key decisions, file changes, errors, user preferences
5. Memory file stored at the session memory directory (per-project)

**Integration with compaction:** Session memory compaction (`sessionMemoryCompact.ts`) is tried before full compaction in the auto-compact flow. If the memory file has sufficient information, it can reconstruct context without a full LLM summarization call.

---

## 11. Tools Orchestration

**Path:** `src/services/tools/`

### Purpose

Manages tool execution lifecycle: permission checking, execution, progress tracking, analytics, and streaming.

### File Inventory

| File | Purpose |
|------|---------|
| `toolExecution.ts` | Core tool execution: permission check, invoke, result formatting |
| `toolOrchestration.ts` | Multi-tool coordination and sequencing |
| `StreamingToolExecutor.ts` | Streaming tool execution with progress events |
| `toolHooks.ts` | Pre/post tool execution hooks |

### Key Responsibilities

- **Permission checking:** Consults `CanUseToolFn` before execution
- **Speculative classifier:** Starts bash command classification in parallel with permission prompt
- **Analytics:** Logs tool use events with sanitized names, file extensions, git commit tracking
- **Progress tracking:** `ToolProgress` events for UI updates
- **Code edit tracking:** Special handling for `FileEdit`, `FileWrite`, `Bash`, `NotebookEdit`

---

## 12. Other Services

### Token Estimation (`services/tokenEstimation.ts`)

Provides accurate token counting by calling the provider's count-tokens API:
- **First-party:** Uses `Anthropic.beta.messages.countTokens()`
- **Bedrock:** Uses `CountTokensCommand` from `@aws-sdk/client-bedrock-runtime`
- **Vertex:** Uses `Anthropic.beta.messages.countTokens()` with filtered betas
- Handles thinking blocks, tool references, and attachments

### Notifier (`services/notifier.ts`)

Sends desktop notifications via configurable channels:
- `auto`: Platform-specific detection
- `iterm2`: iTerm2 notification protocol
- `terminal_bell`: Terminal bell character
- Custom: Via notification hooks

### Voice Services

- `voice.ts`: Voice input recording and processing
- `voiceStreamSTT.ts`: Streaming speech-to-text with WebSocket
- `voiceKeyterms.ts`: Keyterm detection for voice commands

### Tips (`services/tips/`)

- `tipRegistry.ts`: Registry of available tips
- `tipScheduler.ts`: Scheduling logic for showing tips
- `tipHistory.ts`: Tracks which tips have been shown

### Team Memory Sync (`services/teamMemorySync/`)

- `index.ts`: Main sync service
- `watcher.ts`: File watcher for memory changes
- `secretScanner.ts`: Scans for secrets before syncing
- `teamMemSecretGuard.ts`: Guards against secret leakage

### Auto Dream (`services/autoDream/`)

- `autoDream.ts`: Automatic memory consolidation
- `consolidationPrompt.ts`: Prompts for consolidation
- `consolidationLock.ts`: Lock management for concurrent consolidation
- `config.ts`: Configuration

### Plugins (`services/plugins/`)

- `PluginInstallationManager.ts`: Manages plugin installation lifecycle
- `pluginOperations.ts`: CRUD operations on plugins
- `pluginCliCommands.ts`: CLI command registration for plugins

### Settings Sync (`services/settingsSync/`)

- `index.ts`: Settings synchronization service
- `types.ts`: Sync protocol types

### Claude.ai Limits (`services/claudeAiLimits.ts`)

Tracks and displays rate limit status for Claude.ai subscribers. Parses limit headers from API responses (`anthropic-ratelimit-unified-*`), computes utilization percentages, and generates user-facing messages.

### Rate Limit Messages (`services/rateLimitMessages.ts`)

Formats rate limit errors into actionable user messages with:
- Time until reset
- Link to upgrade plan
- Overage credit status
- Extra usage availability

### Prevent Sleep (`services/preventSleep.ts`)

Uses `caffeinate` (macOS) or equivalent to prevent system sleep during long-running operations.

### VCR (`services/vcr.ts`)

Request recording/playback for deterministic testing. Records API requests and responses, replays them during test execution.

---

## 13. Utility Subsystems

### Settings (`utils/settings/`)

**Path:** `src/utils/settings/`

The settings system implements a layered configuration model with 8 sources merged in precedence order:

```
Sources (lowest to highest precedence):
  1. MDM/HKCU (OS-managed)
  2. Managed file settings (managed-settings.json + drop-ins)
  3. Remote managed settings (enterprise API)
  4. User settings (~/.claude/settings.json)
  5. Project settings (.claude/settings.json)
  6. Project settings (local, .claude/settings.local.json)
  7. Flag settings (CLI --settings)
  8. Flag settings inline (CLI --settings-inline)
```

Key files:
- `settings.ts`: Core `getSettings()` function, source loading, merge logic
- `types.ts`: `SettingsJson` schema, `SettingsSchema` zod validator
- `validation.ts`: Settings validation with error accumulation
- `settingsCache.ts`: Per-source caching with invalidation
- `changeDetector.ts`: Hot-reload via `settingsChangeDetector.notifyChange()`
- `managedPath.ts`: Platform-specific managed settings paths
- `mdm/`: macOS MDM and Windows HKCU registry settings
- `pluginOnlyPolicy.ts`: Policy for plugin-only settings
- `permissionValidation.ts`: Permission rule validation
- `toolValidationConfig.ts`: Tool-specific validation configuration

### Permissions (`utils/permissions/`)

**Path:** `src/utils/permissions/`

Comprehensive permission system controlling tool execution:

- `permissions.ts`: Core permission resolution: matches tool use against rules, returns allow/deny/ask
- `PermissionMode.ts`: Permission modes (default, plan, auto-accept, etc.)
- `PermissionRule.ts`: Rule types (allow, deny, ask) with source tracking
- `permissionRuleParser.ts`: Parses rule values from settings strings
- `permissionsLoader.ts`: Loads rules from all settings sources
- `PermissionUpdate.ts`: Applies and persists rule changes
- `pathValidation.ts`: Path-based permission validation
- `shellRuleMatching.ts`: Shell command matching against permission rules
- `filesystem.ts`: Filesystem permission boundaries
- `dangerousPatterns.ts`: Dangerous command pattern detection
- `yoloClassifier.ts`: Auto-mode classifier (LLM-based safety check)
- `bashClassifier.ts`: Bash command safety classifier (ant-only)
- `classifierDecision.ts`: Classifier decision framework
- `autoModeState.ts`: Auto-mode state tracking
- `denialTracking.ts`: Tracks permission denials for analytics
- `shadowedRuleDetection.ts`: Detects rules shadowed by higher-precedence rules

### Swarm (`utils/swarm/`)

**Path:** `src/utils/swarm/`

Multi-agent coordination system ("teammate" agents) using tmux or iTerm2:

- `constants.ts`: Session names, env vars, socket names
- `spawnUtils.ts`: Teammate spawning utilities
- `spawnInProcess.ts`: In-process teammate spawning (no terminal multiplexer)
- `inProcessRunner.ts`: In-process agent runner
- `teammateInit.ts`: Teammate initialization
- `teammateModel.ts`: Teammate model selection
- `teammateLayoutManager.ts`: Pane layout management
- `teamHelpers.ts`: Team coordination helpers
- `permissionSync.ts`: Permission synchronization across teammates
- `leaderPermissionBridge.ts`: Leader-to-teammate permission bridging
- `reconnection.ts`: Teammate reconnection after disconnection
- `teammatePromptAddendum.ts`: Additional prompt context for teammates

**Backends:**
- `TmuxBackend.ts`: tmux-based multiplexing
- `ITermBackend.ts`: iTerm2 native split panes
- `InProcessBackend.ts`: In-process (no UI)
- `PaneBackendExecutor.ts`: Abstract pane execution
- `registry.ts`: Backend registry and selection
- `detection.ts`: Terminal multiplexer detection

### Deep Link (`utils/deepLink/`)

**Path:** `src/utils/deepLink/`

Handles `claude-cli://open` URIs for launching Claude Code from external sources:

- `parseDeepLink.ts`: URI parser with security validation (control char rejection, repo slug validation)
- `protocolHandler.ts`: Protocol handler registration
- `registerProtocol.ts`: OS-level protocol registration
- `terminalLauncher.ts`: Launches Claude Code in a terminal from deep link
- `terminalPreference.ts`: Terminal preference detection
- `banner.ts`: Display banner for deep link launches

### Claude In Chrome (`utils/claudeInChrome/`)

**Path:** `src/utils/claudeInChrome/`

Integration with Chrome browser extension:

- `common.ts`: Shared constants and types
- `chromeNativeHost.ts`: Chrome Native Messaging host
- `mcpServer.ts`: MCP server for Chrome extension communication
- `prompt.ts`: Chrome tool search instructions
- `setup.ts`: Extension setup and detection
- `setupPortable.ts`: Portable setup (no native dependencies)
- `toolRendering.tsx`: Tool result rendering for Chrome

### Session Storage (`utils/sessionStorage.ts`)

The session storage module is one of the largest files (~180KB). It manages:
- Conversation transcript persistence to disk
- Session metadata (model, project, timestamps)
- Message serialization/deserialization
- Transcript search and filtering
- Session listing and cleanup
- Worktree session tracking
- Content replacement (for privacy)
- Context collapse snapshots

### Session Start (`utils/sessionStart.ts`)

Orchestrates session initialization hooks:
- Loads plugin hooks (unless restricted to managed-only)
- Executes `SessionStart` hooks from settings
- Collects additional context and file watch paths
- Handles `initialUserMessage` from hooks (for print mode)
- Skipped entirely in `--bare` mode

### Other Notable Utilities

| Path | Purpose |
|------|---------|
| `utils/auth.ts` | Authentication state management (65KB): OAuth tokens, API keys, credential refresh |
| `utils/config.ts` | Global configuration management (63KB): user config, project config |
| `utils/messages.ts` | Message utilities (193KB): creation, normalization, API format conversion |
| `utils/hooks.ts` | Hook system (159KB): pre/post tool hooks, session hooks, notification hooks |
| `utils/sessionStorage.ts` | Session persistence (180KB): transcript storage, search, metadata |
| `utils/teleport.tsx` | Session teleportation (175KB): move sessions between devices |
| `utils/attachments.ts` | Attachment system (127KB): file, image, delta attachments |
| `utils/model/` | Model utilities: providers, model strings, bedrock, cost calculation |
| `utils/shell/` | Shell execution: provider abstraction, output limits, read-only validation |
| `utils/plugins/` | Plugin system (46 files): marketplace, installation, versioning, caching |
| `utils/gracefulShutdown.ts` | Shutdown orchestration with cleanup registry |
| `utils/worktree.ts` | Git worktree management (50KB) |
| `utils/processUserInput/` | User input processing pipeline |
| `utils/background/` | Background/remote session management |
| `utils/memory/` | Memory file types and versioning |
| `utils/secureStorage/` | Secure credential storage |
| `utils/sandbox/` | Sandboxed execution environment |
| `utils/task/` | Task management |
| `utils/computerUse/` | Computer use (screen interaction) support |
| `utils/git/` | Git operations beyond basic git.ts |

---

## 14. Service Dependency Diagram

```
                              +-------------------+
                              |   Bootstrap/Init  |
                              +-------------------+
                                       |
              +------------------------+------------------------+
              |                        |                        |
              v                        v                        v
     +----------------+      +-----------------+      +------------------+
     | Remote Managed |      |  Policy Limits  |      |    Analytics     |
     |   Settings     |      |                 |      |   (GrowthBook)   |
     +----------------+      +-----------------+      +------------------+
              |                        |                        |
              +----------+-------------+                        |
                         |                                      |
                         v                                      v
                  +-------------+                     +------------------+
                  |  Settings   |<--------------------|  Feature Flags   |
                  |  (merged)   |                     |                  |
                  +-------------+                     +------------------+
                         |
           +-------------+-------------+
           |             |             |
           v             v             v
    +-----------+  +-----------+  +------------+
    |   OAuth   |  |    API    |  | Permissions|
    |  Service  |  |  Service  |  |   System   |
    +-----------+  +-----------+  +------------+
           |             |             |
           +------+------+             |
                  |                    |
                  v                    v
           +------------+       +------------+
           |   Claude   |       |   Tools    |
           |  Messages  |       | Execution  |
           |    API     |       |            |
           +------------+       +------------+
                  |                    |
           +------+------+------+-----+
           |             |             |
           v             v             v
    +-----------+  +-----------+  +------------+
    |  Compact  |  |  Session  |  |    LSP     |
    |  Service  |  |  Memory   |  |  Service   |
    +-----------+  +-----------+  +------------+
           |             |             |
           v             v             v
    +-----------+  +-----------+  +------------+
    |  Session  |  |  Memory   |  | Diagnostic |
    |  Storage  |  |  Files    |  |  Registry  |
    +-----------+  +-----------+  +------------+


    MCP Service  <-->  Tools Execution  <-->  Swarm (Teammates)
         |                    |                      |
         v                    v                      v
    MCP Servers         Tool Results          Tmux/iTerm2/
    (external)          (formatted)           InProcess
```

### Key Dependency Rules

1. **Analytics has zero imports** -- events are queued until sink is attached
2. **Settings never imports from services** -- avoids circular deps during loading
3. **Policy limits and remote settings use their own auth** -- bypass `getSettings()` to avoid circular deps
4. **Compact depends on API** -- forked agents need API client
5. **Session memory depends on compact** -- compaction threshold awareness
6. **LSP depends on plugins** -- servers are exclusively plugin-provided
7. **OAuth is consumed by API client and all authenticated services**
8. **Feature flags (GrowthBook) are consumed everywhere** -- via cached stale reads to avoid blocking

### Error Handling Patterns Across Services

| Pattern | Services | Behavior |
|---------|----------|----------|
| Fail-open | Policy Limits, Remote Settings, Bootstrap | Continue without data on failure |
| Fail-closed | OAuth (expired), Permissions | Block operation until resolved |
| Retry with backoff | API, Session Ingress, Files API, Policy Limits | Exponential backoff with jitter |
| Circuit breaker | Auto-compact | Stop after N consecutive failures |
| Cache-first | Remote Settings, Policy Limits, Bootstrap | Serve stale cache, refresh async |
| Graceful degradation | LSP, MCP, Analytics | Feature disabled if service fails |

---

*Generated from source analysis of `src/services/` and `src/utils/` as of the current codebase state.*
