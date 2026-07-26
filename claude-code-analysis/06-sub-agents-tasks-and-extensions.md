# 06. Sub-Agents, Tasks & Extensions

To handle complex, multi-step engineering workflows without overflowing the main conversation context window, **Claude Code CLI** incorporates a sub-agent task runner, a skills plugin framework, and a local API/WebSocket server bridge.

---

## 🤖 Sub-Agents & Task Runner

When Claude needs to perform parallel or deep exploration (such as researching code across dozens of files or running lengthy test suites), it spawns specialized **sub-agents** using the `Agent` / `Task` tool ([src/tools/AgentTool/AgentTool.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/tools/AgentTool/AgentTool.ts)).

```
┌───────────────────────────────────────────────────────────┐
│                     Main Agent Loop                       │
│ Location: [src/QueryEngine.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/QueryEngine.ts:187)                             │
└───────────────────────────────────────────────────────────┘
                              │
               (Spawns Sub-Agent via TaskTool)
                              ▼
┌───────────────────────────────────────────────────────────┐
│                    Sub-Agent Task Handle                  │
│ Location: [src/Task.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/Task.ts:72)                                   │
│ Types: `explore` | `task` | `general-purpose`             │
│        `code-review` | `research` | `security-review`      │
└───────────────────────────────────────────────────────────┘
                              │
               (Runs in isolated context window)
                              ▼
┌───────────────────────────────────────────────────────────┐
│                   Task Summary & Result                   │
│ - Summarizes findings back to Main Agent                 │
│ - Cleans up sub-agent context                             │
└───────────────────────────────────────────────────────────┘
```

### Specialized Sub-Agent Types:
- **`explore`**: Fast, lightweight agent for codebase search and file structure navigation.
- **`task`**: Command execution agent for builds, lints, and test suites (returns concise summaries on success, full logs on failure).
- **`general-purpose`**: Full-capability reasoning agent for complex multi-step refactoring.
- **`code-review`**: Read-only specialist for reviewing staged/unstaged git diffs.
- **`security-review`**: Read-only specialist focused exclusively on identifying high-confidence security vulnerabilities.

---

## 🎯 Goals & Coordinator Engine

For autonomous, multi-goal objective tracking:
- **Goals State**: [src/goals/goalState.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/goals/goalState.ts) tracks high-level objectives, sub-tasks, and completion conditions.
- **Coordinator**: [src/coordinator/](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/coordinator/) coordinates execution flow across multiple agent swarms when `CLAUDE_CODE_AGENT_SWARMS` is enabled.

---

## 🔌 Skills Plugin Framework

The **Skills** system in [src/skills/](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/skills/) enables domain-specific extensions:

1. **Bundled Skills**: Built-in skill packs loaded from [src/skills/bundledSkills.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/skills/bundledSkills.ts) (such as Claude API guides, verify scripts, and commit helpers).
2. **Directory Loader**: `loadSkillsDir()` ([src/skills/loadSkillsDir.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/skills/loadSkillsDir.ts)) scans `.claude/skills/` for custom user/repository skills written in Markdown (`SKILL.md`).
3. **Dynamic Skill Tool Injection**: Skills are converted into tool schema definitions that Claude can invoke dynamically during prompt execution.

---

## 🌐 Local API & Server Bridge

Claude Code CLI includes a local HTTP/WebSocket API server:
- **Server Entry**: [src/server/](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/server/) exposes endpoints for local API access, UI remote control, and Desktop app integration.
- **Daemon Process**: [src/daemon/](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/daemon/) manages background processes, process persistence, and session keep-alives.
- **Bridge Protocol**: [src/bridge/](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/bridge/) facilitates bi-directional communication between the CLI process and external frontends (such as the React Desktop UI or VS Code extensions).

---

## 🔑 Key Code Locations

- **Task Handle & Types**: [src/Task.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/Task.ts:72)
- **Agent Sub-Task Tool**: [src/tools/AgentTool/AgentTool.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/tools/AgentTool/AgentTool.ts)
- **Bundled Skills Engine**: [src/skills/bundledSkills.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/skills/bundledSkills.ts)
- **Skills Directory Loader**: [src/skills/loadSkillsDir.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/skills/loadSkillsDir.ts)
- **Local Server API**: [src/server/](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/server/)
- **Daemon Manager**: [src/daemon/](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/daemon/)
