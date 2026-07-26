# 13 — Complete src subsystem map

The other chapters explain how the program behaves. This chapter guarantees **coverage**: it accounts for every top-level directory and every root file under [`src/`](../../../src/), says what each is for, whether the code is real or a build stub in this checkout, and which chapter documents it.

Use this page to confirm that no part of the source is silently ignored. For individual files, use [07 — Source atlas](07-source-atlas.md); for symbols, use [09 — Core function reference](09-core-function-reference.md).

## 1. How to read this map

- **Real / Stub** reflects this checked-in source. A “stub” file carries the header `@generated stub from scan-missing-imports`: it is a placeholder for an internal, feature-gated module whose real implementation is not present. Stubs let the build resolve imports; they are not working features here.
- **Covered by** points to the chapter that explains the subsystem. Where a subsystem had no prior chapter, this page is its description.
- Counts exclude `*.test.*` files and can shift as the source changes; regenerate if needed.

## 2. Root files of `src/`

| File | Purpose | Covered by |
|---|---|---|
| [`main.tsx`](../../../src/main.tsx) | CLI entry: options, initialization, mode selection | [02](02-startup-and-one-turn.md), [09](09-core-function-reference.md) |
| [`setup.ts`](../../../src/setup.ts) | Working-directory/trust/setup before dependent code | [02](02-startup-and-one-turn.md) |
| [`replLauncher.tsx`](../../../src/replLauncher.tsx) | Mounts the interactive terminal app | [02](02-startup-and-one-turn.md), [06](06-terminal-ui-commands.md) |
| [`interactiveHelpers.tsx`](../../../src/interactiveHelpers.tsx) | Render/run helpers, setup screens, error exits | [02](02-startup-and-one-turn.md) |
| [`dialogLaunchers.tsx`](../../../src/dialogLaunchers.tsx) | Launchers for resume/settings/teleport dialogs | [06](06-terminal-ui-commands.md) |
| [`query.ts`](../../../src/query.ts) | Agentic-turn state machine | [03](03-agent-loop.md), [09](09-core-function-reference.md) |
| [`QueryEngine.ts`](../../../src/QueryEngine.ts) | Conversation owner across turns | [01](01-orientation.md), [09](09-core-function-reference.md) |
| [`Tool.ts`](../../../src/Tool.ts) | Tool contract and `ToolUseContext` | [04](04-tools-permissions-security.md), [08](08-complete-tool-catalog.md) |
| [`tools.ts`](../../../src/tools.ts) | Tool registration and pool assembly | [08](08-complete-tool-catalog.md) |
| [`commands.ts`](../../../src/commands.ts) | Command registry (built-in + skills + plugins + MCP) | [06](06-terminal-ui-commands.md) |
| [`Task.ts`](../../../src/Task.ts), [`tasks.ts`](../../../src/tasks.ts) | Task/todo primitives used by the loop and UI | [03](03-agent-loop.md), [06](06-terminal-ui-commands.md) |
| [`context.ts`](../../../src/context.ts) | System/user context assembly (memoized) | [05](05-context-persistence-services.md) |
| [`cost-tracker.ts`](../../../src/cost-tracker.ts), [`costHook.ts`](../../../src/costHook.ts) | Cost/usage accounting | [05](05-context-persistence-services.md) |
| [`history.ts`](../../../src/history.ts) | Command history and paste storage | [06](06-terminal-ui-commands.md) |
| [`ink.ts`](../../../src/ink.ts) | Re-export surface for the Ink renderer | [06](06-terminal-ui-commands.md) |
| [`projectOnboardingState.ts`](../../../src/projectOnboardingState.ts) | First-run/onboarding state for a project | [02](02-startup-and-one-turn.md) |
| [`localRecoveryCli.ts`](../../../src/localRecoveryCli.ts) | Local recovery entry (the mode the initial fixes re-routed away from) | [01](01-orientation.md) |

## 3. Top-level directories

### Core execution and model

