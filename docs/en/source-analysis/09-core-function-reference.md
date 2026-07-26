# 09 — Core function reference

This is a behavior-to-source map for the executable core. It complements the narrative chapters with the concrete symbols that own each stage.

> **Reference page:** This chapter is for programmers, auditors, or readers tracing a bug. If you only want to understand or safely use the product, read [00 — Beginner guide](00-beginner-guide.md) and [10 — User FAQ](10-user-faq.md) instead.

## 1. Startup and process construction

| Symbol | Location | What it owns |
|---|---|---|
| `main()` | [`src/main.tsx`](../../../src/main.tsx) | Defines CLI options, validates combinations, loads settings/auth/models, resolves resume/worktree/remote state, creates tools and commands, starts MCP/hooks, then selects interactive or headless execution. |
| `init()` | [`src/entrypoints/init.ts`](../../../src/entrypoints/init.ts) | Memoized process initialization for configuration, telemetry, environment, caches, and runtime prerequisites. Memoization prevents duplicate initialization through multiple entry paths. |
| `setup()` | [`src/setup.ts`](../../../src/setup.ts) | Establishes cwd/project trust and setup-dependent state before code that assumes the workspace is ready. |
| `launchRepl()` | [`src/replLauncher.tsx`](../../../src/replLauncher.tsx) | Mounts the React/Ink application with providers and REPL props. It is the boundary between startup orchestration and interactive UI. |
| `getSystemContext()` | [`src/context.ts`](../../../src/context.ts) | Memoized machine/project snapshot, including Git-oriented context used in the prompt. |
| `getUserContext()` | [`src/context.ts`](../../../src/context.ts) | Memoized user/project instruction discovery, including `CLAUDE.md`-style content and date context. |

### Startup ordering detail

[`src/main.tsx`](../../../src/main.tsx) begins managed-device and keychain reads before its full import graph finishes evaluating. Later startup waits for those prefetched results only when needed. This is why moving apparently ordinary top-level imports can change startup performance.

## 2. Conversation ownership

### `QueryEngine`

[`QueryEngine`](../../../src/QueryEngine.ts) represents one conversation, not one model request. Its long-lived fields include:

| Field | Meaning |
|---|---|
| `mutableMessages` | Full current internal transcript for subsequent turns |
| `abortController` | Conversation/turn cancellation root |
| `permissionDenials` | Denials reported in the SDK result |
| `totalUsage` | Accumulated token/API usage |
| `readFileState` | File content/version cache shared across turns |
| `discoveredSkillNames` | Turn-scoped skill-discovery accounting |
| `loadedNestedMemoryPaths` | Memory files already incorporated |

`submitMessage()` is an async generator. It can yield partial SDK events while retaining the state required for the next user turn.

### `submitMessage()` stages

1. set the requested cwd;
2. clear turn-scoped skill discovery;
3. wrap `canUseTool` so denials are recorded;
4. process user input and command/skill expansion;
5. build system prompt, contexts, and `ToolUseContext`;
6. append/record the user message;
7. delegate the agentic turn to [`query()`](../../../src/query.ts);
8. translate internal stream/messages into SDK messages;
9. update usage, file history, transcript, and final result;
10. flush persistence where required.

## 3. User-input processing

| Symbol | Location | Responsibility |
|---|---|---|
| `handlePromptSubmit()` | [`src/utils/handlePromptSubmit.ts`](../../../src/utils/handlePromptSubmit.ts) | Interactive front-door: expand paste/image references, parse slash command, execute immediate commands, or enqueue model input. |
| `processUserInput()` | [`src/utils/processUserInput/processUserInput.ts`](../../../src/utils/processUserInput/processUserInput.ts) | Shared semantic processing: slash-command expansion, hooks, memory/attachments, local-command output, and final user-message construction. |
| `buildSystemInitMessage()` | [`src/utils/messages/systemInit.ts`](../../../src/utils/messages/systemInit.ts) | Produces SDK initialization metadata: session, cwd, model, tools, MCP servers, permission mode, commands, API-key source, and output style. |

Interactive submission and semantic processing are separate because headless/SDK callers do not use the terminal input component but must still receive the same command, hook, context, and message behavior.

## 4. Agentic turn state machine

| Symbol | Location | Responsibility |
|---|---|---|
| `query()` | [`src/query.ts`](../../../src/query.ts) | Public async-generator wrapper. Delegates to the loop and marks consumed queued commands complete only on normal return. |
| `queryLoop()` | [`src/query.ts`](../../../src/query.ts) | Private iterative state machine for context preparation, model streaming, tool execution, retries, stop hooks, and terminal reason. |
| `buildQueryConfig()` | [`src/query/config.ts`](../../../src/query/config.ts) | Captures feature/config choices used by one query, reducing repeated global lookups and making behavior testable. |
| `handleStopHooks()` | [`src/query/stopHooks.ts`](../../../src/query/stopHooks.ts) | Runs final-answer stop hooks and decides complete, prevent, or append blocking feedback and continue. |
| `checkTokenBudget()` | [`src/query/tokenBudget.ts`](../../../src/query/tokenBudget.ts) | Decides whether to complete or inject a hidden continuation nudge based on turn output and diminishing returns. |

