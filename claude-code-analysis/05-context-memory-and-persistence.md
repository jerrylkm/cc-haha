# 05. Context, Memory & Persistence

For Claude to be an effective coding assistant, it must understand project conventions, track historical decisions, remember user preferences, and preserve session state across terminal restarts.

---

## 🧠 System Context Assembly

Before every prompt turn, [src/context.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/context.ts) compiles system and user context into a structured prompt injection via `getSystemContext()` and `getUserContext()`:

```
┌─────────────────────────────────────────────────────────────┐
│                    System Context Assembly                  │
│ Location: [src/context.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/context.ts:116)                           │
└─────────────────────────────────────────────────────────────┘
                               │
       ┌───────────────────────┼───────────────────────┐
       ▼                       ▼                       ▼
┌──────────────┐        ┌──────────────┐        ┌──────────────┐
│  Git Status  │        │ Project Rules│        │ Memory Directory│
│  `git status`│        │ `CLAUDE.md`  │        │  `memdir`    │
└──────────────┘        └──────────────┘        └──────────────┘
       │                       │                       │
       └───────────────────────┼───────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                   Injected System Prompt                    │
└─────────────────────────────────────────────────────────────┘
```

### Context Components:
1. **Repository Conventions (`CLAUDE.md` / `AGENTS.md`)**: Custom user guidelines placed in repository roots or parent directories are automatically detected, parsed, and prepended as hard system instructions.
2. **Git Status & Working Tree**: Evaluates `git status --short`, current branch name, and untracked changes so Claude is immediately aware of modified files.
3. **Environment & Workspace Information**: Injects current working directory (`cwd`), operating system (macOS/Linux/Windows), shell environment, and available tools.
4. **Memory Directory (`memdir`)**: Inspects `.claude/memory/` for long-term project facts saved in past sessions.

---

## 💾 Session Persistence & Transcripts

History and transcript storage are handled in [src/history.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/history.ts):

1. **Prompt History**:
   User input lines are appended to local history files using `addToHistory()`, allowing users to navigate previous prompts with the `Up`/`Down` arrow keys.

2. **Session Transcripts**:
   Each conversation session generates a JSON transcript stored in `~/.claude/transcripts/`.
   - Allows sessions to be resumed using `claude --resume` or listed using `claude --list`.
   - Enables crash recovery and session audit logging.

3. **Pasted Text & Image References**:
   Large pasted text snippets or pasted images are stored in temporary reference files rather than polluting the prompt buffer, referenced via ID markers like `[Pasted Text #1 (42 lines)]`.

---

## ⚙️ Configuration & App State

Configuration is managed hierarchically across several layers:

```
Global Defaults ──> Enterprise Policy ──> ~/.claude/settings.json ──> Project .claude/settings.json ──> Env Vars / CLI Flags
```

- **Global Config Loader**: [src/utils/config.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/utils/config.ts) reads and validates settings schemas.
- **Application State**: `AppState` ([src/state/AppState.js](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/state/AppState.js)) holds runtime state (active tools, current model, pending permission requests, cost counter, active theme).
- **User Settings Safety**: As specified in system guidelines, `~/.claude/settings.json` is treated as user-owned shared state: unknown fields are preserved, additions are merged additively, and schema markers are never forcefully overwritten.

---

## 🔑 Key Code Locations

- **System Context Assembly**: [src/context.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/context.ts:116)
- **History & Transcript Manager**: [src/history.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/history.ts:190)
- **Memory Directory Engine**: [src/memdir/memdir.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/memdir/memdir.ts)
- **Configuration Reader**: [src/utils/config.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/utils/config.ts)
- **Session Bootstrap State**: [src/bootstrap/state.js](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/bootstrap/state.js)
