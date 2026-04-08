# Product Features & Platform Integration

## Document Scope

This document covers the user-facing features and OS/platform integrations in the Claude Code codebase. Each section describes the feature's purpose, user-facing behavior, implementation approach, configuration surface, and key dependencies.

---

## Table of Contents

1. [Buddy Companion System](#1-buddy-companion-system)
2. [Voice Input Integration](#2-voice-input-integration)
3. [Vim-Mode Editing](#3-vim-mode-editing)
4. [Upstream Proxy (CCR)](#4-upstream-proxy-ccr)
5. [Deep Linking / URL Scheme Handling](#5-deep-linking--url-scheme-handling)
6. [Chrome Browser Integration](#6-chrome-browser-integration)
7. [MoreRight Module](#7-moreright-module)
8. [Native TypeScript Utilities](#8-native-typescript-utilities)
9. [CLI Subcommands & Features](#9-cli-subcommands--features)
10. [Output Styles](#10-output-styles)
11. [Feature Flag System](#11-feature-flag-system)
12. [Server Mode (Direct Connect)](#12-server-mode-direct-connect)
13. [Telemetry & Analytics](#13-telemetry--analytics)
14. [Feature Capability Matrix](#14-feature-capability-matrix)
15. [Integration Architecture](#15-integration-architecture)

---

## 1. Buddy Companion System

**Source:** `src/buddy/`  
**Feature flag:** `BUDDY`

### Purpose

A virtual companion creature that lives beside the user's input box in the terminal REPL. Each user gets a deterministically generated companion based on their account UUID, providing a personality-rich idle presence with ASCII art sprites.

### User-Facing Behavior

- **Teaser phase:** During April 1-7, 2026, a rainbow `/buddy` notification appears on startup if no companion is hatched. Internal (ant) users see it year-round.
- **Hatching:** Running `/buddy` creates ("hatches") a companion. The model generates a name and personality (`CompanionSoul`) stored in `~/.claude.json`.
- **Idle presence:** The `CompanionSprite` component renders the companion as a 5-line, 12-wide ASCII art sprite with idle fidget animations (3 frames per species), displayed beside the prompt input.
- **Reactions:** The companion observes conversation turns and reacts in a speech bubble. When the user addresses the companion by name, the companion's bubble answers directly.
- **Muting:** `companionMuted` config flag suppresses the companion introduction from the system prompt.

### Implementation

**Deterministic generation (bones):** A seeded PRNG (`mulberry32`) keyed on `hash(userId + SALT)` generates all visual traits:

| Attribute | Source | Cardinality |
|-----------|--------|-------------|
| Species | 18 types (duck, goose, blob, cat, dragon, octopus, owl, penguin, turtle, snail, ghost, axolotl, capybara, cactus, robot, rabbit, mushroom, chonk) | 18 |
| Eye style | 6 variants (`·`, `✦`, `×`, `◉`, `@`, `°`) | 6 |
| Hat | 8 types (none, crown, tophat, propeller, halo, wizard, beanie, tinyduck) | 8 |
| Rarity | 5 tiers with weighted probabilities | 5 |
| Stats | 5 attributes (DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK) | continuous |
| Shiny | 1% chance | boolean |

**Rarity weights:** common (60), uncommon (25), rare (10), epic (4), legendary (1). Common companions get no hat; all others receive a random hat.

**Stat generation:** One peak stat (floor + 50 + rand(30)), one dump stat (floor - 10 + rand(15)), remainder at floor + rand(40). The floor increases with rarity (common=5, uncommon=15, rare=25, epic=35, legendary=50).

**Soul (model-generated):** After hatching, the AI generates a name and personality. Only the soul persists in config; bones are regenerated from the user ID on every read, preventing users from editing config to fake a rarity.

**Sprite rendering:** `renderSprite()` substitutes `{E}` placeholders with the eye character and conditionally applies hat lines to the top row. Frame animation cycles through 3 frames per species.

**System prompt integration:** `getCompanionIntroAttachment()` injects a companion introduction into the conversation context, instructing the model to stay out of the way when the companion's bubble is handling direct address.

### Configuration

- `companion` in global config: stores `CompanionSoul` (name, personality, hatchedAt)
- `companionMuted`: boolean to suppress companion prompt injection
- Species names are encoded as hex char codes to avoid build-time string detection for model codename canaries

### Dependencies

- `bun:bundle` feature gate (`BUDDY`)
- `utils/config.js` for global config read/write
- `utils/thinking.js` for rainbow color generation
- React/Ink for sprite rendering

---

## 2. Voice Input Integration

**Source:** `src/voice/`  
**Feature flag:** `VOICE_MODE`

### Purpose

Speech-to-text voice input for the Claude Code REPL, allowing users to dictate prompts instead of typing.

### User-Facing Behavior

- Activated via the `/voice` command or a push-to-talk keybinding (space in normal mode)
- Requires Anthropic OAuth authentication (not available with API keys, Bedrock, Vertex, or Foundry)
- Can be killed remotely via GrowthBook flag `tengu_amber_quartz_disabled`

### Implementation

Three-tier availability check:

1. **`isVoiceGrowthBookEnabled()`** -- Build-time `VOICE_MODE` feature flag AND runtime GrowthBook kill-switch. Used for UI visibility decisions (command registration, config display).
2. **`hasVoiceAuth()`** -- Checks `isAnthropicAuthEnabled()` AND that an OAuth access token exists in the keychain. The keychain read is memoized; first call on macOS spawns the `security` binary (~20-50ms).
3. **`isVoiceModeEnabled()`** -- Full runtime check combining both above. Used by command-time paths (`/voice`, ConfigTool, VoiceModeNotice).

Voice streaming and STT implementation lives in `src/services/voiceStreamSTT.ts`, only reachable in internal builds gated by the feature flag. React integration hooks (`useVoiceIntegration`, `VoiceKeybindingHandler`) are conditionally required at the top of `REPL.tsx`.

### Configuration

- GrowthBook feature flag: `tengu_amber_quartz_disabled` (kill switch, default false)
- Build-time flag: `VOICE_MODE`
- Requires OAuth tokens from `claude.ai`

---

## 3. Vim-Mode Editing

**Source:** `src/vim/`

### Purpose

A complete Vim modal editing implementation for the prompt input field, supporting normal mode, insert mode, motions, operators, text objects, dot-repeat, and persistent state.

### Architecture

The Vim subsystem is a pure state machine with four modules:

```
types.ts          -- State machine types and key group constants
    |
transitions.ts   -- State transition table (dispatches input -> next state + side effects)
    |
motions.ts       -- Pure functions: resolve motion to cursor position
operators.ts     -- Pure functions: execute operators (delete, change, yank, etc.)
textObjects.ts   -- Pure functions: find text object boundaries
```

### State Machine

```
                          VimState
  ┌──────────────────────────┬──────────────────────────────────┐
  │  INSERT                  │  NORMAL                          │
  │  (tracks insertedText)   │  (CommandState machine)          │
  │                          │                                  │
  │                          │  idle ──┬─[d/c/y]──> operator    │
  │                          │         ├─[1-9]────> count       │
  │                          │         ├─[fFtT]───> find        │
  │                          │         ├─[g]──────> g           │
  │                          │         ├─[r]──────> replace     │
  │                          │         └─[><]─────> indent      │
  │                          │                                  │
  │                          │  operator ─┬─[motion]──> execute │
  │                          │            ├─[0-9]────> opCount  │
  │                          │            ├─[ia]─────> textObj  │
  │                          │            └─[fFtT]───> opFind   │
  └──────────────────────────┴──────────────────────────────────┘
```

### Supported Features

**Motions:** `h l j k w b e W B E 0 ^ $ G gg gj gk`

**Operators:** `d` (delete), `c` (change), `y` (yank) -- all composable with motions, counts, and text objects. Line operations: `dd`, `cc`, `yy`.

**Text objects:** `iw aw iW aW i" a" i' a' i( a) ib i[ a] i{ a} iB i< a>`

**Commands:** `x` (delete char), `r` (replace char), `~` (toggle case), `J` (join lines), `p/P` (paste), `o/O` (open line), `D C Y` (shorthand operators), `.` (dot-repeat), `;,` (repeat find), `u` (undo), `>> <<` (indent/dedent), `i I a A` (enter insert mode)

**Persistent state:** Last change (for dot-repeat), last find (for `;` and `,`), register content, linewise flag.

### Implementation Details

- All motion/operator functions are pure -- they take a `Cursor` object and return new positions or text
- The `Cursor` class (from `utils/Cursor.js`) handles grapheme-aware text manipulation, word boundaries, and logical vs. wrapped line navigation
- Count multipliers are supported with a max of 10,000 (`MAX_VIM_COUNT`)
- `cw/cW` has special handling per Vim convention (changes to end of word, not start of next)
- Image reference chips (`[Image #N]`) are handled by `snapOutOfImageRef` to prevent partial placeholder deletion

---

## 4. Upstream Proxy (CCR)

**Source:** `src/upstreamproxy/`

### Purpose

A CONNECT-over-WebSocket relay for Claude Code Remote (CCR) session containers, enabling the server-side upstreamproxy to intercept and inject organization-configured credentials (e.g., Datadog API keys) into outbound HTTPS requests from agent subprocesses.

### Activation Conditions

Requires all of:
1. `CLAUDE_CODE_REMOTE` env var is truthy
2. `CCR_UPSTREAM_PROXY_ENABLED` env var is truthy (set server-side)
3. `CLAUDE_CODE_REMOTE_SESSION_ID` is set
4. Session token file exists at `/run/ccr/session_token`

### Initialization Sequence

```
1. Read session token from /run/ccr/session_token
2. prctl(PR_SET_DUMPABLE, 0) -- block same-UID ptrace of heap (Linux FFI)
3. Download CA cert from ${ANTHROPIC_BASE_URL}/v1/code/upstreamproxy/ca-cert
4. Concatenate with system CA bundle -> ~/.ccr/ca-bundle.crt
5. Start local TCP relay on 127.0.0.1:0 (ephemeral port)
6. Unlink token file (token stays heap-only)
7. Export HTTPS_PROXY + SSL_CERT_FILE env vars for subprocesses
```

### Relay Architecture

The relay (`relay.ts`) operates in two phases per connection:

**Phase 1 (CONNECT parsing):** Accumulates incoming TCP data until `\r\n\r\n` is seen. Validates the `CONNECT host:port HTTP/1.x` request line.

**Phase 2 (WebSocket tunneling):** Opens a WebSocket to `${baseUrl}/v1/code/upstreamproxy/ws` with the session token as Bearer auth. Client bytes are wrapped in `UpstreamProxyChunk` protobuf messages (hand-encoded, no protobuf dep) and sent over the WebSocket. Server responses are decoded and piped back to the TCP client.

**Protocol details:**
- Protobuf: `message UpstreamProxyChunk { bytes data = 1; }` -- hand-encoded (tag 0x0a + varint length + data)
- Max chunk size: 512KB (envoy per-request buffer cap)
- Keepalive: empty chunk every 30s (sidecar idle timeout is 50s)
- Runtime dispatch: Bun.listen for Bun runtime, net.createServer for Node (CCR containers run Node)

### Subprocess Environment

`getUpstreamProxyEnv()` returns environment variables merged into every agent subprocess:

| Variable | Value |
|----------|-------|
| `HTTPS_PROXY` / `https_proxy` | `http://127.0.0.1:<port>` |
| `NO_PROXY` / `no_proxy` | localhost, RFC1918, IMDS, anthropic.com, github.com, npm, pypi, crates.io, proxy.golang.org |
| `SSL_CERT_FILE` | `~/.ccr/ca-bundle.crt` |
| `NODE_EXTRA_CA_CERTS` | same |
| `REQUESTS_CA_BUNDLE` | same |
| `CURL_CA_BUNDLE` | same |

### Security

- Token file is unlinked after relay starts (heap-only retention)
- `prctl(PR_SET_DUMPABLE, 0)` prevents same-UID ptrace (blocks prompt-injected `gdb -p $PPID`)
- Every step fails open: errors log warnings and disable the proxy rather than breaking the session

---

## 5. Deep Linking / URL Scheme Handling

**Source:** `src/utils/deepLink/`  
**GrowthBook flag:** `tengu_lodestone_enabled`

### Purpose

Registers the `claude-cli://` custom URI scheme with the OS, allowing browser links (or any app) to open Claude Code in a terminal with pre-filled prompts and a target working directory.

### URI Format

```
claude-cli://open[?q=<prompt>&cwd=<path>&repo=<owner/name>]
```

| Parameter | Description | Validation |
|-----------|-------------|------------|
| `q` | Pre-fill prompt (not auto-submitted) | Max 5000 chars, no ASCII control chars, Unicode sanitized |
| `cwd` | Working directory | Must be absolute path, max 4096 chars |
| `repo` | GitHub owner/repo slug | Must match `[\w.-]+/[\w.-]+` pattern |

### Protocol Registration

Platform-specific registration in `registerProtocol.ts`:

| Platform | Mechanism | Artifact |
|----------|-----------|----------|
| macOS | .app trampoline with `CFBundleURLTypes` in Info.plist | `~/Applications/Claude Code URL Handler.app` (symlink to claude binary) |
| Linux | .desktop file + `xdg-mime` registration | `$XDG_DATA_HOME/applications/claude-code-url-handler.desktop` |
| Windows | Registry keys under `HKCU\Software\Classes\claude-cli` | Registry entries |

**macOS details:** The .app's `CFBundleExecutable` is a symlink to the installed claude binary. This avoids shipping a separate executable that would need signing. When macOS opens a `claude-cli://` URL, it launches claude through the app bundle. A NAPI module (`url-handler-napi`) reads the URL from the Apple Event. Detection: `__CFBundleIdentifier === 'com.anthropic.claude-code-url-handler'`.

### URI Handling Flow

```
OS invokes: claude --handle-uri <url>
    |
    v
parseDeepLink(uri)  -- validates, sanitizes, extracts action
    |
    v
resolveCwd()  -- precedence: explicit cwd > repo lookup (MRU clone) > home
    |
    v
readLastFetchTime()  -- check git FETCH_HEAD age for staleness warning
    |
    v
launchInTerminal()  -- detect terminal, open new window with claude --prefill
```

### Terminal Detection & Launch

`terminalLauncher.ts` detects and launches across platforms:

**macOS:** iTerm2, Ghostty, Kitty, Alacritty, WezTerm, Terminal.app  
**Linux:** $TERMINAL, x-terminal-emulator, ghostty, kitty, alacritty, wezterm, gnome-terminal, konsole, xfce4-terminal, mate-terminal, tilix, xterm  
**Windows:** Windows Terminal, PowerShell 7+, PowerShell 5.1, cmd.exe

Two launch strategies:
- **Pure argv** (Ghostty, Alacritty, Kitty, WezTerm, all Linux, Windows Terminal): No shell, user input travels as distinct argv elements. No injection risk.
- **Shell-string** (iTerm, Terminal.app via AppleScript; PowerShell, cmd.exe): User input is shell-quoted. Correctness of quoting is security-critical.

### Auto-Registration

`ensureDeepLinkProtocolRegistered()` runs every session from background housekeeping. It checks if the artifact points at the current binary, re-registers if stale, and backs off 24h on deterministic failures (EACCES, ENOSPC).

---

## 6. Chrome Browser Integration

**Source:** `src/utils/claudeInChrome/`

### Purpose

Integrates Claude Code with a Chrome browser extension via a native messaging host and MCP server, enabling Claude Code to control the browser (navigate, read pages, fill forms, take screenshots, execute JavaScript).

### Architecture

```
┌─────────────────┐     Native Messaging     ┌──────────────────┐
│ Chrome Extension │ <======================> │ Native Host      │
│ (browser)        │     (stdin/stdout)       │ (claude binary)  │
└─────────────────┘                           └────────┬─────────┘
                                                       │ Unix Socket / Named Pipe
                                                       v
                                              ┌──────────────────┐
                                              │ MCP Server       │
                                              │ (claude process) │
                                              └──────────────────┘
                                                       │
                                                       v
                                              ┌──────────────────┐
                                              │ Claude Code REPL │
                                              │ (tool calls)     │
                                              └──────────────────┘
```

**Alternative path (Bridge):** For users without local Chrome or in remote environments, a WebSocket bridge (`wss://bridge.claudeusercontent.com`) connects the extension to Claude Code without the native host. Enabled via GrowthBook flag `tengu_copper_bridge`.

### Supported Browsers

Chrome, Brave, Arc, Chromium, Edge, Vivaldi, Opera -- all Chromium-based browsers with native messaging support. Configuration per-browser covers:
- Data paths (for extension detection)
- Native messaging host directories (for manifest installation)
- Windows registry keys

### Native Host

`chromeNativeHost.ts` implements Chrome's native messaging protocol:
- 4-byte little-endian length prefix + JSON payload over stdin/stdout
- Manages MCP client connections over Unix sockets (`/tmp/claude-mcp-browser-bridge-<username>/<pid>.sock`)
- Message types: `ping`, `get_status`, `tool_response`, `notification`
- Stale socket cleanup on startup (checks if PID is alive)
- Max message size: 1MB

### MCP Server

`mcpServer.ts` creates a full MCP server using `@ant/claude-for-chrome-mcp`:
- Runs as a subprocess (`claude --claude-in-chrome-mcp`)
- Exposes browser tools: `computer`, `find`, `form_input`, `get_page_text`, `gif_creator`, `javascript_tool`, `navigate`, `read_console_messages`, `read_network_requests`, `read_page`, `resize_window`, `shortcuts_execute/list`, `switch_browser`, `tabs_context/create_mcp`, `update_plan`, `upload_image`
- Permission modes: `ask`, `skip_all_permission_checks`, `follow_a_plan`
- Inference support for `browser_task` tool (internal only): runs a lightning-mode agent loop via `sideQuery`

### Configuration

- CLI flag: `--chrome` (explicit enable)
- Environment: `CLAUDE_CODE_ENABLE_CFC` (truthy/falsy)
- Config: `claudeInChromeDefaultEnabled` in global config
- Auto-enable: GrowthBook flag `tengu_chrome_auto_enable` + extension installed + interactive session
- Default: disabled in non-interactive sessions (SDK, CI)

---

## 7. MoreRight Module

**Source:** `src/moreright/`

### Purpose

An internal-only feature hook with an opaque interface. The external build contains a no-op stub.

### Implementation

The external stub exposes three callbacks:
- `onBeforeQuery(input, messages, turnNumber) -> boolean` -- called before each query, returns whether to proceed
- `onTurnComplete(messages, aborted) -> void` -- called after each turn completes
- `render() -> null` -- React render slot

The real implementation is internal only and replaced at build time. The stub returns `true` for onBeforeQuery (always proceed), no-ops for onTurnComplete, and `null` for render.

### Interface

```typescript
useMoreRight({
  enabled: boolean
  setMessages: (action) => void
  inputValue: string
  setInputValue: (s: string) => void
  setToolJSX: (args) => void
}): {
  onBeforeQuery: (input, all, n) => Promise<boolean>
  onTurnComplete: (all, aborted) => Promise<void>
  render: () => null
}
```

---

## 8. Native TypeScript Utilities

**Source:** `src/native-ts/`

### Purpose

Pure TypeScript ports of Rust NAPI modules, eliminating native binary dependencies while preserving API compatibility.

### Modules

#### 8.1 Color Diff (`native-ts/color-diff/`)

Port of `vendor/color-diff-src` (Rust + syntect + bat + similar crate).

- **Syntax highlighting:** Uses `highlight.js` (lazy-loaded to avoid 100-200ms startup penalty from registering 190+ grammars)
- **Word diffing:** Uses the `diff` npm package's `diffArrays`
- **Color mapping:** Scope colors measured from syntect's output for approximate parity. Known gap: hljs doesn't scope plain identifiers and operators, so they render in default fg instead of white/pink
- **API:** Matches `vendor/color-diff-src/index.d.ts` exactly

#### 8.2 File Index (`native-ts/file-index/`)

Port of `vendor/file-index-src` (Rust + nucleo fuzzy matcher).

- **Scoring:** nucleo-style scoring with SCORE_MATCH(16), BONUS_BOUNDARY(8), BONUS_CAMEL(6), BONUS_CONSECUTIVE(4), BONUS_FIRST_CHAR(8), PENALTY_GAP_START(3), PENALTY_GAP_EXTENSION(1)
- **Optimization:** O(1) bitmap rejection via per-path a-z character bitmaps (26-bit). Fused indexOf scan accumulates gap/consecutive terms inline with position finding
- **Async loading:** `loadFromFileListAsync()` yields to event loop every ~4ms so 270k+ file indexes don't block the main thread
- **Smart case:** Lowercase query = case-insensitive; any uppercase = case-sensitive
- **Test file penalty:** Paths containing "test" get a 1.05x score penalty
- **Top-level cache:** Precomputes top 100 top-level path segments for empty-query results

#### 8.3 Yoga Layout (`native-ts/yoga-layout/`)

Port of Meta's Yoga flexbox layout engine (C++, ~2500 lines in `CalculateLayout.cpp`).

Covers the subset of features Ink actually uses:
- flex-direction (row/column + reverse)
- flex-grow / flex-shrink / flex-basis
- align-items / align-self / justify-content
- margin / padding / border / gap
- width / height / min / max (point, percent, auto)
- position: relative / absolute
- display: flex / none / contents
- measure functions (for text nodes)
- flex-wrap: wrap / wrap-reverse
- align-content, baseline alignment

---

## 9. CLI Subcommands & Features

**Source:** `src/cli/`

### 9.1 Doctor Command

**Source:** `src/screens/Doctor.tsx`, `src/utils/doctorDiagnostic.ts`, `src/utils/doctorContextWarnings.ts`

Comprehensive health check that diagnoses:
- Current version and available updates (npm/GCS dist tags)
- Installation type detection (native, npm, local)
- Multiple installation detection
- Agent configuration (active agents by source, directory existence, failed files)
- Keybinding warnings
- MCP server parsing warnings
- Sandbox configuration
- Settings validation errors
- Plugin errors
- Context warnings (CLAUDE.md issues)
- Version lock status (PID-based locking)
- Environment variable validation (BASH_MAX_OUTPUT, etc.)

### 9.2 Install & Update

**Source:** `src/cli/update.ts`, `src/utils/nativeInstaller/`, `src/utils/autoUpdater.ts`

- Detects installation method (native installer, npm global, local project)
- Checks for latest version via npm dist-tags or GCS
- Handles native installer updates, npm global updates, and local installer updates
- Regenerates completion cache after update
- Warns about multiple installations

### 9.3 MCP Helper Commands

**Source:** `src/cli/handlers/mcp.tsx`

Full MCP server management:
- `mcp serve` -- starts Claude Code as an MCP server
- `mcp add` -- add an MCP server (stdio/SSE/HTTP/URL types, with scope: user/project/mcpz)
- `mcp remove` -- remove an MCP server (cleans up secure storage for OAuth tokens)
- `mcp list` -- list configured servers with health checks
- `mcp get` -- get details for a specific server
- `mcp reset-project-choices` -- clear project-level MCP enable/disable choices

### 9.4 Auth Commands

**Source:** `src/cli/handlers/auth.ts`

- `setup-token` -- long-lived (1-year) OAuth token setup flow
- `login` / `logout` -- OAuth login/logout with organization support
- Token refresh handling

### 9.5 Auto-Mode Commands

**Source:** `src/cli/handlers/autoMode.ts`

- `auto-mode defaults` -- dumps default external auto-mode classifier rules
- `auto-mode config` -- dumps effective merged config (user overrides + defaults)
- `auto-mode critique` -- uses sideQuery to critique user-written rules

### 9.6 Agent & Plugin Commands

**Source:** `src/cli/handlers/agents.ts`, `src/cli/handlers/plugins.ts`

- `agents` -- lists all configured agents grouped by source (built-in, user, project, plugin) with override resolution
- Plugin management commands

---

## 10. Output Styles

**Source:** `src/outputStyles/`, `src/constants/outputStyles.ts`

### Purpose

Configurable output formatting that controls how Claude responds -- from terse to educational.

### Built-in Styles

| Style | Description | Behavior |
|-------|-------------|----------|
| `default` | Standard mode | No additional prompt injection |
| `Explanatory` | Educational explanations | Adds "Insight" sections with 2-3 educational points before/after code |
| `Learning` | Hands-on practice | Pauses and asks user to write 2-10 line code pieces for design decisions, business logic, key algorithms |

### Custom Output Styles

Users create markdown files in:
- `~/.claude/output-styles/*.md` -- user-level styles
- `.claude/output-styles/*.md` -- project-level styles (override user-level)
- Plugin-provided styles

**Frontmatter fields:**
- `name` -- display name (defaults to filename)
- `description` -- style description
- `keep-coding-instructions` -- boolean, whether to retain default coding instructions alongside the custom prompt
- `force-for-plugin` -- (plugin styles only) auto-apply when plugin is enabled

### Loading

`getOutputStyleDirStyles()` (memoized) loads markdown files from both user and project directories using `loadMarkdownFilesForSubdir`. The active style's prompt replaces or augments the default system prompt.

---

## 11. Feature Flag System

### Mechanism

The codebase uses a two-tier feature flag system:

#### Build-Time Flags (`bun:bundle`)

```typescript
import { feature } from 'bun:bundle'
if (feature('VOICE_MODE')) { ... }
```

These are resolved at bundle time. When a flag is `false`, the entire conditional branch (including all transitively imported modules) is tree-shaken from the build. This is the primary mechanism for separating internal (`ant`) and external builds.

**Pattern:** Positive ternary (`feature('X') ? ... : false`) is preferred over negative pattern (`if (!feature('X')) return`) because the negative pattern doesn't eliminate inline string literals from external builds.

#### Runtime Flags (GrowthBook)

```typescript
import { getFeatureValue_CACHED_MAY_BE_STALE } from './growthbook.js'
const value = getFeatureValue_CACHED_MAY_BE_STALE('tengu_flag_name', defaultValue)
```

GrowthBook SDK with remote evaluation. Cached to disk so fresh installs work without waiting for GrowthBook init. User attributes include: device ID, session ID, platform, organization UUID, account UUID, subscription type, app version.

### Complete Feature Flag Registry

**Core UI/UX:**

| Flag | Domain |
|------|--------|
| `BUDDY` | Companion creature system |
| `VOICE_MODE` | Voice input |
| `MESSAGE_ACTIONS` | Message action buttons |
| `QUICK_SEARCH` | Quick file search |
| `TERMINAL_PANEL` | Embedded terminal panel |
| `HISTORY_PICKER` | History browser |
| `HISTORY_SNIP` | History snippet extraction |
| `AUTO_THEME` | Automatic theme detection |

**AI/Model Behavior:**

| Flag | Domain |
|------|--------|
| `COORDINATOR_MODE` | Multi-agent coordinator |
| `ULTRAPLAN` | Advanced planning UI |
| `ULTRATHINK` | Extended thinking mode |
| `PROACTIVE` | Proactive suggestions |
| `KAIROS` / `KAIROS_CHANNELS` / `KAIROS_BRIEF` / `KAIROS_DREAM` / `KAIROS_GITHUB_WEBHOOKS` / `KAIROS_PUSH_NOTIFICATION` | Persistent agent / channels system |
| `TRANSCRIPT_CLASSIFIER` | Auto-mode transcript classification |
| `BASH_CLASSIFIER` | Bash command classification |
| `EXTRACT_MEMORIES` | Memory extraction from conversations |
| `TEMPLATES` | Job/template classification |
| `TOKEN_BUDGET` | Token budget management |
| `REACTIVE_COMPACT` | Reactive compaction |
| `CACHED_MICROCOMPACT` | Cached micro-compaction |
| `COMPACTION_REMINDERS` | Compaction reminder prompts |
| `ANTI_DISTILLATION_CC` | Anti-distillation measures |
| `CONTEXT_COLLAPSE` | Context window collapse |
| `STREAMLINED_OUTPUT` | Streamlined output mode |
| `CONNECTOR_TEXT` | Connector text blocks |

**Infrastructure:**

| Flag | Domain |
|------|--------|
| `BRIDGE_MODE` | Bridge connectivity |
| `DIRECT_CONNECT` | Direct server connection |
| `SSH_REMOTE` | SSH remote sessions |
| `CCR_AUTO_CONNECT` / `CCR_MIRROR` / `CCR_REMOTE_SETUP` | Cloud Code Remote |
| `DAEMON` | Background daemon process |
| `BG_SESSIONS` | Background sessions |
| `AGENT_TRIGGERS` / `AGENT_TRIGGERS_REMOTE` | Scheduled agent triggers |
| `SELF_HOSTED_RUNNER` / `BYOC_ENVIRONMENT_RUNNER` | Custom runners |

**Tools & Capabilities:**

| Flag | Domain |
|------|--------|
| `WEB_BROWSER_TOOL` | Web browser tool |
| `CHICAGO_MCP` | MCP tool marketplace |
| `MCP_RICH_OUTPUT` / `MCP_SKILLS` | MCP enhancements |
| `MONITOR_TOOL` | Monitoring tool |
| `WORKFLOW_SCRIPTS` | Workflow scripts |
| `FORK_SUBAGENT` | Forked sub-agents |
| `VERIFICATION_AGENT` | Verification agent |
| `BUILTIN_EXPLORE_PLAN_AGENTS` | Built-in explore/plan agents |
| `TREE_SITTER_BASH` / `TREE_SITTER_BASH_SHADOW` | Tree-sitter bash parsing |

**Telemetry & Internal:**

| Flag | Domain |
|------|--------|
| `COWORKER_TYPE_TELEMETRY` | Coworker type tracking |
| `MEMORY_SHAPE_TELEMETRY` | Memory shape tracking |
| `ENHANCED_TELEMETRY_BETA` | Enhanced telemetry |
| `PERFETTO_TRACING` | Perfetto trace support |
| `SHOT_STATS` | Shot statistics |
| `SLOW_OPERATION_LOGGING` | Slow op logging |
| `ABLATION_BASELINE` | A/B test baseline |
| `TORCH` | Torch integration |

**Other:**

| Flag | Domain |
|------|--------|
| `LODESTONE` | Deep link system |
| `TEAMMEM` | Team memory sync |
| `NATIVE_CLIPBOARD_IMAGE` / `NATIVE_CLIENT_ATTESTATION` | Native capabilities |
| `COMMIT_ATTRIBUTION` | Git commit attribution |
| `HOOK_PROMPTS` | Hook prompt injection |
| `DOWNLOAD_USER_SETTINGS` / `UPLOAD_USER_SETTINGS` | Settings sync |
| `NEW_INIT` | New initialization flow |
| `BUILDING_CLAUDE_APPS` | Claude app building mode |
| `EXPERIMENTAL_SKILL_SEARCH` / `SKILL_IMPROVEMENT` / `RUN_SKILL_GENERATOR` | Skill system |
| `REVIEW_ARTIFACT` | Review artifact support |
| `AWAY_SUMMARY` | Away summary on return |
| `FILE_PERSISTENCE` | File persistence |
| `PROMPT_CACHE_BREAK_DETECTION` | Cache break detection |
| `UNATTENDED_RETRY` | Unattended retry logic |
| `POWERSHELL_AUTO_MODE` | PowerShell auto mode |
| `HARD_FAIL` | Hard failure mode |
| `BREAK_CACHE_COMMAND` | Cache break command |
| `DUMP_SYSTEM_PROMPT` | System prompt dumping |
| `OVERFLOW_TEST_TOOL` / `ALLOW_TEST_VERSIONS` | Testing aids |
| `AGENT_MEMORY_SNAPSHOT` | Agent memory snapshots |
| `UDS_INBOX` | Unix domain socket inbox |
| `IS_LIBC_GLIBC` / `IS_LIBC_MUSL` | libc detection |

---

## 12. Server Mode (Direct Connect)

**Source:** `src/server/`

### Purpose

Runs Claude Code as a persistent server that manages multiple sessions, accepting WebSocket connections for real-time interaction. Used for remote/hosted Claude Code deployments.

### Architecture

**Session lifecycle:**

```
Client ──POST /sessions──> Server (creates session subprocess)
   |                          |
   └──WebSocket /ws──────────>| (bidirectional streaming)
                              |
                     ┌────────┴────────┐
                     │ Session Process  │
                     │ (claude --input  │
                     │  -format         │
                     │  stream-json)    │
                     └─────────────────┘
```

### Configuration (`ServerConfig`)

| Field | Description |
|-------|-------------|
| `port` / `host` | Listen address |
| `authToken` | Bearer token for authentication |
| `unix` | Optional Unix domain socket path |
| `idleTimeoutMs` | Idle timeout for detached sessions (0 = never) |
| `maxSessions` | Maximum concurrent sessions |
| `workspace` | Default workspace directory |

### Session Management

Sessions have states: `starting` -> `running` -> `detached` -> `stopping` -> `stopped`

**Session index:** Persisted to `~/.claude/server-sessions.json` for resume across server restarts. Each entry tracks: session ID, transcript session ID (for `--resume`), cwd, permission mode, timestamps.

### WebSocket Protocol

The `DirectConnectSessionManager` handles:
- **Inbound:** SDK messages (user input), control responses (permission grants/denials), interrupt signals
- **Outbound:** Assistant messages, tool results, system messages, control requests (permission prompts)
- Message format: NDJSON matching the `--input-format stream-json` protocol
- Keep-alive messages are filtered out from client delivery

### Client Integration

`createDirectConnectSession()` posts to `${serverUrl}/sessions` with cwd and permission mode, receiving back a session ID, WebSocket URL, and optional work directory.

---

## 13. Telemetry & Analytics

**Source:** `src/services/analytics/`

### Architecture

```
logEvent(name, metadata)  -- fire-and-forget, no dependencies
         |
         v
   Event Queue (pre-init)  ──> attachAnalyticsSink() during startup
         |
         v
   ┌─────┴──────┐
   │ Sink Router │ -- initializeAnalyticsSink()
   └─────┬──────┘
         |
    ┌────┴────┐
    v         v
 Datadog    1P Logger
```

### Sink: Datadog

**Source:** `src/services/analytics/datadog.ts`

- Endpoint: `https://http-intake.logs.us5.datadoghq.com/api/v2/logs`
- Client token: public Datadog client token
- Gated by GrowthBook flag: `tengu_log_datadog_events`
- Batched: max 100 events, flushed every 15 seconds
- Network timeout: 5 seconds
- Allowlisted events only (explicit set of ~60 event names including `tengu_init`, `tengu_exit`, `tengu_api_success/error`, `tengu_tool_use_*`, `tengu_voice_*`, Chrome bridge events, etc.)
- `_PROTO_*` metadata keys are stripped before Datadog delivery (PII-tagged, 1P-only)

### Sink: First-Party (1P) Event Logger

**Source:** `src/services/analytics/firstPartyEventLogger.ts`

- Built on OpenTelemetry SDK (`@opentelemetry/sdk-logs`, `BatchLogRecordProcessor`)
- Custom `FirstPartyEventLoggingExporter` sends to Anthropic's 1P endpoint
- Event sampling: configurable per-event via GrowthBook dynamic config `tengu_event_sampling_config`
- Receives full payload including `_PROTO_*` keys (hoisted to proto fields for PII-tagged BQ columns)
- Kill switch via `sinkKillswitch.ts`

### Event Metadata Enrichment

`getEventMetadata()` in `metadata.ts` enriches every event with:
- Device/session identifiers (hashed)
- Platform, OS version, shell
- Model and provider info
- Auth state and subscription type
- Session statistics (turns, duration, tool use counts)
- Permission mode and organization info
- Coworker type (when `COWORKER_TYPE_TELEMETRY` flag enabled)
- Kairos state (when `KAIROS` flag enabled)

### GrowthBook Integration

**Source:** `src/services/analytics/growthbook.ts`

- SDK: `@growthbook/growthbook` with remote evaluation
- Client key loaded from constants
- User attributes: platform, organization UUID, account UUID, subscription type, email, app version, GitHub Actions metadata
- Disk cache: features cached to `~/.claude.json` for offline/startup use
- Experiment exposure tracking logged to 1P
- Auth-aware: recreates client when auth becomes available

### Privacy & Opt-Out

**Source:** `src/services/analytics/config.ts`

Analytics is disabled when:
- `NODE_ENV === 'test'`
- Third-party cloud providers (Bedrock/Vertex/Foundry)
- Privacy level is `no-telemetry` or `essential-traffic`

---

## 14. Feature Capability Matrix

```
┌───────────────────────┬────────┬─────────┬──────────┬─────────┬──────────┐
│ Feature               │ macOS  │ Linux   │ Windows  │ Auth    │ Build    │
│                       │        │         │          │ Req.    │ Gate     │
├───────────────────────┼────────┼─────────┼──────────┼─────────┼──────────┤
│ Buddy Companion       │   ✓    │    ✓    │    ✓     │ OAuth   │ BUDDY    │
│ Voice Input           │   ✓    │    ✓    │    ✓     │ OAuth   │ VOICE    │
│ Vim Mode              │   ✓    │    ✓    │    ✓     │   —     │   —      │
│ Upstream Proxy        │   —    │    ✓    │    —     │ CCR     │   —      │
│ Deep Linking          │   ✓    │    ✓    │    ✓     │   —     │ LODESTON │
│ Chrome Integration    │   ✓    │    ✓    │    ✓     │ OAuth*  │   —      │
│ MoreRight             │   —    │    —    │    —     │   —     │ Internal │
│ Server Mode           │   ✓    │    ✓    │    ✓     │ Token   │ DIRECT   │
│ Output Styles         │   ✓    │    ✓    │    ✓     │   —     │   —      │
│ Doctor                │   ✓    │    ✓    │    ✓     │   —     │   —      │
│ MCP Commands          │   ✓    │    ✓    │    ✓     │   —     │   —      │
│ Telemetry (Datadog)   │   ✓    │    ✓    │    ✓     │   —     │   —      │
│ Telemetry (1P)        │   ✓    │    ✓    │    ✓     │   —     │   —      │
│ Coordinator Mode      │   ✓    │    ✓    │    ✓     │   —     │ COORD    │
│ Kairos (Persistent)   │   ✓    │    ✓    │    ✓     │   —     │ KAIROS   │
│ Web Browser Tool      │   ✓    │    ✓    │    ✓     │   —     │ WEB_BR   │
└───────────────────────┴────────┴─────────┴──────────┴─────────┴──────────┘

* Chrome bridge requires OAuth for identity; native messaging does not.
```

---

## 15. Integration Architecture

### System Interaction Diagram

```
                           ┌─────────────────────────────────┐
                           │        User's Terminal           │
                           │  (iTerm, Ghostty, etc.)          │
                           └────────────────┬────────────────┘
                                            │
                    ┌───────────────────────┬┴───────────────────────┐
                    │                       │                        │
                    v                       v                        v
          ┌─────────────────┐   ┌────────────────────┐   ┌──────────────────┐
          │  Deep Link URI  │   │   Claude Code REPL  │   │  Server Mode     │
          │  Trampoline     │   │                     │   │  (WebSocket)     │
          │  (headless)     │   │  ┌──────────────┐   │   └────────┬─────────┘
          └────────┬────────┘   │  │ Vim Input    │   │            │
                   │            │  │ Engine       │   │            │
                   └───────────>│  └──────────────┘   │            │
                                │  ┌──────────────┐   │            │
              ┌────────────────>│  │ Voice STT    │   │<───────────┘
              │                 │  └──────────────┘   │
              │                 │  ┌──────────────┐   │
              │                 │  │ Buddy Sprite │   │
              │                 │  └──────────────┘   │
              │                 │  ┌──────────────┐   │
              │                 │  │ Output Style │   │
              │                 │  └──────────────┘   │
              │                 └─────────┬───────────┘
              │                           │
    ┌─────────┴──────────┐     ┌─────────┴──────────────┐
    │  Chrome Extension  │     │    Tool Execution       │
    │  (Native Msg /     │     │                         │
    │   Bridge WS)       │     │  ┌───────────────────┐  │
    └────────────────────┘     │  │ MCP Servers        │  │
                               │  ├───────────────────┤  │
                               │  │ Bash / File Tools  │  │
                               │  ├───────────────────┤  │
                               │  │ Agent Subprocesses │  │
                               │  └────────┬──────────┘  │
                               └───────────┼─────────────┘
                                           │
                               ┌───────────┴──────────────┐
                               │   Subprocess Environment  │
                               │                           │
                               │  HTTPS_PROXY (if CCR)     │
                               │  SSL_CERT_FILE (if CCR)   │
                               │  NO_PROXY                 │
                               └───────────────────────────┘
```

### Telemetry Data Flow

```
┌───────────────────┐     ┌──────────────┐     ┌───────────────────┐
│  logEvent()       │────>│  Event Queue │────>│  Sink Router      │
│  (no deps, sync)  │     │  (pre-init)  │     │                   │
└───────────────────┘     └──────────────┘     └─────┬─────────────┘
                                                     │
                    ┌────────────────────────────────┬┘
                    │                                │
                    v                                v
          ┌─────────────────┐           ┌────────────────────────┐
          │    Datadog       │           │   1P Event Logger      │
          │                  │           │   (OpenTelemetry)      │
          │  Allowlist-only  │           │                        │
          │  Batch (100/15s) │           │  All events            │
          │  _PROTO_ stripped│           │  _PROTO_ -> proto cols │
          │  GrowthBook gate │           │  Sampling config       │
          └─────────────────┘           └────────────────────────┘
                    │                                │
                    v                                v
          ┌─────────────────┐           ┌────────────────────────┐
          │  Datadog Logs   │           │  Anthropic 1P Backend  │
          │  (us5 region)   │           │  (BigQuery)            │
          └─────────────────┘           └────────────────────────┘
```

### Feature Flag Resolution Flow

```
┌──────────────────────┐
│  Source Code          │
│                       │
│  feature('FLAG')      │──── Build Time ────> Bundle (tree-shaken)
│                       │
│  getFeatureValue()    │──── Runtime ────┐
└──────────────────────┘                  │
                                          v
                               ┌──────────────────────┐
                               │  GrowthBook SDK      │
                               │                      │
                               │  Disk Cache           │
                               │  ~/.claude.json       │
                               │                      │
                               │  Remote Eval          │
                               │  (Anthropic API)      │
                               │                      │
                               │  User Attributes      │
                               │  (platform, org,      │
                               │   account, version)   │
                               └──────────────────────┘
```
