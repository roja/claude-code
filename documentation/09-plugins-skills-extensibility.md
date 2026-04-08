# Plugin & Skills System -- Extensibility Architecture

## Table of Contents

1. [Overview](#overview)
2. [Plugin System (`plugins/`, `utils/plugins/`)](#plugin-system)
3. [Skills Framework (`skills/`)](#skills-framework)
4. [Hooks System (`schemas/hooks.ts`, `utils/hooks/`)](#hooks-system)
5. [Settings & Configuration (`utils/settings/`)](#settings--configuration)
6. [Memory Directory System (`memdir/`)](#memory-directory-system)
7. [Data Migrations (`migrations/`)](#data-migrations)
8. [Extension Point Summary](#extension-point-summary)

---

## Overview

Claude Code's extensibility is built on four interlocking systems: **plugins** (third-party bundles of skills, hooks, and MCP servers), **skills** (slash-command prompts defined as markdown files), **hooks** (lifecycle event handlers that run shell commands, LLM prompts, HTTP calls, or agent verifiers), and a **settings hierarchy** that governs which extensions are active and what policies constrain them.

```
                              Settings Hierarchy
                    (user > project > local > flag > policy)
                                    |
                  +-----------------+-----------------+
                  |                 |                 |
              Plugins           Skills             Hooks
           (marketplace)     (markdown)        (settings.json)
                  |                 |                 |
         +-------+-------+        |        +--------+--------+
         |       |       |        |        |        |        |
       Skills  Hooks   MCP      Bundled  Command  Prompt  Agent/HTTP
      (plugin) (plugin) Servers  Skills
```

---

## Plugin System

### Architecture

The plugin system is implemented across two major areas:

- **`src/plugins/`** -- Built-in plugin registry and bundled plugin initialization
- **`src/utils/plugins/`** -- Marketplace management, plugin loading, caching, hook integration, and policy enforcement (45+ modules)

### Plugin Identity Model

Plugins are identified by a composite key: `{name}@{marketplace}`. Built-in plugins use the sentinel marketplace `builtin` (e.g., `my-plugin@builtin`). Marketplace plugins reference their marketplace name (e.g., `formatter@anthropic-tools`). Session-only plugins from `--plugin-dir` use `inline`.

```
PluginId = "{pluginName}@{marketplaceName}"
                 |                |
                 |                +-- "builtin" | "inline" | marketplace name
                 +-- kebab-case, no spaces
```

### Plugin Directory Structure

A plugin on disk follows this layout:

```
my-plugin/
  plugin.json          # Manifest with metadata (optional)
  commands/            # Slash commands (markdown files)
    build.md
    deploy.md
  agents/              # Agent definitions (markdown files)
    test-runner.md
  skills/              # Skill directories (SKILL.md pattern)
    my-skill/
      SKILL.md
  hooks/               # Hook configurations
    hooks.json         # Hook definitions (HooksSettings format)
  output-styles/       # Custom output style definitions
```

### Plugin Manifest Schema (`plugin.json`)

The manifest is validated by `PluginManifestSchema` in `src/utils/plugins/schemas.ts`:

```typescript
{
  name: string          // Unique identifier, kebab-case, no spaces
  version?: string      // Semver string
  description?: string  // User-facing explanation
  author?: {
    name: string
    email?: string
    url?: string
  }
  homepage?: string     // URL
  repository?: string   // Source code URL
  license?: string      // SPDX identifier
  keywords?: string[]   // Discovery tags
  dependencies?: Array<{
    plugin: string      // "name" or "name@marketplace"
    marketplace?: string
  }>

  // Component declarations (extend default directory conventions):
  commands?: string | string[] | Record<string, CommandMetadata>
  agents?: string | string[]
  skills?: string | string[]
  hooks?: string | HooksSettings | Array<string | HooksSettings>
  outputStyles?: string | string[]

  // MCP servers provided by this plugin:
  mcpServers?: Record<string, McpServerConfig>

  // LSP servers provided by this plugin:
  lspServers?: Record<string, LspServerConfig>

  // Plugin-scoped settings overlays:
  settings?: Record<string, unknown>
}
```

### Plugin Discovery and Loading

Plugin loading is orchestrated by `loadAllPluginsCacheOnly()` in `src/utils/plugins/pluginLoader.ts`, which is memoized per process.

```
Discovery Flow:
                                                        
  settings.json                                         
  (enabledPlugins)                                      
       |                                                
       v                                                
  parsePluginIdentifier()                               
       |                                                
       +----> Is it "name@builtin"?                     
       |         Yes -> getBuiltinPlugins()             
       |                                                
       +----> Is it "name@marketplace"?                 
       |         Yes -> Look up in known_marketplaces   
       |                  |                             
       |                  v                             
       |         getMarketplaceCacheOnly()              
       |                  |                             
       |                  v                             
       |         getPluginByIdCacheOnly()               
       |                  |                             
       |                  v                             
       |         resolvePluginPath()                    
       |           (versioned cache > legacy cache)     
       |                  |                             
       |                  v                             
       |         Load plugin.json manifest              
       |         Load hooks/hooks.json                  
       |         Resolve MCP/LSP servers                
       |                                                
       +----> Session plugins (--plugin-dir)?           
                 Yes -> Load directly from path         
                                                        
  All paths converge to:                                
       |                                                
       v                                                
  PluginLoadResult {                                    
    enabled: LoadedPlugin[]                             
    disabled: LoadedPlugin[]                            
    errors: PluginError[]                               
  }                                                     
```

**Discovery sources, in precedence order:**

1. **Marketplace-based plugins** -- `enabledPlugins` in settings maps `pluginId@marketplace` to `true/false/string[]`
2. **Session-only plugins** -- From `--plugin-dir` CLI flag or SDK `plugins` option
3. **Built-in plugins** -- Registered via `registerBuiltinPlugin()` at startup

### Plugin Caching

Plugins are cached under `~/.claude/plugins/cache/{marketplace}/{plugin}/{version}/`. The system supports:

- **Versioned cache paths** -- SHA-pinned or semver-tagged installations
- **Legacy cache paths** -- Backward compatibility with pre-versioning installations
- **Seed directories** -- Pre-populated caches (BYOC/enterprise deployment) probed via `CLAUDE_CODE_PLUGIN_SEED_DIRS`
- **ZIP cache** -- Optional compressed storage (`isPluginZipCacheEnabled()`)

### Plugin Installation Sources

| Source | Mechanism | Example |
|--------|-----------|---------|
| GitHub | `git clone --depth 1` | `source: { source: "github", repo: "org/repo" }` |
| Git URL | `git clone` with optional branch/SHA | `source: { source: "git", url: "https://..." }` |
| npm | `npm install` to global cache | `source: { source: "npm", package: "pkg-name" }` |
| URL | HTTP fetch of JSON manifest | `source: { source: "url", url: "https://..." }` |
| Settings inline | Inline in `extraKnownMarketplaces` | `source: { source: "settings", name: "..." }` |
| Local path | Direct filesystem reference | Marketplace entry with `source: "./path"` |
| MCPB/DXT | MCP bundle download + extraction | `source: "./tool.mcpb"` or URL |

### Built-in Plugin System

**`src/plugins/builtinPlugins.ts`** manages plugins that ship with the CLI binary and appear in the `/plugin` UI for user toggle:

```typescript
type BuiltinPluginDefinition = {
  name: string
  description: string
  version?: string
  skills?: BundledSkillDefinition[]
  hooks?: HooksSettings
  mcpServers?: Record<string, McpServerConfig>
  isAvailable?: () => boolean    // Hide if system lacks capability
  defaultEnabled?: boolean       // Defaults to true
}
```

**Key distinctions from bundled skills:**
- Built-in plugins appear in `/plugin` UI under "Built-in" section
- Users can enable/disable them (persisted to `enabledPlugins` in user settings)
- They can provide multiple component types (skills + hooks + MCP servers)
- Their plugin ID uses `{name}@builtin` format

**`src/plugins/bundled/index.ts`** is the initialization entry point, called at startup via `initBuiltinPlugins()`. The scaffolding is in place but no built-in plugins are currently registered -- this is the migration path for bundled skills that should be user-toggleable.

### LoadedPlugin Type

The runtime representation of a resolved plugin:

```typescript
type LoadedPlugin = {
  name: string
  manifest: PluginManifest
  path: string                     // Filesystem path (or "builtin" sentinel)
  source: string                   // "name@marketplace" identifier
  repository: string               // Usually same as source
  enabled?: boolean
  isBuiltin?: boolean
  sha?: string                     // Git commit SHA for version pinning
  commandsPath?: string            // Path to commands/ directory
  commandsPaths?: string[]         // Additional command paths from manifest
  commandsMetadata?: Record<string, CommandMetadata>
  agentsPath?: string
  agentsPaths?: string[]
  skillsPath?: string
  skillsPaths?: string[]
  outputStylesPath?: string
  outputStylesPaths?: string[]
  hooksConfig?: HooksSettings      // Parsed hook definitions
  mcpServers?: Record<string, McpServerConfig>
  lspServers?: Record<string, LspServerConfig>
  settings?: Record<string, unknown>
}
```

### Plugin Error Taxonomy

`PluginError` in `src/types/plugin.ts` is a 25-variant discriminated union covering every failure mode:

| Error Type | When |
|-----------|------|
| `path-not-found` | Component directory missing |
| `git-auth-failed` | SSH/HTTPS auth failure during clone |
| `git-timeout` | Clone/pull exceeded timeout |
| `network-error` | HTTP fetch failure |
| `manifest-parse-error` | JSON parse failure in plugin.json |
| `manifest-validation-error` | Schema validation failure |
| `plugin-not-found` | Plugin ID not in marketplace |
| `marketplace-not-found` | Unknown marketplace name |
| `marketplace-load-failed` | Marketplace fetch/parse failure |
| `marketplace-blocked-by-policy` | Enterprise blocklist/strictlist hit |
| `mcp-config-invalid` | Invalid MCP server configuration |
| `mcp-server-suppressed-duplicate` | Duplicate MCP server name |
| `mcpb-download-failed` | MCPB bundle download failure |
| `mcpb-extract-failed` | MCPB extraction failure |
| `mcpb-invalid-manifest` | Invalid DXT manifest in MCPB |
| `hook-load-failed` | Hook definition parse/load failure |
| `component-load-failed` | Generic component failure |
| `dependency-unsatisfied` | Required plugin dependency missing/disabled |
| `plugin-cache-miss` | Versioned cache empty, needs refresh |
| `lsp-*` (5 variants) | LSP server lifecycle failures |
| `generic-error` | Catch-all |

### Plugin Policy and Security

Enterprise administrators control plugins through managed settings:

- **`strictKnownMarketplaces`** -- Allowlist of marketplace sources; blocks all others before download
- **`blockedMarketplaces`** -- Denylist of marketplace sources
- **`strictPluginOnlyCustomization`** -- When set, blocks non-plugin customization for specified surfaces (`skills`, `agents`, `hooks`, `mcp`)
- **Official marketplace name reservation** -- Names like `claude-code-marketplace`, `anthropic-plugins` are reserved for `anthropics` GitHub org
- **Homograph attack protection** -- Non-ASCII characters in marketplace names are blocked

### Marketplace Management

`src/utils/plugins/marketplaceManager.ts` manages the marketplace lifecycle:

```
~/.claude/
  plugins/
    known_marketplaces.json       # All configured marketplace sources
    marketplaces/                 # Cached marketplace data
      my-marketplace.json         # URL-sourced marketplace manifest
      github-marketplace/         # Git-cloned marketplace repo
        .claude-plugin/
          marketplace.json
    cache/                        # Installed plugin caches
      {marketplace}/
        {plugin}/
          {version}/              # Versioned plugin directory
```

Auto-update behavior: official Anthropic marketplaces auto-update by default (except those in `NO_AUTO_UPDATE_OFFICIAL_MARKETPLACES`). Third-party marketplaces do not auto-update unless explicitly configured.

### Plugin Hook Hot Reload

When remote managed settings change (e.g., enterprise policy push), `setupPluginHookHotReload()` in `src/utils/plugins/loadPluginHooks.ts` detects changes to plugin-affecting settings (`enabledPlugins`, `extraKnownMarketplaces`, `strictKnownMarketplaces`, `blockedMarketplaces`) and atomically swaps registered hooks.

---

## Skills Framework

### Skill Definition Model

Skills are markdown files with YAML frontmatter that define slash commands. The system supports two directory conventions:

**Legacy format (commands/):**
```
~/.claude/commands/build.md      -> /build
.claude/commands/deploy.md       -> /deploy
```

**Modern format (skills/):**
```
~/.claude/skills/my-skill/SKILL.md     -> /my-skill
.claude/skills/test-runner/SKILL.md    -> /test-runner
```

### Skill Sources and Precedence

Skills are loaded from these locations (defined by `SettingSource`):

| Source | Path | Priority |
|--------|------|----------|
| Policy (managed) | `{managedPath}/.claude/skills/` | Highest |
| Flag (`--settings`) | Custom path | |
| Local (gitignored) | `.claude/skills/` (local settings context) | |
| Project (shared) | `.claude/skills/` | |
| User (global) | `~/.claude/skills/` | |
| Plugin | Plugin's `skills/` directory | |
| Bundled | Compiled into CLI binary | |
| MCP | Provided by MCP server prompts | Lowest |

Deduplication uses `realpath` resolution to detect the same file accessed through symlinks or overlapping parent directories.

### Frontmatter Schema

A skill's YAML frontmatter supports these fields (parsed by `parseSkillFrontmatterFields()` in `src/skills/loadSkillsDir.ts`):

```yaml
---
name: display-name              # Optional display name (default: filename)
description: "What this skill does"
when_to_use: "Trigger conditions for the Skill tool"
argument-hint: "[file] [options]"
arguments: ["file", "options"]  # Named argument substitution
allowed-tools:                  # Tools permitted during execution
  - Bash
  - Read
  - Write
model: claude-sonnet-4-6        # Model override (or "inherit")
disable-model-invocation: false # If true, no LLM call, just inject prompt
user-invocable: true            # If false, only the Skill tool can invoke
context: fork                   # "fork" runs in sub-agent; default inline
agent: test-runner              # Named agent to delegate to
effort: high                    # Reasoning effort level
shell: bash                     # Shell for !`...` inline commands
version: "1.0.0"
paths:                          # File path patterns for contextual activation
  - src/tests/**
  - "*.test.ts"
hooks:                          # Skill-scoped hooks (registered as session hooks)
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "echo pre-hook"
---

Your skill prompt content here.

Supports $ARGUMENTS, $0, $1 for argument substitution.
Supports ${CLAUDE_SKILL_DIR} for self-referencing.
Supports ${CLAUDE_SESSION_ID} for session context.
Supports !`shell command` for inline shell execution.
```

### Bundled Skills

Bundled skills are compiled into the CLI binary and registered programmatically via `registerBundledSkill()` in `src/skills/bundledSkills.ts`:

```typescript
type BundledSkillDefinition = {
  name: string
  description: string
  aliases?: string[]
  whenToUse?: string
  argumentHint?: string
  allowedTools?: string[]
  model?: string
  disableModelInvocation?: boolean
  userInvocable?: boolean
  isEnabled?: () => boolean
  hooks?: HooksSettings
  context?: 'inline' | 'fork'
  agent?: string
  files?: Record<string, string>  // Reference files extracted to disk on first use
  getPromptForCommand: (args: string, context: ToolUseContext) => Promise<ContentBlockParam[]>
}
```

The `files` field enables bundled skills to ship reference documents that are lazily extracted to `~/.claude/bundled-skills/{skillName}/` on first invocation. Extraction uses secure file writing (`O_NOFOLLOW | O_EXCL`, `0o600` permissions) with per-process nonce directories to prevent symlink attacks.

### Skill Invocation Flow

Skills are invoked either by the user typing `/{skill-name}` or by the model calling the `Skill` tool:

```
User types /build          Model calls Skill tool
     |                            |
     v                            v
processSlashCommand()      SkillTool.call()
     |                            |
     +----> findCommand(name) <---+
                  |
                  v
           Command found?
                  |
          Yes     |     No
           |      |      |
           v      |      v
     getPromptForCommand()  Error
           |
           v
     substituteArguments()
           |
           v
     executeShellCommandsInPrompt()
     (for non-MCP skills only)
           |
           v
     Inject into conversation
     (inline or fork based on context)
```

### MCP Skills Integration

MCP servers can provide skills via their `prompts/list` endpoint. `src/skills/mcpSkillBuilders.ts` provides a write-once registry that breaks circular dependencies between the MCP client and skill loading:

```typescript
type MCPSkillBuilders = {
  createSkillCommand: typeof createSkillCommand
  parseSkillFrontmatterFields: typeof parseSkillFrontmatterFields
}
```

Registration happens at `loadSkillsDir.ts` module init (evaluated eagerly at startup). MCP-sourced skills have `loadedFrom: 'mcp'` and are blocked from executing inline shell commands (`!`\`...\``) for security -- they are remote and untrusted.

### Skill Telemetry

Skill invocations emit analytics events via `logEvent()`:
- Skill name and source (bundled, user, project, plugin, MCP)
- Plugin provenance for marketplace skills (marketplace name, plugin name, official status)
- Argument usage patterns
- Model override usage

---

## Hooks System

### Overview

Hooks are user-configurable lifecycle event handlers that execute at specific points during Claude Code's operation. They are defined in `settings.json` (any settings source), skill frontmatter, plugins, or programmatically via the SDK.

### Hook Events

28 hook events are defined in `src/entrypoints/sdk/coreTypes.ts`:

| Event | When It Fires | Matcher Field |
|-------|--------------|---------------|
| `PreToolUse` | Before tool execution | `tool_name` |
| `PostToolUse` | After successful tool execution | `tool_name` |
| `PostToolUseFailure` | After tool execution failure | `tool_name` |
| `PermissionDenied` | After auto-mode classifier denies a tool call | `tool_name` |
| `PermissionRequest` | When permission is requested | `tool_name` |
| `Notification` | When notifications are sent | `notification_type` |
| `UserPromptSubmit` | When user submits a prompt | -- |
| `SessionStart` | When a new session starts | `source` |
| `SessionEnd` | When session ends | -- |
| `Stop` | Before Claude concludes its response | -- |
| `StopFailure` | After Stop hook failure | -- |
| `SubagentStart` | When a sub-agent starts | -- |
| `SubagentStop` | When a sub-agent stops | -- |
| `PreCompact` | Before context compaction | -- |
| `PostCompact` | After context compaction | -- |
| `Setup` | During initial setup | -- |
| `TeammateIdle` | When a teammate agent is idle | -- |
| `TaskCreated` | When a task is created | -- |
| `TaskCompleted` | When a task completes | -- |
| `Elicitation` | When an elicitation dialog opens | -- |
| `ElicitationResult` | When an elicitation completes | -- |
| `ConfigChange` | When configuration changes | -- |
| `WorktreeCreate` | When a git worktree is created | -- |
| `WorktreeRemove` | When a git worktree is removed | -- |
| `InstructionsLoaded` | When CLAUDE.md instructions are loaded | -- |
| `CwdChanged` | When working directory changes | -- |
| `FileChanged` | When a watched file changes | -- |

### Hook Exit Code Semantics

For `PreToolUse`:
- **Exit 0** -- Hook succeeded; stdout/stderr not shown
- **Exit 2** -- Blocking error: show stderr to model, block the tool call
- **Other** -- Show stderr to user only, continue with tool call

For `PostToolUse`:
- **Exit 0** -- Stdout shown in transcript mode (Ctrl+O)
- **Exit 2** -- Show stderr to model immediately
- **Other** -- Show stderr to user only

For `Stop`:
- **Exit 0** -- Stdout/stderr not shown
- **Exit 2** -- Show stderr to model and continue conversation
- **Other** -- Show stderr to user only

### Hook Types

Four persistent hook types are defined by `HookCommandSchema` in `src/schemas/hooks.ts`:

#### 1. Shell Command Hook (`type: "command"`)

```json
{
  "type": "command",
  "command": "npm test -- --bail",
  "if": "Bash(npm *)",
  "shell": "bash",
  "timeout": 30,
  "statusMessage": "Running tests...",
  "once": false,
  "async": false,
  "asyncRewake": false
}
```

- `if` -- Permission rule syntax filter (e.g., `Bash(git *)`) evaluated before spawning
- `shell` -- `"bash"` (default, uses `$SHELL`) or `"powershell"` (uses `pwsh`)
- `async` -- Run in background without blocking
- `asyncRewake` -- Run in background; exit code 2 wakes the model with a blocking error

#### 2. Prompt Hook (`type: "prompt"`)

```json
{
  "type": "prompt",
  "prompt": "Review this code change: $ARGUMENTS",
  "model": "claude-sonnet-4-6",
  "timeout": 60,
  "statusMessage": "Reviewing..."
}
```

Evaluates a prompt with an LLM. `$ARGUMENTS` is replaced with the hook input JSON.

#### 3. Agent Hook (`type: "agent"`)

```json
{
  "type": "agent",
  "prompt": "Verify that unit tests ran and passed.",
  "model": "claude-sonnet-4-6",
  "timeout": 60
}
```

Runs a full agentic verification loop. Returns structured output `{ ok: boolean, reason?: string }`.

#### 4. HTTP Hook (`type: "http"`)

```json
{
  "type": "http",
  "url": "https://hooks.example.com/pre-tool",
  "headers": {
    "Authorization": "Bearer $MY_TOKEN"
  },
  "allowedEnvVars": ["MY_TOKEN"],
  "timeout": 10
}
```

POSTs hook input JSON to a URL. Header values support `$VAR_NAME` interpolation, gated by the `allowedEnvVars` field. Enterprise policy can restrict target URLs via `allowedHttpHookUrls` and restrict env var exposure via `httpHookAllowedEnvVars`.

#### 5. Function Hook (runtime only, not persistable)

```typescript
type FunctionHook = {
  type: 'function'
  id?: string
  timeout?: number
  callback: (messages: Message[], signal?: AbortSignal) => boolean | Promise<boolean>
  errorMessage: string
  statusMessage?: string
}
```

Function hooks are in-memory callbacks registered via `addFunctionHook()`. They cannot be serialized to settings and are session-scoped only. Used internally for structured output enforcement and schema validation.

### Hook Configuration Schema

Hooks are configured in `settings.json` under the `hooks` key:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'pre-bash hook'",
            "if": "Bash(git *)"
          }
        ]
      }
    ],
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "agent",
            "prompt": "Verify tests pass"
          }
        ]
      }
    ]
  }
}
```

### Hook Sources and Priority

Hooks are aggregated from multiple sources by `getAllHooks()` in `src/utils/hooks/hooksSettings.ts`:

```
Hook Source Priority (all fire, not override):

  1. Policy (managed) settings     -- Enterprise-mandated hooks
  2. User settings (~/.claude/)    -- Global user hooks
  3. Project settings              -- Per-project hooks
  4. Local settings                -- Gitignored local hooks
  5. Plugin hooks                  -- From enabled plugins
  6. Session hooks                 -- Programmatic/skill-registered hooks
  7. Built-in hooks                -- Internal system hooks