| Directory | Real/Stub | Purpose | Covered by |
|---|---|---|---|
| [`entrypoints/`](../../../src/entrypoints/) | 12 real / 2 stub | CLI, print/headless, SDK type surface, init | [02](02-startup-and-one-turn.md) |
| [`query/`](../../../src/query/) | 4 real / 1 stub | Query config, transitions, stop hooks, token budget | [03](03-agent-loop.md), [09](09-core-function-reference.md) |
| [`bootstrap/`](../../../src/bootstrap/) | 1 real | Global session state store | [09](09-core-function-reference.md) |
| [`state/`](../../../src/state/) | 6 real | AppState store and change handling | [01](01-orientation.md), [09](09-core-function-reference.md) |
| [`context/`](../../../src/context/) | 9 real | Stats/context providers for the UI/session | [05](05-context-persistence-services.md) |

### Tools and safety

| Directory | Real/Stub | Purpose | Covered by |
|---|---|---|---|
| [`tools/`](../../../src/tools/) | 189 real / 25 stub | All tool implementations | [08](08-complete-tool-catalog.md), [04](04-tools-permissions-security.md) |
| [`hooks/`](../../../src/hooks/) | 104 real / 2 stub | React hooks and the tool-permission machinery | [04](04-tools-permissions-security.md), [09](09-core-function-reference.md) |
| [`schemas/`](../../../src/schemas/) | 1 real | Extracted hook Zod schemas (breaks import cycles) | [04](04-tools-permissions-security.md) |

Bash security specifically is documented in [11 — Bash security checks in depth](11-bash-security-checks.md); the parser/permission helpers live under [`utils/bash/`](../../../src/utils/bash/) and [`tools/BashTool/`](../../../src/tools/BashTool/).

### Services

| Directory | Real/Stub | Purpose | Covered by |
|---|---|---|---|
| [`services/`](../../../src/services/) | 151 real / 19 stub | Model API, MCP, LSP, compaction, analytics, plugins, memory, auth | [05](05-context-persistence-services.md), [12](12-context-and-compaction.md), [09](09-core-function-reference.md) |

Notable service groups (see section 5):

- [`services/api/`](../../../src/services/api/) — model requests, streaming, retry, watchdog, errors.
- [`services/compact/`](../../../src/services/compact/), [`services/contextCollapse/`](../../../src/services/contextCollapse/) — context management ([12](12-context-and-compaction.md)).
- [`services/mcp/`](../../../src/services/mcp/), [`services/lsp/`](../../../src/services/lsp/) — external tool/language protocols ([19](19-mcp-and-lsp.md)).
- [`services/analytics/`](../../../src/services/analytics/) — telemetry with privacy markers.
- [`services/oauth/`](../../../src/services/oauth/), [`grokAuth/`](../../../src/services/grokAuth/), [`openaiAuth/`](../../../src/services/openaiAuth/) — provider auth.
- [`services/SessionMemory/`](../../../src/services/SessionMemory/), [`extractMemories/`](../../../src/services/extractMemories/), [`autoDream/`](../../../src/services/autoDream/), [`teamMemorySync/`](../../../src/services/teamMemorySync/) — memory features ([15](15-memory-system.md); repo guide [`docs/memory/`](../memory/)).
- [`services/skillSearch/`](../../../src/services/skillSearch/), [`tips/`](../../../src/services/tips/), [`PromptSuggestion/`](../../../src/services/PromptSuggestion/), [`toolUseSummary/`](../../../src/services/toolUseSummary/), [`AgentSummary/`](../../../src/services/AgentSummary/) — assistive/UX services.
- [`services/policyLimits/`](../../../src/services/policyLimits/), [`remoteManagedSettings/`](../../../src/services/remoteManagedSettings/), [`settingsSync/`](../../../src/services/settingsSync/) — managed policy/config.

### Local server and remote access