### Loop state

At the top of each iteration, `queryLoop()` reads one `State` object:

- messages and current tool context;
- compaction tracking;
- max-output recovery count;
- reactive-compaction guard;
- output-token override;
- pending tool summary;
- stop-hook activity;
- turn count;
- transition reason.

Every retry/continue branch writes the complete next state. This makes it visible which guards reset and which survive.

## 5. Model request and stream

| Area | Main location | Function |
|---|---|---|
| Request construction and streaming | [`src/services/api/claude.ts`](../../../src/services/api/claude.ts) | Converts internal prompt/messages/tools into provider request parameters, applies model/thinking/task-budget options, and yields normalized stream messages. |
| Retry | [`src/services/api/withRetry.ts`](../../../src/services/api/withRetry.ts) | Applies source-aware backoff, rate-limit handling, fallback signals, and unattended heartbeat behavior. |
| Stream watchdog | [`src/services/api/streamWatchdog.ts`](../../../src/services/api/streamWatchdog.ts) | Detects first-event, first-content, idle, and maximum-duration stalls while tracking whether effects have begun. |
| Commit buffer | [`src/services/api/streamAssistantCommitBuffer.ts`](../../../src/services/api/streamAssistantCommitBuffer.ts) | Holds retry-safe assistant blocks so a failed attempt does not leak partial duplicate content. |
| Error classification | [`src/services/api/errors.ts`](../../../src/services/api/errors.ts) | Recognizes API, rate-limit, context-length, and prompt-too-long errors for user messages and recovery branches. |

### Why streaming output is accumulated as well as yielded

The UI/SDK needs deltas immediately, but tool execution and recovery need complete assistant messages. The loop therefore performs two operations at once:

```text
stream event -> yield to consumer
             -> add to current assistant-message assembly
```

Recoverable error messages are added internally but withheld from consumers until recovery fails.

## 6. Tool-pool construction

| Symbol | Location | Responsibility |
|---|---|---|
| `getAllBaseTools()` | [`src/tools.ts`](../../../src/tools.ts) | Exhaustive built-in registration branches for the current build/environment. |
| `filterToolsByDenyRules()` | [`src/tools.ts`](../../../src/tools.ts) | Removes blanket-denied built-ins or MCP server/tool entries before prompt exposure. |
| `getTools()` | [`src/tools.ts`](../../../src/tools.ts) | Applies simple/REPL mode, special-helper exclusion, deny rules, and `isEnabled()`. |
| `assembleToolPool()` | [`src/tools.ts`](../../../src/tools.ts) | Combines built-ins and MCP tools, preserves a stable built-in prefix, sorts partitions, and deduplicates by name. |
| `findToolByName()` | [`src/Tool.ts`](../../../src/Tool.ts) | Resolves primary names and compatibility aliases. |

See [08 — Complete tool catalog](08-complete-tool-catalog.md) for every registered entry and gate.

## 7. Tool execution

| Symbol | Location | Responsibility |
|---|---|---|
| `runTools()` | [`src/services/tools/toolOrchestration.ts`](../../../src/services/tools/toolOrchestration.ts) | Partitions calls into safe concurrent batches and exclusive serial batches, then yields messages/context updates in model order. |
| `partitionToolCalls()` | [`src/services/tools/toolOrchestration.ts`](../../../src/services/tools/toolOrchestration.ts) | Parses each input and calls `isConcurrencySafe()`. Parse/check failures become serial execution. |
| `runToolsSerially()` | [`src/services/tools/toolOrchestration.ts`](../../../src/services/tools/toolOrchestration.ts) | Runs calls one at a time and applies each returned context modifier before the next call. |
| `runToolsConcurrently()` | [`src/services/tools/toolOrchestration.ts`](../../../src/services/tools/toolOrchestration.ts) | Runs safe calls under the configured concurrency limit. |
| `runToolUse()` | [`src/services/tools/toolExecution.ts`](../../../src/services/tools/toolExecution.ts) | Owns one call’s schema parsing, validation, hooks, permission, execution, progress, error conversion, result budgeting, and completion state. |
| `StreamingToolExecutor` | [`src/services/tools/StreamingToolExecutor.ts`](../../../src/services/tools/StreamingToolExecutor.ts) | Starts eligible calls before the assistant stream ends and discards/bubbles results correctly on fallback or abort. |
| `applyToolResultBudget()` | [`src/utils/toolResultStorage.ts`](../../../src/utils/toolResultStorage.ts) | Persists output over a tool-specific threshold and replaces it with a bounded preview/reference. |

### One-call failure categories

`runToolUse()` has to distinguish:

- unknown tool or malformed schema;
- tool-specific validation failure;
- hook block;
- permission denial;
- user cancellation;
- execution exception;
- sibling cancellation;
- oversized result persistence;
- successful result with a context modifier.

These cases cannot all be represented as thrown exceptions because many must become a valid `tool_result` sent back to the model.

## 8. Permission decision ownership

