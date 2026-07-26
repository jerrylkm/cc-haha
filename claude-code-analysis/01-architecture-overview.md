# 01. Architecture Overview & Lifecycle

This document provides a simple, step-by-step explanation of how the **Claude Code CLI** starts up, initializes its subsystems, parses user commands, and enters the interactive agent loop.

---

## 🚀 Startup Lifecycle Step-by-Step

When you run `claude` in your terminal, the application undergoes a multi-phase initialization pipeline:

```
[Terminal Execution: `claude`]
          │
          ▼
┌──────────────────────────────────────────────────────────┐
│ Phase 1: High-Priority Side-Effects & Performance Checks │
│ Location: [src/main.tsx](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/main.tsx:1)                         │
│ - Startup profiler (`profileCheckpoint`)                 │
│ - Parallel raw MDM read                                 │
│ - macOS Keychain prefetch for API keys & OAuth          │
└──────────────────────────────────────────────────────────┘
          │
          ▼
┌──────────────────────────────────────────────────────────┐
│ Phase 2: Commander Command Line Parsing                  │
│ Location: [src/main.tsx](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/main.tsx:885)                       │
│ - Parse flags (`--print`, `--settings`, `--compact`)      │
│ - Register slash commands & subcommands                 │
└──────────────────────────────────────────────────────────┘
          │
          ▼
┌──────────────────────────────────────────────────────────┐
│ Phase 3: Subsystem Initialization                        │
│ Location: [src/entrypoints/init.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/entrypoints/init.ts:57)          │
│ - Load configuration & user settings                    │
│ - Setup telemetry & GrowthBook feature flags            │
│ - Load remote policy limits & managed settings           │
└──────────────────────────────────────────────────────────┘
          │
          ▼
┌──────────────────────────────────────────────────────────┐
│ Phase 4: Ink UI Root Creation & REPL Launch              │
│ Location: [src/replLauncher.tsx](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/replLauncher.tsx:12)            │
│ - Create React-Ink terminal rendering root               │
│ - Mount main `<App />` container                          │
│ - Transfer control to `QueryEngine`                      │
└──────────────────────────────────────────────────────────┘
```

---

## 🔍 Detailed Phase Breakdown

### 1. High-Priority Performance Initialization
Before evaluating large NPM modules, [src/main.tsx](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/main.tsx:1) runs critical async prefetch steps in parallel:
- **Keychain Prefetch**: [src/utils/secureStorage/keychainPrefetch.js](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/utils/secureStorage/keychainPrefetch.js) pre-loads OAuth credentials and API keys from macOS Keychain / OS secure store so sync reads won't block startup (~65ms saved).
- **MDM Read**: [src/utils/settings/mdm/rawRead.js](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/utils/settings/mdm/rawRead.js) reads Mobile Device Management policies in parallel.
- **Profiler Checkpoints**: [src/utils/startupProfiler.js](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/utils/startupProfiler.js) records startup latency.

### 2. Command Line Option Parsing
The CLI uses **Commander.js** inside `run()` at [src/main.tsx](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/main.tsx:885):
- Configures options like `-p` / `--print` (non-interactive mode), `--settings`, `--init-only`, `--model`, `--dangerously-skip-permissions`.
- Determines whether the session is interactive (terminal REPL) or batch execution (`--print`).

### 3. Subsystem Initialization (`init()`)
The `init()` function in [src/entrypoints/init.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/entrypoints/init.ts:57):
- Loads user settings from `~/.claude/settings.json`.
- Merges environment variables safely.
- Initializes telemetry and feature flags (GrowthBook).
- Checks policy limits and remote managed enterprise settings.

### 4. Terminal REPL & React-Ink Root
If running interactively:
- `createRoot()` in [src/ink.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/ink.ts:17) sets up Ink's terminal rendering engine wrapped in `ThemeProvider`.
- `launchRepl()` in [src/replLauncher.tsx](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/replLauncher.tsx:12) mounts the top-level `<App />` component.
- The user is presented with the prompt input box, and input is dispatched to [src/QueryEngine.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/QueryEngine.ts:187).

---

## 🗂️ Module Architecture Map

The repository is modularly structured across specialized directories:

```
src/
├── entrypoints/      # Bootstrapping, CLI CLI entry definitions & Agent SDK types
├── main.tsx          # Main entry file containing Commander CLI definition
├── replLauncher.tsx  # Mounts the Ink React UI and launches the REPL
├── QueryEngine.ts    # Class controlling the agent loop and prompt sessions
├── query.ts          # Core streaming interaction generator with Claude API
├── Tool.ts           # Base Tool type definitions and validation schemas
├── tools.ts          # Aggregates built-in tools and applies permission rules
├── tools/            # Concrete tool implementations (Bash, Edit, View, Grep, etc.)
├── components/       # Terminal UI components built with React + Ink
├── ink.ts            # Custom React-Ink terminal renderer configuration
├── context.ts        # System context assembly (Git status, OS, CLAUDE.md)
├── history.ts        # Chat transcript reading/writing & prompt history
├── memdir/           # Memory directories & long-term agent memory
├── state/            # React & CLI application state management
├── Task.ts           # Background task and sub-agent task handles
├── skills/           # Bundled & user-defined Skills extension engine
└── services/         # API clients, MCP registry, telemetry, OAuth, voice
```

---

## 🔑 Key Takeaways

1. **Lazy & Parallel Bootstrapping**: Time-consuming OS tasks (Keychain, MDM, Settings) are prefetched before loading heavy UI libraries.
2. **Commander + Ink Architecture**: Commander handles CLI arguments while Ink renders React UI components to ANSI terminal output.
3. **Decoupled Execution**: The UI (`replLauncher.tsx`) communicates with the Agent Engine (`QueryEngine.ts`) via event callbacks and React state updates.