| Directory | Real/Stub | Purpose | Covered by |
|---|---|---|---|
| [`server/`](../../../src/server/) | 147 real / 8 stub | Local HTTP/WebSocket API, provider proxy, session services | [18](18-local-server-and-proxy.md) |
| [`bridge/`](../../../src/bridge/) | 31 real / 2 stub | Remote-control bridge client (CCR/remote message relay over HTTP) | [17](17-remote-access-and-bridge.md) or repo [`docs/channel/`](../channel/) |
| [`remote/`](../../../src/remote/) | 4 real | Remote session manager handling SDK control (cancel/permission) requests | [17](17-remote-access-and-bridge.md) |
| [`upstreamproxy/`](../../../src/upstreamproxy/) | 2 real | CONNECT-over-WebSocket relay used by the CCR upstream proxy | [17](17-remote-access-and-bridge.md) |
| [`daemon/`](../../../src/daemon/) | 0 real / 2 stub | Background daemon entry (internal, gated) | stub only |
| [`ssh/`](../../../src/ssh/) | 0 real / 2 stub | SSH session manager (internal, gated) | stub only |
| [`self-hosted-runner/`](../../../src/self-hosted-runner/) | 0 real / 1 stub | Self-hosted runner entry (internal, gated) | stub only |
| [`environment-runner/`](../../../src/environment-runner/) | 0 real / 1 stub | Environment runner entry (internal, gated) | stub only |

### Terminal interface

| Directory | Real/Stub | Purpose | Covered by |
|---|---|---|---|
| [`screens/`](../../../src/screens/) | 3 real | REPL and top-level screens | [06](06-terminal-ui-commands.md) |
| [`components/`](../../../src/components/) | 402 real / 18 stub | Ink UI components (prompt, messages, settings, dialogs) | [06](06-terminal-ui-commands.md) |
| [`ink/`](../../../src/ink/) | 96 real / 4 stub | Custom React-to-terminal renderer and primitives | [06](06-terminal-ui-commands.md) |
| [`keybindings/`](../../../src/keybindings/) | 14 real / 1 stub | Keybinding context and disambiguation | [06](06-terminal-ui-commands.md) |
| [`vim/`](../../../src/vim/) | 5 real | Vim-mode input behavior | [06](06-terminal-ui-commands.md) |
| [`buddy/`](../../../src/buddy/) | 7 real | Terminal companion sprite/animation ("pet") | this chapter; repo [`docs/desktop/pets.md`](../desktop/pets.md) |
| [`cli/`](../../../src/cli/) | 19 real / 4 stub | Headless/print output rendering and CLI helpers | [02](02-startup-and-one-turn.md) |

### Commands, skills, plugins, output

| Directory | Real/Stub | Purpose | Covered by |
|---|---|---|---|
| [`commands/`](../../../src/commands/) | 195 real / 16 stub | Slash-command implementations and dialogs | [06](06-terminal-ui-commands.md) |
| [`skills/`](../../../src/skills/) | 21 real / 4 stub | Skill discovery/loading and bundled skills | [06 §5](06-terminal-ui-commands.md) |
| [`plugins/`](../../../src/plugins/) | 2 real | Built-in plugin registry | [06 §5](06-terminal-ui-commands.md) |
| [`outputStyles/`](../../../src/outputStyles/) | 1 real | Output-style discovery/loading | this chapter |
| [`moreright/`](../../../src/moreright/) | 1 real (external stub) | Placeholder for an internal-only UI hook | this chapter |

### Persistence, config, migrations

| Directory | Real/Stub | Purpose | Covered by |
|---|---|---|---|
| [`migrations/`](../../../src/migrations/) | 11 real | Forward migrations for models/settings/permissions | [05 §13](05-context-persistence-services.md) |
| [`memdir/`](../../../src/memdir/) | 8 real / 1 stub | Memory-directory loading and relevant-memory retrieval | [15](15-memory-system.md) or repo [`docs/memory/`](../memory/) |
| [`constants/`](../../../src/constants/) | 23 real / 1 stub | Shared constants (tools, oauth, product, xml, etc.) | referenced throughout |
| [`types/`](../../../src/types/) | 12 real / 7 stub | Shared type definitions (messages, permissions, hooks, logs) | [01](01-orientation.md), [04](04-tools-permissions-security.md) |

### Agents, teams, and alternate modes

