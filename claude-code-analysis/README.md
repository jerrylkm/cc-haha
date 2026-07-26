# Claude Code CLI Architecture & Source Code Analysis

Welcome to the comprehensive, plain-English technical breakdown of the **Claude Code CLI** source code.

This documentation suite explains how Claude Code CLI is engineered under the hood—from CLI invocation and user input processing to LLM streaming, tool execution loops, context compaction, React/Ink terminal rendering, and state persistence.

---

## 📚 Documentation Index

1. **[01-architecture-overview.md](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/claude-code-analysis/01-architecture-overview.md)**  
   *High-Level System Architecture, Entry Points, and Lifecycle.*  
   Explains how the CLI boots up, parses flags, runs pre-flight checks, loads configurations, and launches the interactive REPL.

2. **[02-agent-loop-and-query-engine.md](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/claude-code-analysis/02-agent-loop-and-query-engine.md)**  
   *The Agent Loop, QueryEngine, and Streaming Interaction.*  
   Deep dive into `QueryEngine` and `query()`, detailing how user prompts are processed, streamed through Anthropic LLMs, converted into tool call dispatches, and handled recursively until completion. Includes auto-compaction and token management.

3. **[03-tool-system.md](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/claude-code-analysis/03-tool-system.md)**  
   *Tool Definitions, Registry, Permissions, and Execution.*  
   How tools (`Bash`, `Edit`, `View`, `Glob`, `Grep`, `Task`, `MCP`) are structured via the `Tool` schema, how permissions are checked before execution, and how tool outputs feed back into the model context.

4. **[04-terminal-ui-and-interaction.md](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/claude-code-analysis/04-terminal-ui-and-interaction.md)**  
   *React + Ink Terminal UI, REPL, and Interactive Screens.*  
   Explains how Ink renders React components directly into terminal ANSI escapes, showing streaming response text, spinners, diffs, user confirmation popups, and status updates.

5. **[05-context-memory-and-persistence.md](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/claude-code-analysis/05-context-memory-and-persistence.md)**  
   *Context Assembly, System Prompts, History, and Memory (`memdir`).*  
   How project context (Git status, `CLAUDE.md` instructions, environment specs) is gathered into system prompts, how chat transcripts are stored, and how memory directories persist across sessions.

6. **[06-sub-agents-tasks-and-extensions.md](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/claude-code-analysis/06-sub-agents-tasks-and-extensions.md)**  
   *Sub-Agents, Swarms, Tasks, Skills, and Local Services.*  
   Overview of sub-agent task runners, goals coordination, skills plugins system, and local API / WebSocket server bridge.

---

## 🧩 Core Architecture Diagram

```
+-----------------------------------------------------------------------+
|                            USER TERMINAL                              |
+-----------------------------------------------------------------------+
                                   |
                                   v
+-----------------------------------------------------------------------+
|  1. ENTRY POINT & REPL LAUNCHER ([src/main.tsx](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/main.tsx:885))      |
|     - Pre-flight checks (keychain, MDM, settings)                    |
|     - Bootstraps React + Ink TUI Root ([src/ink.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/ink.ts:17))                |
+-----------------------------------------------------------------------+
                                   |
                                   v
+-----------------------------------------------------------------------+
|  2. QUERY ENGINE / AGENT LOOP ([src/QueryEngine.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/QueryEngine.ts:187))     |
|     - Assembles System Prompt ([src/context.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/context.ts:116))               |
|     - Manages conversation state & history                            |
|     - Invokes query() stream generator ([src/query.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/query.ts:222))              |
+-----------------------------------------------------------------------+
           |                                             ^
           | (Stream prompt & messages)                  | (Stream response blocks)
           v                                             |
+-------------------+                           +-----------------------+
|  ANTHROPIC API    |                           | 3. TOOL EXECUTION     |
|  (Claude Models)  |--- (Tool Call Requests) ->|    ENGINE             |
+-------------------+                           |    ([src/tools.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/tools.ts:194))|
                                                +-----------------------+
                                                 | Bash, View, Edit,     |
                                                 | Glob, Grep, MCP, etc. |
                                                 +-----------------------+
```

---

## 📁 Key File Map

| System Area | File / Directory | Description |
| :--- | :--- | :--- |
| **CLI Entry Point** | [src/main.tsx](../src/main.tsx) | Main CLI runner, Commander.js argument parsing, REPL initialization. |
| **Init & Setup** | [src/entrypoints/init.ts](../src/entrypoints/init.ts) | Pre-flight settings, telemetry initialization, keychain prefetching. |
| **Agent Controller** | [src/QueryEngine.ts](../src/QueryEngine.ts) | Orchestrates conversation turns, tool responses, app state, and interruptions. |
| **Query Stream** | [src/query.ts](../src/query.ts) | Async generator handling LLM API requests, event streaming, tool loops. |
| **Tool Interface** | [src/Tool.ts](../src/Tool.ts) | Core schema, types, and builder utilities for all tools. |
| **Tool Registry** | [src/tools.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/tools.ts) | Tool aggregator, permission filtering, and base tool loader. |
| **Built-in Tools** | [src/tools/](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/tools/) | Individual implementations for `BashTool`, `FileEditTool`, `FileReadTool`, `GlobTool`, etc. |
| **Terminal UI** | [src/ink.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/ink.ts) | Ink React rendering wrapper with custom design system themes. |
| **UI Components** | [src/components/](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/components/) | React Ink components rendering chat transcript, tools output, diffs, inputs. |
| **Context Assembly** | [src/context.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/context.ts) | Gathers Git status, OS context, user rules (`CLAUDE.md`), and directory tree. |
| **History & Storage**| [src/history.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/history.ts) | Chat transcript reading, prompt history storage, image/text reference mapping. |
| **Sub-Agents & Tasks**| [src/Task.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/Task.ts) | Task handle state, background task spawning, sub-agent execution context. |
| **Skills Plugin System**| [src/skills/](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/skills/) | Skill loading, custom instructions injection, MCP skills integration. |