```

**Enterprise control:**
- `allowManagedHooksOnly: true` in managed settings disables all non-managed hooks
- `disableAllHooks: true` disables all hooks globally

### Hook Execution Chain

```
Event fires (e.g., PreToolUse for Bash)
     |
     v
getAllHooks(event)
     |
     v
sortMatchersByPriority()
     |
     v
For each matcher:
     |
     +---> Does matcher string match? (e.g., "Bash" matches tool_name)
     |         No -> skip
     |         Yes -> continue
     |
     +---> Does `if` condition match? (permission rule syntax)
     |         No -> skip hook (no process spawned)
     |         Yes -> continue
     |
     +---> Execute hook based on type:
     |       command -> spawn shell process
     |       prompt  -> LLM inference call
     |       agent   -> agentic loop with tools
     |       http    -> POST to URL
     |       function-> in-process callback
     |
     +---> Collect result (exit code, stdout, stderr)
     |
     v
Aggregate results:
  - Any exit code 2? -> Blocking error
  - All exit code 0? -> Success
  - Otherwise        -> Non-blocking warning
```

### Skill-Scoped Hooks

Skills can define hooks in their frontmatter that are registered as session hooks when the skill is invoked. `registerSkillHooks()` in `src/utils/hooks/registerSkillHooks.ts` processes the skill's `hooks` frontmatter field and calls `addSessionHook()` for each entry.

If a hook has `once: true`, an `onHookSuccess` callback is registered that removes the hook after its first successful execution.

### Plugin Hook Integration

`loadPluginHooks()` in `src/utils/plugins/loadPluginHooks.ts`:

1. Calls `loadAllPluginsCacheOnly()` to get enabled plugins
2. Converts each plugin's `hooksConfig` to `PluginHookMatcher[]` (adds `pluginRoot` and `pluginName` context)
3. Atomically clears old plugin hooks and registers new ones via `clearRegisteredPluginHooks()` + `registerHookCallbacks()`
4. Supports hot reload when policy settings change

The atomic clear-then-register pattern prevents a window where plugin hooks are missing (fixed in gh-29767).

---

## Settings & Configuration

### Settings Hierarchy

Settings are loaded from five sources, defined in `src/utils/settings/constants.ts`:

```
Source              File Location                           Precedence
------              -------------                           ----------
userSettings        ~/.claude/settings.json                 Lowest
projectSettings     .claude/settings.json (shared)          |
localSettings       .claude/settings.local.json (gitignored)|
flagSettings        --settings <path>                       |
policySettings      managed-settings.json + drop-ins        Highest
                    + remote managed settings (API)
