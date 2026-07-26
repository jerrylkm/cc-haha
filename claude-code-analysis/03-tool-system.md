# 03. Tool System & Execution Engine

Tools give Claude Code CLI agency—enabling it to read and write files, execute bash commands, search codebases, and interact with external systems via the Model Context Protocol (MCP).

---

## 🛠️ Tool Architecture Overview

All tools in Claude Code CLI share a uniform definition and execution pattern built around Zod validation schemas and permission policies:

```
┌─────────────────────────────────────────────────────────────┐
│                       Tool Definition                       │
│ Location: [src/Tool.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/Tool.ts:364)                                 │
│ - Name, Description                                         │
│ - Input Zod JSON Schema                                     │
│ - Permissions & Approval Policy                             │
└─────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                      Permission Gate                        │
│ Location: [src/hooks/useCanUseTool.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/hooks/useCanUseTool.ts)                      │
│ - Auto-approve read-only tools (if configured)              │
│ - Prompt user for confirmation (Bash, Edit, dangerous tools) │
└─────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                       Tool Execution                        │
│ - Call `tool.call(input, context)`                          │
│ - Return structured result or error message                 │
└─────────────────────────────────────────────────────────────┘
```

---

## 📐 Tool Schema & Construction

Tools are typed and constructed using `buildTool()` in [src/Tool.ts](../src/Tool.ts#L785):
```typescript
// Simplified structural representation from Tool.ts
export type Tool<InputSchema, ResultType> = {
  name: string
  description: string
  inputSchema: InputSchema
  isReadOnly: (input: Input) => boolean
  userFacingName: (input: Input) => string
  call: (input: Input, context: ToolUseContext) => Promise<ToolResult<ResultType>>
  renderToolUseMessage?: (input: Input, options: RenderOptions) => React.ReactNode
  renderToolResultMessage?: (result: ResultType, options: RenderOptions) => React.ReactNode
}
```

Key features of this structure:
1. **Zod Validation**: Input parameters sent by the model are strictly validated against JSON schema before execution.
2. **Read-Only Inspection**: Tools declare whether an operation is read-only (`isReadOnly()`) or mutates system state.
3. **Custom UI Rendering**: Tools provide custom React-Ink components (`renderToolUseMessage`, `renderToolResultMessage`) to present rich interactive output in the terminal.

---

## 🧰 Primary Built-in Tools

Loaded via `getAllBaseTools()` in [src/tools.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/tools.ts:194):

| Tool Name | Source File | Function & Behavior |
| :--- | :--- | :--- |
| **`Bash`** | [src/tools/BashTool/BashTool.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/tools/BashTool/BashTool.ts) | Executes arbitrary shell commands. Supports sync/async modes, process detach, output streaming, and security sanitization filters. |
| **`FileEdit`** | [src/tools/FileEditTool/FileEditTool.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/tools/FileEditTool/FileEditTool.ts) | Performs surgical exact string replacement in source files. Includes match uniqueness checks and auto-backup safety. |
| **`FileRead` / `View`** | [src/tools/FileReadTool/FileReadTool.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/tools/FileReadTool/FileReadTool.ts) | Reads file contents with support for line-range pagination (`view_range`), size truncation (20KB safety threshold), and image base64 conversion. |
| **`Glob`** | [src/tools/GlobTool/GlobTool.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/tools/GlobTool/GlobTool.ts) | Fast file pattern matching using glob patterns across directory trees. |
| **`Grep`** | [src/tools/GrepTool/GrepTool.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/tools/GrepTool/GrepTool.ts) | Fast code search using ripgrep (`rg`) with regex pattern support, line context flags (`-A`, `-B`, `-C`), and output mode choices. |
| **`Agent` / `Task`** | [src/tools/AgentTool/AgentTool.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/tools/AgentTool/AgentTool.ts) | Spawns background sub-agents in separate context windows for sub-tasks (explore, research, code-review). |
| **`WebFetch`** | [src/tools/WebFetchTool/WebFetchTool.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/tools/WebFetchTool/WebFetchTool.ts) | Fetches web pages over HTTP and converts HTML to simplified Markdown. |

---

## 🌐 Model Context Protocol (MCP) Integration

Claude Code CLI includes native support for Anthropic's **Model Context Protocol (MCP)**:

- **Server Manager**: [src/services/mcp/McpServerManager.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/services/mcp/McpServerManager.ts) handles connections to external MCP servers (stdio, SSE, HTTP).
- **Official Registry**: [src/services/mcp/officialRegistry.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/services/mcp/officialRegistry.ts) resolves certified MCP tools.
- **Dynamic Conversion**: MCP tools are dynamically wrapped into native `Tool` instances at runtime and merged into the active tool pool inside `assembleToolPool()` in [src/tools.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/tools.ts:346).

---

## 🛡️ Permission System & Approval Policies

Before any non-read-only tool executes:
1. `useCanUseTool` ([src/hooks/useCanUseTool.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/hooks/useCanUseTool.ts)) inspects the permission context (`ToolPermissionContext`).
2. If auto-approval is enabled for read operations or the command matches pre-approved rules in `.claude/settings.json`, execution proceeds without prompt.
3. If user confirmation is needed, the Ink UI displays an interactive dialog ([src/services/mcpServerApproval.tsx](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/services/mcpServerApproval.tsx)) prompting the user to Allow or Deny the tool execution.

---

## 🔑 Key Code Locations

- **Tool Types & Builder**: [src/Tool.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/Tool.ts:364)
- **Tool Aggregator**: [src/tools.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/tools.ts:194)
- **Bash Tool**: [src/tools/BashTool/BashTool.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/tools/BashTool/BashTool.ts)
- **File Edit Tool**: [src/tools/FileEditTool/FileEditTool.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/tools/FileEditTool/FileEditTool.ts)
- **Permission Hook**: [src/hooks/useCanUseTool.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/hooks/useCanUseTool.ts)
- **MCP Server Manager**: [src/services/mcp/McpServerManager.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/services/mcp/McpServerManager.ts)