| Directory | Real/Stub | Purpose | Covered by |
|---|---|---|---|
| [`tasks/`](../../../src/tasks/) | 12 real / 2 stub | Backgrounding the main session (Ctrl+B) and local task runners | this chapter |
| [`coordinator/`](../../../src/coordinator/) | 1 real / 1 stub | Coordinator-mode context and tool filtering | [08 §7](08-complete-tool-catalog.md) |
| [`goals/`](../../../src/goals/) | 1 real | Goal-state tracking for a session | this chapter |
| [`assistant/`](../../../src/assistant/) | 1 real / 4 stub | KAIROS assistant mode (mostly internal/gated) | stub-heavy |
| [`proactive/`](../../../src/proactive/) | 0 real / 2 stub | Proactive mode (internal, gated) | stub only |
| [`jobs/`](../../../src/jobs/) | 0 real / 1 stub | Template job classifier (internal, gated) | stub only |
| [`voice/`](../../../src/voice/) | 1 real | Voice-mode enablement gate | this chapter |

Multi-agent orchestration itself (the `Agent` tool, subagents, teams) is implemented under [`tools/AgentTool/`](../../../src/tools/AgentTool/) and [`utils/swarm/`](../../../src/utils/swarm/); see [14 — Multi-agent, subagents, and teams](14-multi-agent-and-teams.md) (repo guide: [`docs/agent/`](../agent/)).

### Computer Use and native

| Directory | Real/Stub | Purpose | Covered by |
|---|---|---|---|
| [`vendor/`](../../../src/vendor/) | 12 real | Vendored `computer-use-mcp` implementation (screenshot/click/type tool calls) | [16](16-computer-use.md) |
| [`native-ts/`](../../../src/native-ts/) | 4 real | TypeScript bindings/enums for native helpers | [16](16-computer-use.md) |

The Computer Use platform helpers themselves live outside `src/` in [`runtime/`](../../../runtime/) (Python) and are described by the repo guide [`docs/features/computer-use-architecture.md`](../features/computer-use-architecture.md).

### Utilities

| Directory | Real/Stub | Purpose | Covered by |
|---|---|---|---|
| [`utils/`](../../../src/utils/) | 582 real / 13 stub | Shared runtime utilities (see section 4) | many chapters |

## 4. Notable `utils/` groups

`utils/` is the largest area. It is not one subsystem but a collection of focused helper groups. The most important for behavior:

| Group | Purpose | Covered by |
|---|---|---|
| [`utils/bash/`](../../../src/utils/bash/) | Bash parser and AST-based security | [11](11-bash-security-checks.md) |
| [`utils/permissions/`](../../../src/utils/permissions/) | Permission modes, rules, filesystem scope | [04](04-tools-permissions-security.md) |
| [`utils/sandbox/`](../../../src/utils/sandbox/) | Sandbox adapter integration | [04 §6](04-tools-permissions-security.md) |
| [`utils/messages/`](../../../src/utils/messages/) | Message construction/normalization/mappers | [09](09-core-function-reference.md) |
| [`utils/processUserInput/`](../../../src/utils/processUserInput/) | Slash-command/hook/attachment input processing | [02](02-startup-and-one-turn.md), [06](06-terminal-ui-commands.md) |
| [`utils/model/`](../../../src/utils/model/) | Model resolution/capabilities/strings | [02](02-startup-and-one-turn.md) |
| [`utils/settings/`](../../../src/utils/settings/) | Settings sources, MDM/managed policy, validation | [05 §2](05-context-persistence-services.md) |
| [`utils/hooks/`](../../../src/utils/hooks/) | Hook event execution helpers | [04 §8](04-tools-permissions-security.md) |
| [`utils/swarm/`](../../../src/utils/swarm/) | Teammate/team orchestration support | [14](14-multi-agent-and-teams.md) |
| [`utils/memory/`](../../../src/utils/memory/), [`utils/todo/`](../../../src/utils/todo/), [`utils/task/`](../../../src/utils/task/) | Memory/todo/task helpers | [15](15-memory-system.md), [03](03-agent-loop.md) |
| [`utils/computerUse/`](../../../src/utils/computerUse/), [`utils/claudeInChrome/`](../../../src/utils/claudeInChrome/) | Computer Use / browser integration | [16](16-computer-use.md) or  repo [`docs/features/`](../features/) |
| [`utils/git/`](../../../src/utils/git/), [`utils/github/`](../../../src/utils/github/), [`utils/teleport/`](../../../src/utils/teleport/), [`utils/worktree*`](../../../src/utils/) | Git/GitHub/worktree/teleport support | [05](05-context-persistence-services.md) |
| [`utils/secureStorage/`](../../../src/utils/secureStorage/), [`utils/telemetry/`](../../../src/utils/telemetry/) | Keychain prefetch and telemetry | [02](02-startup-and-one-turn.md), [05 §12](05-context-persistence-services.md) |