```

**Merge semantics:** Later sources override earlier ones (via `mergeWith` with custom `settingsMergeCustomizer`). Array fields like `allowedMcpServers` merge across sources.

**Always-active sources:** `policySettings` and `flagSettings` cannot be disabled. Other sources can be restricted via `--setting-sources` CLI flag.

### Managed Settings

Enterprise managed settings come from two places:

1. **File-based:** `{managedPath}/managed-settings.json` + `{managedPath}/managed-settings.d/*.json` drop-ins
2. **Remote:** Synced from API (cached in `syncCacheState`)

Drop-in files are sorted alphabetically and merged on top of the base file (following systemd/sudoers convention).

Platform-specific managed paths:
- **macOS:** `/Library/Application Support/ClaudeCode/`
- **Linux:** `/etc/claude-code/`
- **Windows:** Registry (HKCU) + MDM settings

### Settings Schema

The full settings schema is defined in `src/utils/settings/types.ts` via `SettingsSchema`. Key extensibility-relevant fields:

```typescript
{
  // Hooks configuration
  hooks?: HooksSettings

  // Plugin management
  enabledPlugins?: Record<string, boolean | string[] | undefined>
  extraKnownMarketplaces?: Record<string, {
    source: MarketplaceSource
    installLocation?: string
    autoUpdate?: boolean
  }>

  // Enterprise plugin policy
  strictKnownMarketplaces?: MarketplaceSource[]
  blockedMarketplaces?: MarketplaceSource[]
  strictPluginOnlyCustomization?: boolean | ('skills' | 'agents' | 'hooks' | 'mcp')[]

  // Hook policy
  allowManagedHooksOnly?: boolean
  disableAllHooks?: boolean
  allowedHttpHookUrls?: string[]
  httpHookAllowedEnvVars?: string[]

  // MCP server policy
  enableAllProjectMcpServers?: boolean
  enabledMcpjsonServers?: string[]
  disabledMcpjsonServers?: string[]
  allowedMcpServers?: AllowedMcpServerEntry[]
  deniedMcpServers?: DeniedMcpServerEntry[]

  // Permissions
  permissions?: {
    allow?: PermissionRule[]
    deny?: PermissionRule[]
    ask?: PermissionRule[]
    defaultMode?: PermissionMode
    disableBypassPermissionsMode?: 'disable'
    additionalDirectories?: string[]
  }

  // Environment
  env?: Record<string, string>
}
```

### Settings Caching and Change Detection

- **Parse caching:** `parseSettingsFile()` caches parsed results keyed by file path. Callers receive cloned copies to prevent mutation of cached entries.
- **Session-level caching:** `getSettings_DEPRECATED()` aggregates all sources with memoization.
- **Change detection:** `settingsChangeDetector` in `src/utils/settings/changeDetector.ts` publishes change events by source. Subscribers (like plugin hook hot reload) can react to specific source changes.

### Settings Validation

Settings files are validated against `SettingsSchema` using Zod. Invalid fields are **preserved in the file** (not deleted) so users can fix them. Invalid permission rules are filtered before schema validation to prevent one bad rule from rejecting the entire file.

---

## Memory Directory System

### Architecture

The memory system (`src/memdir/`) provides Claude Code with persistent, file-based memory across sessions. It is organized as a hierarchical directory of markdown files with YAML frontmatter.

### Directory Structure

```
~/.claude/projects/{sanitized-git-root}/memory/
  MEMORY.md                    # Index/entrypoint (always loaded)
  user_role.md                 # Individual memory file
  feedback_testing.md          # Individual memory file
  project_api_migration.md     # Individual memory file
  logs/                        # Daily logs (assistant mode)
    2026/
      04/
        2026-04-03.md
```

### Path Resolution

Memory directory paths are resolved by `getAutoMemPath()` in `src/memdir/paths.ts`:

1. **`CLAUDE_COWORK_MEMORY_PATH_OVERRIDE`** env var -- Full-path override (Cowork SDK)
2. **`autoMemoryDirectory`** in settings -- User/policy settings (supports `~/` expansion)
3. **Default:** `~/.claude/projects/{sanitized-git-root}/memory/`

**Security constraints on path overrides:**
- `projectSettings` is excluded from `autoMemoryDirectory` to prevent malicious repos from redirecting writes to sensitive directories
- Paths must be absolute, >= 3 chars, no UNC paths, no null bytes
- `~` alone and `~/` are rejected (would match all of `$HOME`)

Git worktrees share memory through `findCanonicalGitRoot()` so all worktrees of the same repo use one memory directory.

### Memory Type Taxonomy

Four memory types are defined in `src/memdir/memoryTypes.ts`:

| Type | Scope | Purpose |
|------|-------|---------|
| `user` | Always private | User's role, goals, preferences, knowledge |
| `feedback` | Default private, team for project-wide conventions | Guidance about approach -- corrections and confirmations |
| `project` | Bias toward team | Ongoing work, goals, bugs, incidents not in code/git |
| `reference` | Private or team | External knowledge, API docs, tool usage notes |

Content derivable from the current project state (code patterns, architecture, git history) is explicitly excluded.

### Memory File Format

```markdown
---
name: Testing preferences
description: User prefers integration tests over mocks
type: feedback
---

Use real database connections in integration tests, not mocks.

**Why:** Prior incident where mock/prod divergence masked a broken migration.
**How to apply:** Any test file under tests/integration/.
```

### MEMORY.md Entrypoint

`MEMORY.md` is the index file, always loaded into the system prompt. It has strict limits:

- **Line cap:** 200 lines (`MAX_ENTRYPOINT_LINES`)
- **Byte cap:** 25,000 bytes (`MAX_ENTRYPOINT_BYTES`)

Truncation appends a warning. Entries should be one-line pointers: `- [Title](file.md) -- one-line hook`.

### Memory Recall (Query-Time)

`findRelevantMemories()` in `src/memdir/findRelevantMemories.ts` implements smart memory recall:

1. **Scan:** `scanMemoryFiles()` reads frontmatter from all `.md` files in memory dir (recursive, capped at 200 files, sorted by mtime desc)
2. **Select:** A side-query to Sonnet evaluates the user's query against memory file descriptions and selects up to 5 relevant files
3. **Filter:** Previously surfaced memories are excluded from selection
4. **Return:** Absolute paths and mtime for the main model to read on demand

### Memory Freshness

`src/memdir/memoryAge.ts` provides staleness warnings:
- Memories > 1 day old get a caveat: "This memory is N days old... claims about code behavior or file:line citations may be outdated."
- Warnings are injected via `<system-reminder>` tags

### Auto-Memory Controls

Auto-memory features can be disabled via:
- `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` env var
- `CLAUDE_CODE_SIMPLE` (bare mode)
- `autoMemoryEnabled: false` in settings.json
- Remote sessions without `CLAUDE_CODE_REMOTE_MEMORY_DIR`

---

## Data Migrations

### Overview

The migration system in `src/migrations/` handles one-time data transformations when Claude Code updates. Migrations run synchronously at startup in `src/main.tsx`.

### Migration Pattern

Each migration is a standalone function that:
1. Checks if migration is needed (reads current state)
2. Transforms data (settings, config)
3. Writes the new state
4. Emits telemetry event
5. Cleans up old data

```typescript
// Typical migration pattern:
export function migrateFooToBar(): void {
  const currentSettings = getSettingsForSource('userSettings')

  // Guard: skip if already migrated
  if (!currentSettings?.oldField) return

  try {
    updateSettingsForSource('userSettings', {
      ...currentSettings,
      newField: transformedValue,
    })

    // Clean up old config
    saveGlobalConfig(current => {
      const { oldField: _, ...rest } = current
      return rest
    })

    logEvent('tengu_migrate_foo_to_bar', { success: true })
  } catch (error) {
    logError(new Error(`Migration failed: ${error}`))
  }
}
```

### Current Migrations

| Migration | Purpose |
|-----------|---------|
| `migrateAutoUpdatesToSettings` | Move auto-update preference from global config to settings.json env var |
| `migrateBypassPermissionsAcceptedToSettings` | Move bypass permissions acceptance to settings |
| `migrateEnableAllProjectMcpServersToSettings` | Move MCP server approval state to settings |
| `migrateFennecToOpus` | Model string migration (Fennec -> Opus) |
| `migrateLegacyOpusToCurrent` | Model string migration (legacy Opus -> current) |
| `migrateOpusToOpus1m` | Model string migration (Opus -> Opus 1M) |
| `migrateSonnet1mToSonnet45` | Model string migration (Sonnet 1M -> 4.5) |
| `migrateSonnet45ToSonnet46` | Model string migration (Sonnet 4.5 -> 4.6) |
| `migrateReplBridgeEnabledToRemoteControlAtStartup` | Feature flag migration |
| `resetAutoModeOptInForDefaultOffer` | Reset auto-mode opt-in state |
| `resetProToOpusDefault` | Reset Pro tier default model to Opus |

Migrations run in a specific order annotated by `@[MODEL LAUNCH]` comments to remind engineers to add model string migrations when launching new models.

---

## Extension Point Summary

### For Plugin Authors

| Extension Point | Mechanism | Discovery |
|----------------|-----------|-----------|
| Slash commands | Markdown files in `commands/` or `skills/` | Auto-discovered from plugin directory |
| Agent definitions | Markdown files in `agents/` | Auto-discovered from plugin directory |
| Lifecycle hooks | `hooks/hooks.json` or manifest `hooks` field | Registered on plugin load |
| MCP servers | Manifest `mcpServers` field or `.mcpb`/`.dxt` bundles | Started on plugin enable |
| LSP servers | Manifest `lspServers` field | Started on plugin enable |
| Output styles | Files in `output-styles/` | Auto-discovered from plugin directory |
| Settings overlays | Manifest `settings` field | Merged at load time |

### For Enterprise Administrators

| Control | Settings Key | Effect |
|---------|-------------|--------|
| Plugin marketplace allowlist | `strictKnownMarketplaces` | Only listed sources can be added |
| Plugin marketplace blocklist | `blockedMarketplaces` | Listed sources are blocked |
| Force plugin-only customization | `strictPluginOnlyCustomization` | Block non-plugin skills/hooks/MCP/agents |
| Managed hooks only | `allowManagedHooksOnly` | Ignore user/project/local hooks |
| Managed permission rules only | `allowManagedPermissionRulesOnly` | Ignore non-managed permission rules |
| HTTP hook URL allowlist | `allowedHttpHookUrls` | Restrict HTTP hook target URLs |
| HTTP hook env var allowlist | `httpHookAllowedEnvVars` | Restrict env var exposure in HTTP headers |
| MCP server allowlist | `allowedMcpServers` | Restrict which MCP servers can be used |
| MCP server denylist | `deniedMcpServers` | Block specific MCP servers |
| Model allowlist | `availableModels` | Restrict selectable models |
| Disable all hooks | `disableAllHooks` | Suppress all hook execution |

### For SDK Integrators

| API | Module | Purpose |
|-----|--------|---------|
| `registerBundledSkill()` | `skills/bundledSkills.ts` | Add a compiled-in skill |
| `registerBuiltinPlugin()` | `plugins/builtinPlugins.ts` | Add a toggleable built-in plugin |
| `addSessionHook()` | `utils/hooks/sessionHooks.ts` | Add a runtime hook (command/prompt) |
| `addFunctionHook()` | `utils/hooks/sessionHooks.ts` | Add a runtime callback hook |
| `registerHookCallbacks()` | `bootstrap/state.ts` | Register plugin-level hook matchers |
| `registerMCPSkillBuilders()` | `skills/mcpSkillBuilders.ts` | Register MCP skill creation functions |

### Configuration File Locations

```
~/.claude/
  settings.json              # User settings (hooks, plugins, permissions)
  skills/                    # User skills
    my-skill/SKILL.md
  commands/                  # Legacy user commands
    my-command.md
  plugins/
    known_marketplaces.json  # Marketplace registry
    marketplaces/            # Cached marketplace manifests
    cache/                   # Installed plugin files
  projects/
    {sanitized-path}/
      memory/                # Auto-memory directory
        MEMORY.md
  bundled-skills/            # Extracted bundled skill files
    {skillName}/

.claude/                     # Project-level (in repo root)
  settings.json              # Project settings (shared)
  settings.local.json        # Local settings (gitignored)
  skills/                    # Project skills
  commands/                  # Legacy project commands

{managedPath}/               # Enterprise managed
  managed-settings.json      # Base managed settings
  managed-settings.d/        # Drop-in settings fragments
    10-security.json
    20-otel.json
```