| Area | Location | Responsibility |
|---|---|---|
| Types and modes | [`src/types/permissions.ts`](../../../src/types/permissions.ts) | Defines rule sources, modes, allow/ask/deny/passthrough decisions, classifier state, and updated input. |
| General evaluation | [`src/utils/permissions/permissions.ts`](../../../src/utils/permissions/permissions.ts) | Applies blanket/tool-content rules, policy, mode transformations, and sensitive checks. |
| Initial context | [`src/utils/permissions/permissionSetup.ts`](../../../src/utils/permissions/permissionSetup.ts) | Parses CLI/settings modes and rules, checks bypass availability, strips dangerous rules, and creates the session context. |
| Filesystem scope | [`src/utils/permissions/filesystem.ts`](../../../src/utils/permissions/filesystem.ts) | Normalizes paths and enforces cwd/additional-directory/settings/sensitive-location boundaries. |
| Bash-specific checks | [`src/tools/BashTool/bashPermissions.ts`](../../../src/tools/BashTool/bashPermissions.ts) | Parses command structure, matches command rules, handles wrappers, and decides safe/ask/deny behavior. |
| Interactive resolution | [`src/hooks/toolPermission/PermissionContext.ts`](../../../src/hooks/toolPermission/PermissionContext.ts) | Coordinates UI choice and asynchronous classifier through a resolve-once promise. |

The central evaluator calls `Tool.checkPermissions()` only after schema and tool validation. A tool-specific allow does not necessarily bypass later global sensitive-path or classifier behavior.

## 9. Transcript and resume

| Symbol | Location | Responsibility |
|---|---|---|
| `recordTranscript()` | [`src/utils/sessionStorage.ts`](../../../src/utils/sessionStorage.ts) | Serializes a typed entry with session/cwd/version/Git metadata and queues/appends it to the JSONL transcript. |
| `flushSessionStorage()` | [`src/utils/sessionStorage.ts`](../../../src/utils/sessionStorage.ts) | Waits for queued transcript writes so process exit or SDK completion does not lose entries. |
| `readTranscriptForLoad()` | [`src/utils/sessionStoragePortable.ts`](../../../src/utils/sessionStoragePortable.ts) | Loads and repairs the transcript view used for resume, including compact-boundary and tombstone handling. |
| `restoreSessionStateFromLog()` | [`src/utils/sessionRestore.ts`](../../../src/utils/sessionRestore.ts) | Reconstructs messages, todos, file/attribution history, context-collapse state, and other runtime resume data. |
| `loadConversationForResume()` | [`src/utils/conversationRecovery.ts`](../../../src/utils/conversationRecovery.ts) | Coordinates selecting/loading a prior conversation and producing the processed resume payload expected by startup. |
| `restoreCostStateForSession()` | [`src/cost-tracker.ts`](../../../src/cost-tracker.ts) | Restores saved cost/timing/model usage only when the project’s saved session ID matches. |

### Durability model

The JSONL transcript is authoritative. The SQLite local index is a derived acceleration structure. A damaged or stale index can be rebuilt from transcripts; treating the index as the only record would lose metadata and recovery semantics.

## 10. Core data crossing boundaries

| Boundary | Internal form | External form |
|---|---|---|
| Model API | Internal `Message[]`, `Tool[]`, contexts | Anthropic/provider request and stream events |
| Tool call | `ToolUseBlock` + `ToolUseContext` | `ToolResultBlockParam` returned to model |
| SDK | Internal messages/events | `SDKMessage` union |
| Transcript | Runtime messages and metadata | One typed JSON object per line |
| Desktop | CLI/server session events | WebSocket messages and permission requests |
| MCP | Internal dynamic `Tool` wrappers | MCP transport calls/results/resources |

Most “extra” mapper code exists to preserve semantics at these boundaries: names, IDs, thinking signatures, partial messages, tool-result pairing, metadata, and cancellation all have to survive conversion.

## 11. Fast debugging map

When behavior is wrong, start here:

| Symptom | First source to inspect |
|---|---|
| CLI option/startup problem | [`src/main.tsx`](../../../src/main.tsx) |
| Prompt interpreted incorrectly | [`processUserInput.ts`](../../../src/utils/processUserInput/processUserInput.ts) |
| Tool missing from model | [`src/tools.ts`](../../../src/tools.ts), then [`src/utils/toolPool.ts`](../../../src/utils/toolPool.ts) |
| Tool requested but rejected | [`toolExecution.ts`](../../../src/services/tools/toolExecution.ts), then permission implementation |
| Shell command unexpectedly asks/denies | [`bashPermissions.ts`](../../../src/tools/BashTool/bashPermissions.ts) |
| Repeated or orphaned stream output | [`src/query.ts`](../../../src/query.ts) and stream commit/watchdog modules |
| Context limit loop | compaction services under [`src/services/compact/`](../../../src/services/compact/) and recovery state in [`src/query.ts`](../../../src/query.ts) |
| Resume lost state | [`sessionStorage.ts`](../../../src/utils/sessionStorage.ts) and [`sessionRestore.ts`](../../../src/utils/sessionRestore.ts) |
| Desktop permission hangs | [`conversationService.ts`](../../../src/server/services/conversationService.ts), WebSocket handler, and permission context |
