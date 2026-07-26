# 01 — Orientation

## What should a non-programmer learn from this page?

This repository is not one small command-line program. It contains the AI engine, terminal interface, local server, desktop application, messaging integrations, tests, and documentation. Different pieces may have been added at different times, so a file’s presence does not prove it came from the claimed source exposure.

If the names in “central objects” become too technical, remember only:

- `QueryEngine` remembers one conversation;
- `query()` carries out one request from you;
- a `Tool` is an action the AI can ask the program to perform;
- messages are the records passed among you, the program, tools, and the AI provider.

## 1. What is in the repository?

The repository combines several applications:

| Area | Runtime | Responsibility |
|---|---|---|
| [`src/entrypoints/`](../../../src/entrypoints/) and [`src/main.tsx`](../../../src/main.tsx) | Bun | CLI startup, flags, interactive/headless selection |
| [`src/screens/`](../../../src/screens/), [`src/components/`](../../../src/components/), [`src/ink/`](../../../src/ink/) | Terminal | REPL and custom React/Ink terminal rendering |
| [`src/query.ts`](../../../src/query.ts), [`src/QueryEngine.ts`](../../../src/QueryEngine.ts) | Bun | Conversation and agent loop |
| [`src/tools/`](../../../src/tools/), [`src/Tool.ts`](../../../src/Tool.ts) | Bun | Tool contracts and implementations |
| [`src/services/`](../../../src/services/) | Bun | Model API, MCP, LSP, compaction, analytics, plugins |
| [`src/server/`](../../../src/server/) | Bun | Local HTTP/WebSocket API used by desktop and remote clients |
| [`desktop/`](../../../desktop/) | Electron + Chromium | Graphical client and native host |
| [`adapters/`](../../../adapters/) | Bun | Telegram, Feishu, WeChat, DingTalk, and WhatsApp bridges |
| [`runtime/`](../../../runtime/) | Python | Computer Use platform helpers |

This matters because “Claude Code source” and “everything in this repository” are not equivalent. The current tree includes recovery fixes and product additions made after the claimed source exposure.

## 2. The central objects

### `QueryEngine`

[`QueryEngine`](../../../src/QueryEngine.ts) owns one conversation in the headless/SDK path. It retains messages, file state, usage, permission denials, discovered skills, and an abort controller across calls to `submitMessage()`.

Think of it as the session manager:

```text
one QueryEngine
  ├─ turn 1: submitMessage("inspect the bug")
  ├─ turn 2: submitMessage("fix it")
  └─ turn 3: submitMessage("run tests")
```

### `query()`

[`query()`](../../../src/query.ts) is the state machine for one agentic turn. A turn can contain many model requests because each tool result is sent back to the model.

### `Tool`

[`Tool`](../../../src/Tool.ts) is a large structural contract, not just a function. Each tool describes:

- its name, aliases, prompt, and input/output schemas;
- validation and permission behavior;
- whether it is read-only, destructive, open-world, deferred, or safe to run concurrently;
- execution and cancellation behavior;
- model-facing result conversion;
- terminal rendering, transcript search text, progress, and compact summaries.

The breadth of this contract explains why a tool can behave consistently in the model loop, permissions UI, SDK stream, transcript, and terminal.

### `ToolUseContext`

Also defined in [`src/Tool.ts`](../../../src/Tool.ts), this is the runtime environment handed to tools. It carries configuration, abort signals, agent identity, file caches, permission state, application-state accessors, MCP handling, and callbacks.

In everyday terms, it is the “folder of information and controls” given to a tool so it knows where it is running, what it may do, how to stop, and how to report progress.

### Messages

The discriminated message types live in [`src/types/message.ts`](../../../src/types/message.ts). Important categories are:

- `user`: human input, meta instructions, and tool results;
- `assistant`: model text, thinking, tool requests, and API errors;
- `system`: runtime status or synthetic guidance;
- `attachment`: metadata that belongs in the session but is not a normal API message;
- `tombstone`: remove an orphaned/failed streamed message;
- `tool_use_summary`: compact display of tool activity.

Not every internal message is visible as a chat bubble. Some are bookkeeping needed for recovery, tool pairing, compacting history, or updating the interface.

## 3. Static structure versus runtime structure

A directory tree alone can be misleading. At runtime, the important dependency direction is:

```text
entrypoint
  -> initialization/configuration
  -> command or UI front end
  -> QueryEngine / query
  -> model API and context services
  -> tool orchestration
  -> permissions/hooks/sandbox
  -> concrete tool
  -> transcript, cost, analytics, UI updates
```

Circular dependencies are avoided in several places with lazy `require()` calls. Examples include team tools in [`src/tools.ts`](../../../src/tools.ts), optional feature modules in [`src/query.ts`](../../../src/query.ts), and teammate/coordinator features in [`src/main.tsx`](../../../src/main.tsx). These are deliberate load-order and dead-code-elimination choices, not ordinary style.

## 4. Feature-gated code

The source uses `feature()` from `bun:bundle`. A feature gate may completely remove a module from a build, so conditional `require()` is used instead of a static import. Major gates visible in the core path include:

- reactive compaction and context collapse;
- history snipping and cached micro-compaction;
- coordinator, team, assistant, and background-session modes;
- token budgets and streaming tool execution;
- experimental skill search and templates;
- internal-only diagnostics and cache-breaking tools.

Do not assume all checked-in branches execute in every distributed build.

**When tracing code, watch for `feature()`.** A `feature('SOME_GATE')` call — usually paired with a conditional `require()` rather than a static `import` — marks a branch that a build can include, exclude, or alter. If a symbol seems to have no caller, or a module appears never to load, check whether it sits behind a feature gate before concluding it is dead code. The same pattern also explains why the static import graph does not fully predict runtime behavior.

## 5. Recovered-source fixes

The repository documents a small set of initial repair points in [`docs/en/reference/fixes.md`](../reference/fixes.md):

- route no-argument startup to the full CLI instead of recovery mode;
- supply missing skill and prompt resources;
- supply a missing persistence type module;
- tolerate a missing native modifier-key package so Enter still submits;
- stop forcing a recovery environment variable that skipped setup.

Those fixes show that the exposed material was incomplete and could not be treated as a directly runnable release.

## 6. How to navigate the analysis

For behavior, read the numbered chapters. For a specific file, search [the source atlas](07-source-atlas.md). For exact implementation, follow the source link from either place. Tests are included in the atlas because they often state edge-case intent more clearly than production code.