Other `utils/` groups (background, deepLink, dxt, filePersistence, nativeInstaller, powershell, shell, skills, suggestions, ultraplan) are supporting helpers used by the subsystems above and are enumerated file-by-file in the [atlas](07-source-atlas.md).

## 5. Notable `services/` groups

| Group | Purpose | Covered by |
|---|---|---|
| [`services/api/`](../../../src/services/api/) | Provider requests, streaming, retry, watchdog, errors | [05 §4](05-context-persistence-services.md), [09](09-core-function-reference.md) |
| [`services/compact/`](../../../src/services/compact/) | Auto/micro/session-memory compaction | [12](12-context-and-compaction.md) |
| [`services/contextCollapse/`](../../../src/services/contextCollapse/) | Structured context collapse/replay | [12 §6](12-context-and-compaction.md) |
| [`services/mcp/`](../../../src/services/mcp/) | MCP client, config, transports, auth | [19](19-mcp-and-lsp.md) |
| [`services/lsp/`](../../../src/services/lsp/) | Language-server manager | [19](19-mcp-and-lsp.md) |
| [`services/analytics/`](../../../src/services/analytics/) | Telemetry sink, growthbook gates, privacy markers | [05 §12](05-context-persistence-services.md) |
| [`services/plugins/`](../../../src/services/plugins/) | Plugin CLI/install/management services | [06 §5](06-terminal-ui-commands.md) |

## 6. `server/` structure

The local server ([`server/index.ts`](../../../src/server/index.ts)) is documented in depth in [18 — Local server and provider proxy](18-local-server-and-proxy.md) (summary also in [05 §11](05-context-persistence-services.md)). Its subdirectories:

| Subdir | Purpose |
|---|---|
| [`server/api/`](../../../src/server/api/) | REST resource handlers (sessions, providers, MCP, etc.) |
| [`server/ws/`](../../../src/server/ws/) | WebSocket protocol, handler lifecycle, events |
| [`server/services/`](../../../src/server/services/) | Session service, conversation service, local index, persistence migrations |
| [`server/proxy/`](../../../src/server/proxy/) | Provider protocol transforms (Anthropic ↔ OpenAI chat/responses) |
| [`server/backends/`](../../../src/server/backends/) | Backend integrations behind the API |
| [`server/config/`](../../../src/server/config/) | Provider presets/config |
| [`server/middleware/`](../../../src/server/middleware/) | Auth and CORS middleware |
| [`server/types/`](../../../src/server/types/) | Server-side types (provider, requests) |

## 7. Fully-stubbed subsystems

These directories contain only generated stubs in this checkout. Their real behavior cannot be described or verified from this source, only named:

- [`daemon/`](../../../src/daemon/) — background daemon entry.
- [`ssh/`](../../../src/ssh/) — SSH session manager.
- [`self-hosted-runner/`](../../../src/self-hosted-runner/) — self-hosted runner entry.
- [`environment-runner/`](../../../src/environment-runner/) — environment runner entry.
- [`proactive/`](../../../src/proactive/) — proactive mode.
- [`jobs/`](../../../src/jobs/) — template job classifier.

Additional stubbed modules appear inside otherwise-real directories (for example parts of `tools/`, `services/`, `assistant/`, `coordinator/`). The [tool catalog](08-complete-tool-catalog.md) already lists the stubbed tools individually.

## 8. Completeness statement

Every top-level directory and root file under [`src/`](../../../src/) appears in this chapter with a purpose and a coverage pointer. This is directory-level completeness; file-level completeness is provided by the [source atlas](07-source-atlas.md), and symbol-level detail by the [core function reference](09-core-function-reference.md).

What this map deliberately does **not** claim:

- it does not reproduce source code;
- it does not invent behavior for generated stubs;
- it does not restate the arbitrary contents of external MCP servers or plugins;
- it does not duplicate the repo’s standalone guides for agents, memory, skills, Computer Use, IM/channels, and desktop, which remain the deeper references for those product areas.
