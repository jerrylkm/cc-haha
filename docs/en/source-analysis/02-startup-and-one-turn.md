# 02 — Startup and one turn

## In everyday language

When you press Enter, your text is not sent straight to the AI with no preparation. The program first loads rules and context, decides which capabilities are available, sends a structured request, checks every proposed action, and may repeat this process several times.

What you may see:

- status text while the model is responding;
- file reads or searches that happen quickly;
- an approval dialog for a command or edit;
- several tool activities for one request;
- a final answer only after tool results have been examined;
- a saved session that can later be resumed.

## 1. Process startup

The shell entry is [`bin/claude-haha`](../../../bin/claude-haha). The Bun preload in [`preload.ts`](../../../preload.ts) supplies compatibility behavior before the main bundle loads. The CLI entrypoint is [`src/entrypoints/cli.tsx`](../../../src/entrypoints/cli.tsx), which reaches the main command construction in [`src/main.tsx`](../../../src/main.tsx).

At the very top of [`src/main.tsx`](../../../src/main.tsx), startup work is intentionally launched before the rest of the imports finish:

1. a startup profiler records the earliest checkpoint;
2. managed-device settings reads begin;
3. macOS keychain reads begin;
4. normal imports continue while those operations run.

This hides otherwise serial startup latency. It also means import order is behaviorally significant.

## 2. Initialization

The startup path combines:

- settings and managed policy;
- environment-variable application;
- authentication and subscription state;
- model resolution and capability metadata;
- migrations;
- plugin and bundled-skill initialization;
- MCP configuration and policy filtering;
- sandbox, LSP, telemetry, and analytics setup;
- session/worktree/resume handling;
- trust and permission-mode setup.

The relevant orchestration is spread across [`src/main.tsx`](../../../src/main.tsx), [`src/setup.ts`](../../../src/setup.ts), and [`src/entrypoints/init.ts`](../../../src/entrypoints/init.ts). It is large because the same executable supports interactive REPL, print/headless SDK use, remote sessions, resume, teleport, assistant/team modes, and local server integration.

## 3. Interactive versus headless

### Interactive

[`launchRepl()`](../../../src/replLauncher.tsx) mounts the terminal application. [`src/screens/REPL.tsx`](../../../src/screens/REPL.tsx) owns the main interactive state and prompt queue.

### Headless/SDK

The headless route constructs one [`QueryEngine`](../../../src/QueryEngine.ts) per conversation. `submitMessage()` yields SDK messages as an async generator. The engine:

- sets the working directory;
- wraps permission checks to record denials for the SDK result;
- processes user input and slash-command expansion;
- builds prompt/context and tool state;
- invokes [`query()`](../../../src/query.ts);
- normalizes streamed internal messages into SDK events;
- records transcript and usage state.

## 4. A single ordinary user turn

Suppose the user enters “read the config and explain it.”

### Step A — Input becomes a queued command

[`handlePromptSubmit()`](../../../src/utils/handlePromptSubmit.ts) expands paste/image references and decides whether the input is an immediate local slash command or model input. Model input becomes a `QueuedCommand` from [`src/types/textInputTypes.ts`](../../../src/types/textInputTypes.ts).

**What this means to you:** text beginning with a supported slash command may control the local application instead of becoming an ordinary question to the model. Remote messages can have slash-command handling restricted for safety.

### Step B — User input is processed

[`processUserInput()`](../../../src/utils/processUserInput/processUserInput.ts) handles command expansion, hooks, local output, memory, attachments, and message creation. The resulting user message is appended to conversation state.

### Step C — Context is assembled

[`src/context.ts`](../../../src/context.ts) collects memoized system and user context. [`fetchSystemPromptParts()`](../../../src/utils/queryContext.ts) builds the system prompt from tool descriptions, agents, commands, environment details, project instructions, and configured additions.

Because this snapshot is memoized, some startup facts (such as Git status) are captured once and reused rather than refreshed every turn. See [05 — Context, persistence, and services](05-context-persistence-services.md) for why this can make later context intentionally stale.

**What this means to you:** the model receives more than the sentence you just typed. It may receive project instructions, relevant earlier conversation, available capability descriptions, and workspace information.

### Step D — The model request starts

[`query()`](../../../src/query.ts) prepares messages, applies result budgets and compaction mechanisms, then calls the model API through [`src/services/api/claude.ts`](../../../src/services/api/claude.ts).

### Step E — Output streams

Streaming events are yielded as they arrive. Text can render immediately. A `tool_use` block is accumulated until its JSON input is complete.

### Step F — The tool is approved and run

The tool framework finds the named tool, validates the schema and input, evaluates permission rules/hooks, optionally asks the user, and executes it. Read-only tools may run concurrently; mutating or uncertain tools are serialized.

**What this means to you:** an approval is for a proposed action, not for the model’s entire plan. Later actions may require separate checks—or may match a rule you already allowed.

### Step G — Tool result returns to the model

The result is converted to an Anthropic `tool_result` block. If it is too large, [`src/utils/toolResultStorage.ts`](../../../src/utils/toolResultStorage.ts) persists it and supplies a preview/path instead. The assistant tool request and user tool result are appended to the same turn.

### Step H — The model continues

The loop asks the model again with the new result. This repeats until no follow-up tool request remains.

### Step I — Completion

Stop hooks and token-budget logic run. Transcript/cost/session state is flushed, consumed command lifecycle IDs are marked complete, and the UI returns to an input-ready state.

“Complete” means the runtime finished the turn. It does not guarantee that the answer is correct, tests passed, or every external side effect can be undone.

## 5. Cancellation

Cancellation is layered:

- the conversation has an abort controller;
- tool execution may have a child controller;
- tool definitions choose `cancel` or `block` when new user input arrives;
- a permission denial can bubble from a tool controller to the turn controller;
- partial streamed output may be tombstoned when a retry/fallback would otherwise leave orphan content.

Blocking is the safe default for tools that must not be interrupted mid-write. Read/re-runnable operations can opt into cancellation.

## 6. Why one prompt can cause many API calls

“One turn” means one user intention, not one HTTP request:

```text
user message
  -> model requests Read
  -> Read result
  -> model requests Grep and Read
  -> both results
  -> model writes final explanation
```

All of these messages belong to one trajectory. Thinking blocks and tool/result pairing must remain valid throughout that trajectory, which is why [`src/query.ts`](../../../src/query.ts) contains careful message-preservation logic.
