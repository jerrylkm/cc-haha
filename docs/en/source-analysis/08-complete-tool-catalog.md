# 08 — Complete tool catalog

This chapter answers a precise question: **which tools can this source register, and which ones can the model actually see?**

The short answer is that there is no single fixed list at runtime. [`getAllBaseTools()`](../../../src/tools.ts) is the exhaustive built-in registration list for the current build, but the final list changes with build flags, environment variables, product mode, permission rules, feature state, MCP connections, and request options.

## How a non-programmer should use this page

You do not need to memorize the table. Search for the name shown in an approval dialog or activity indicator.

Pay special attention to:

- `Bash` and `PowerShell`, because general shell commands can perform many kinds of actions;
- `Edit`, `Write`, and `NotebookEdit`, because they change local files;
- web, notification, message, PR, and remote tools, because they can create network-visible effects;
- `Agent`, because helper agents can perform additional model calls and tools;
- MCP tools, because their behavior comes from a connected external server;
- rows marked as stubs, because this checkout does not contain enough implementation to explain or verify them.

## 1. The five different “tool lists”

| Stage | Source | Meaning |
|---|---|---|
| Potential built-ins | [`getAllBaseTools()`](../../../src/tools.ts) | Every built-in object that this build/environment could register |
| Enabled built-ins | [`getTools()`](../../../src/tools.ts) | Potential tools after simple/REPL mode, deny-rule, and `isEnabled()` filtering |
| Built-ins plus MCP | [`assembleToolPool()`](../../../src/tools.ts) | Enabled built-ins plus connected MCP tools, filtered, sorted, and deduplicated |
| Agent-specific tools | [`src/constants/tools.ts`](../../../src/constants/tools.ts) and [`src/utils/toolPool.ts`](../../../src/utils/toolPool.ts) | Tools narrowed for subagents, teammates, coordinator mode, or custom agents |
| API-visible tools | [`src/services/api/claude.ts`](../../../src/services/api/claude.ts) | Final schemas sent now; deferred tools may be represented only through ToolSearch |

Therefore:

```text
present in repository
  does not imply imported in this build
  does not imply enabled in this session
  does not imply allowed by permissions
  does not imply exposed directly to the model
```

## 2. Always registered in the base list

These entries are unconditional members of [`getAllBaseTools()`](../../../src/tools.ts). Individual `isEnabled()` checks or later mode filtering can still remove them.

| Model-facing name | Implementation | Purpose and important inputs |
|---|---|---|
| `Agent` | [`AgentTool.tsx`](../../../src/tools/AgentTool/AgentTool.tsx) | Launches a specialized subagent. Inputs include a short description, full prompt, optional agent type/model, background execution, worktree isolation, cwd, name/team, and permission mode. Alias: `Task`. |
| `TaskOutput` | [`TaskOutputTool.tsx`](../../../src/tools/TaskOutputTool/TaskOutputTool.tsx) | Reads or waits for output from a background task/agent. Compatibility aliases: `AgentOutputTool` and `BashOutputTool`. |
| `Bash` | [`BashTool.tsx`](../../../src/tools/BashTool/BashTool.tsx) | Executes a shell command with optional timeout, description, background mode, and policy-controlled sandbox override. Its result records stdout, stderr, interruption, background task ID, and sandbox state. |
| `ExitPlanMode` | [`ExitPlanModeV2Tool.ts`](../../../src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts) | Submits a completed plan and requests transition from planning to execution. It is main-thread state, so ordinary subagents do not receive it. |
| `Read` | [`FileReadTool.ts`](../../../src/tools/FileReadTool/FileReadTool.ts) | Reads an absolute path with optional line offset/limit or PDF page range. It supports text, images, notebooks, PDFs, extracted PDF pages, and unchanged-file responses. |
| `Edit` | [`FileEditTool.ts`](../../../src/tools/FileEditTool/FileEditTool.ts) | Performs exact string replacement in one file using `file_path`, `old_string`, `new_string`, and optional `replace_all`. Produces a structured patch and Git diff metadata. |
| `Write` | [`FileWriteTool.ts`](../../../src/tools/FileWriteTool/FileWriteTool.ts) | Creates or overwrites a complete file from `file_path` and `content`, returning original content and a structured patch. |
| `NotebookEdit` | [`NotebookEditTool.ts`](../../../src/tools/NotebookEditTool/NotebookEditTool.ts) | Replaces, inserts, or deletes Jupyter notebook cells while preserving notebook structure rather than editing raw JSON text. |
| `WebFetch` | [`WebFetchTool.ts`](../../../src/tools/WebFetchTool/WebFetchTool.ts) | Fetches one URL and applies a prompt to the retrieved content. Permission rules are reduced to `domain:<hostname>`. This tool is deferred. |
| `TodoWrite` | [`TodoWriteTool.ts`](../../../src/tools/TodoWriteTool/TodoWriteTool.ts) | Updates the legacy in-session todo list shown by the UI. It is hidden when the newer task system is active. |
| `WebSearch` | [`WebSearchTool.ts`](../../../src/tools/WebSearchTool/WebSearchTool.ts) | Performs web search from a query with optional allowed/blocked domains. The server-side web-search primitive is capped at eight uses per call. |
| `TaskStop` | [`TaskStopTool.ts`](../../../src/tools/TaskStopTool/TaskStopTool.ts) | Stops a running background task. Compatibility alias: `KillShell`. |
| `AskUserQuestion` | [`AskUserQuestionTool.tsx`](../../../src/tools/AskUserQuestionTool/AskUserQuestionTool.tsx) | Presents one to four questions, each with two to four options, optional multi-select and previews. The UI automatically supplies free-form “Other” handling. |
| `Skill` | [`SkillTool.ts`](../../../src/tools/SkillTool/SkillTool.ts) | Invokes a discovered skill with arguments. A skill can execute inline or in a forked context and can register hooks or additional instructions. |
| `EnterPlanMode` | [`EnterPlanModeTool.ts`](../../../src/tools/EnterPlanModeTool/EnterPlanModeTool.ts) | Moves the main conversation into restricted planning mode before implementation. |
| `SendMessage` | [`SendMessageTool.ts`](../../../src/tools/SendMessageTool/SendMessageTool.ts) | Sends messages among team members/agents and supports team coordination semantics. It is registered unconditionally but has its own enablement checks. |
| `SendUserMessage` | [`BriefTool.ts`](../../../src/tools/BriefTool/BriefTool.ts) | Sends a concise user-facing message in assistant-style modes. Compatibility alias: `Brief`. |

### Search-tool substitution

`Glob` and `Grep` are normally included beside the tools above:

| Name | Implementation | Purpose |
|---|---|---|
| `Glob` | [`GlobTool.ts`](../../../src/tools/GlobTool/GlobTool.ts) | Finds paths by glob pattern, defaults to cwd, returns at most 100 files, and is read-only/concurrency-safe. |
| `Grep` | [`GrepTool.ts`](../../../src/tools/GrepTool/GrepTool.ts) | Uses ripgrep-compatible regular-expression search with path/glob/type filters, context lines, case/multiline controls, offsets, and output modes. Its default result cap is 250 entries. |

They are omitted when [`hasEmbeddedSearchTools()`](../../../src/utils/embeddedTools.ts) reports that the Bun binary already embeds `bfs`/`ugrep` and aliases shell `find`/`grep` to those implementations. In that build, shell search replaces the dedicated schemas.

## 3. Conditionally registered concrete tools

The implementations below are present in this checkout, but registration requires the stated condition.

| Name | Condition in [`src/tools.ts`](../../../src/tools.ts) | Purpose |
|---|---|---|
| `Config` | `USER_TYPE=ant` | Reads/changes internal configuration through [`ConfigTool.ts`](../../../src/tools/ConfigTool/ConfigTool.ts). |
| `tungsten` | `USER_TYPE=ant` | Internal virtual-terminal capability implemented by [`TungstenTool.ts`](../../../src/tools/TungstenTool/TungstenTool.ts). It is excluded from async agents because it uses singleton terminal state. |
| `TaskCreate` | New task system enabled | Creates a durable task with subject, description, active form, and dependency metadata through [`TaskCreateTool.ts`](../../../src/tools/TaskCreateTool/TaskCreateTool.ts). |
| `TaskGet` | New task system enabled | Retrieves one task by ID through [`TaskGetTool.ts`](../../../src/tools/TaskGetTool/TaskGetTool.ts). |
| `TaskUpdate` | New task system enabled | Changes task status/details/dependencies through [`TaskUpdateTool.ts`](../../../src/tools/TaskUpdateTool/TaskUpdateTool.ts). |
| `TaskList` | New task system enabled | Lists current tasks through [`TaskListTool.ts`](../../../src/tools/TaskListTool/TaskListTool.ts). |
| `LSP` | `ENABLE_LSP_TOOL` truthy and tool enabled | Executes language-server operations through [`LSPTool.ts`](../../../src/tools/LSPTool/LSPTool.ts). |
| `EnterWorktree` | Worktree mode enabled | Creates/enters isolated Git worktree state through [`EnterWorktreeTool.ts`](../../../src/tools/EnterWorktreeTool/EnterWorktreeTool.ts). |
| `ExitWorktree` | Worktree mode enabled | Leaves and reconciles worktree mode through [`ExitWorktreeTool.ts`](../../../src/tools/ExitWorktreeTool/ExitWorktreeTool.ts). |
| `TeamCreate` | Agent swarms enabled | Creates team metadata and coordination state through [`TeamCreateTool.ts`](../../../src/tools/TeamCreateTool/TeamCreateTool.ts). |
| `TeamDelete` | Agent swarms enabled | Removes a team after lifecycle checks through [`TeamDeleteTool.ts`](../../../src/tools/TeamDeleteTool/TeamDeleteTool.ts). |
| `PowerShell` | Windows/PowerShell enablement check succeeds | Executes PowerShell with its platform-specific parsing, permissions, and output handling in [`PowerShellTool.tsx`](../../../src/tools/PowerShellTool/PowerShellTool.tsx). |
| `CronCreate` | `AGENT_TRIGGERS` build feature | Creates a scheduled trigger through [`CronCreateTool.ts`](../../../src/tools/ScheduleCronTool/CronCreateTool.ts). |
| `CronUpdate` | `AGENT_TRIGGERS` build feature | Updates an existing trigger through [`CronUpdateTool.ts`](../../../src/tools/ScheduleCronTool/CronUpdateTool.ts). |
| `CronDelete` | `AGENT_TRIGGERS` build feature | Deletes a trigger through [`CronDeleteTool.ts`](../../../src/tools/ScheduleCronTool/CronDeleteTool.ts). |
| `CronList` | `AGENT_TRIGGERS` build feature | Lists triggers through [`CronListTool.ts`](../../../src/tools/ScheduleCronTool/CronListTool.ts). |
| `RemoteTrigger` | `AGENT_TRIGGERS_REMOTE` build feature | Creates or activates remote-trigger behavior through [`RemoteTriggerTool.ts`](../../../src/tools/RemoteTriggerTool/RemoteTriggerTool.ts). |
| `ToolSearch` | Optimistic tool-search enablement | Searches deferred tool metadata and loads matching schemas through [`ToolSearchTool.ts`](../../../src/tools/ToolSearchTool/ToolSearchTool.ts). |
| `TestingPermission` | `NODE_ENV=test` | Test-only permission behavior from [`TestingPermissionTool.tsx`](../../../src/tools/testing/TestingPermissionTool.tsx); not a production capability. |

## 4. Registered helper tools for MCP

Two helper objects occur in [`getAllBaseTools()`](../../../src/tools.ts) but are deliberately removed from the ordinary built-in list. MCP connection setup adds them when relevant:

| Name | Implementation | Purpose |
|---|---|---|
| `ListMcpResourcesTool` | [`ListMcpResourcesTool.ts`](../../../src/tools/ListMcpResourcesTool/ListMcpResourcesTool.ts) | Lists resources advertised by connected MCP servers. |
| `ReadMcpResourceTool` | [`ReadMcpResourceTool.ts`](../../../src/tools/ReadMcpResourceTool/ReadMcpResourceTool.ts) | Reads one advertised MCP resource URI. |

Each remote MCP tool is then created dynamically from [`MCPTool.ts`](../../../src/tools/MCPTool/MCPTool.ts). There cannot be a static exhaustive name list because names and schemas come from the user’s connected servers. They normally follow a server/tool-derived naming convention, unless an SDK no-prefix mode is active.

[`assembleToolPool()`](../../../src/tools.ts) filters MCP entries with the same blanket deny rules, sorts built-ins and MCP tools separately for prompt-cache stability, and deduplicates by name with the built-in winning.

## 5. Structured-output implementation tool

`StructuredOutput` is intentionally excluded from normal [`getTools()`](../../../src/tools.ts) filtering. When a headless request supplies a valid JSON schema, [`src/main.tsx`](../../../src/main.tsx) creates a request-specific tool with [`createSyntheticOutputTool()`](../../../src/tools/SyntheticOutputTool/SyntheticOutputTool.ts) and appends it afterward.

This is not a general user capability. It forces the model’s final response to conform to the requested schema.

## 6. Feature-gated modules that are only stubs here

The registration source refers to additional internal tools, but their checked-in files are generated proxy placeholders marked `@generated stub from scan-missing-imports`. Their real schemas, names, permission behavior, and execution code are **not present**, so only the registration symbol and gate can be stated reliably.

| Registration symbol | Gate | Stub location | What can safely be concluded |
|---|---|---|---|
| `SuggestBackgroundPRTool` | `USER_TYPE=ant` | [`SuggestBackgroundPRTool.ts`](../../../src/tools/SuggestBackgroundPRTool/SuggestBackgroundPRTool.ts) | Intended to suggest background PR work; implementation unavailable. |
| `WebBrowserTool` | `WEB_BROWSER_TOOL` | [`WebBrowserTool.ts`](../../../src/tools/WebBrowserTool/WebBrowserTool.ts) | Browser-related tool; exact API unavailable. |
| `OverflowTestTool` | `OVERFLOW_TEST_TOOL` | [`OverflowTestTool.ts`](../../../src/tools/OverflowTestTool/OverflowTestTool.ts) | Internal overflow test capability; exact API unavailable. |
| `CtxInspectTool` | `CONTEXT_COLLAPSE` | [`CtxInspectTool.ts`](../../../src/tools/CtxInspectTool/CtxInspectTool.ts) | Context-collapse inspection capability; exact API unavailable. |
| `TerminalCaptureTool` | `TERMINAL_PANEL` | [`TerminalCaptureTool.ts`](../../../src/tools/TerminalCaptureTool/TerminalCaptureTool.ts) | Terminal-panel capture capability; exact API unavailable. |
| `ListPeersTool` | `UDS_INBOX` | [`ListPeersTool.ts`](../../../src/tools/ListPeersTool/ListPeersTool.ts) | Peer discovery for an inbox/IPC mode; exact API unavailable. |
| `VerifyPlanExecutionTool` | `CLAUDE_CODE_VERIFY_PLAN=true` | [`VerifyPlanExecutionTool.ts`](../../../src/tools/VerifyPlanExecutionTool/VerifyPlanExecutionTool.ts) | Plan-execution verification capability; exact API unavailable. |
| `REPLTool` | `USER_TYPE=ant` and REPL mode/build support | [`REPLTool.ts`](../../../src/tools/REPLTool/REPLTool.ts) | Wraps primitive operations in an internal REPL/VM mode; exact implementation unavailable. |
| `WorkflowTool` | `WORKFLOW_SCRIPTS` | [`WorkflowTool.ts`](../../../src/tools/WorkflowTool/WorkflowTool.ts) | Runs bundled workflow scripts; exact implementation unavailable. The declared name constant is `workflow`. |
| `SleepTool` | `PROACTIVE` or `KAIROS` | [`SleepTool.ts`](../../../src/tools/SleepTool/SleepTool.ts) | Wait/sleep capability for proactive/assistant mode; declared name is `Sleep`, implementation unavailable. |
| `MonitorTool` | `MONITOR_TOOL` | [`MonitorTool.ts`](../../../src/tools/MonitorTool/MonitorTool.ts) | Monitoring capability; exact API unavailable. |
| `SendUserFileTool` | `KAIROS` | [`SendUserFileTool.ts`](../../../src/tools/SendUserFileTool/SendUserFileTool.ts) | User-file delivery capability; exact API unavailable. |
| `PushNotificationTool` | `KAIROS` or `KAIROS_PUSH_NOTIFICATION` | [`PushNotificationTool.ts`](../../../src/tools/PushNotificationTool/PushNotificationTool.ts) | Push-notification capability; exact API unavailable. |
| `SubscribePRTool` | `KAIROS_GITHUB_WEBHOOKS` | [`SubscribePRTool.ts`](../../../src/tools/SubscribePRTool/SubscribePRTool.ts) | PR webhook/subscription capability; exact API unavailable. |
| `SnipTool` | `HISTORY_SNIP` | [`SnipTool.ts`](../../../src/tools/SnipTool/SnipTool.ts) | History-snipping capability; exact API unavailable. |

The proxy stubs return callable proxies for arbitrary properties. They exist so builds in which dead-code elimination removes these paths can still resolve imports. They are not evidence that the internal feature works in this reconstructed project.

## 7. Simple, REPL, coordinator, and agent filtering

### Simple mode

With `CLAUDE_CODE_SIMPLE`, [`getTools()`](../../../src/tools.ts) normally returns only:

- `Bash`
- `Read`
- `Edit`

Coordinator mode additionally needs `Agent`, `TaskStop`, and `SendMessage`. In internal REPL mode, the wrapper can replace the raw primitives.

### REPL mode

If the REPL wrapper is enabled, names in [`REPL_ONLY_TOOLS`](../../../src/tools/REPLTool/constants.ts) are hidden from direct model access because the VM wrapper exposes them internally.

### Coordinator mode

[`COORDINATOR_MODE_ALLOWED_TOOLS`](../../../src/constants/tools.ts) narrows the coordinator to:

- `Agent`
- `TaskStop`
- `SendMessage`
- `StructuredOutput`

Workers receive their own filtered pool.

### Ordinary asynchronous agents

[`ASYNC_AGENT_ALLOWED_TOOLS`](../../../src/constants/tools.ts) permits:

- `Read`, `Glob`, `Grep`
- `Bash`/supported shell names
- `Edit`, `Write`, `NotebookEdit`
- `WebSearch`, `WebFetch`
- `TodoWrite`, `Skill`, `ToolSearch`
- `StructuredOutput`
- `EnterWorktree`, `ExitWorktree`

The allowlist intentionally omits recursive agent/task-output control, plan-mode transitions, user questions, main-thread task stop, MCP tools, and singleton terminal tools.

In-process teammates additionally receive `TaskCreate`, `TaskGet`, `TaskList`, `TaskUpdate`, `SendMessage`, and selected cron operations.

## 8. Blanket deny and `isEnabled()` filtering

A permission rule that denies an entire tool—with no narrower rule content—removes that tool before the model sees it. MCP server-prefix denials can similarly remove every tool from a server.

After deny filtering, [`getTools()`](../../../src/tools.ts) invokes each tool’s `isEnabled()`. Examples include:

- legacy `TodoWrite` versus the new task tools;
- plan-mode and question availability;
- background task output support;
- PowerShell platform support;
- proactive `Sleep` activation;
- web search/provider capability;
- tool-search threshold/configuration.

This differs from a call-time denial: a filtered tool is absent from the prompt, while a call-time denial returns an error/result to a model that already requested it.

## 9. Deferred tools

A tool with `shouldDefer: true` can be sent with deferred loading instead of its full schema. ToolSearch receives compact searchable metadata. Once the model searches for and selects a tool, the runtime marks its full schema as available.

This saves prompt tokens when many MCP or specialized tools are connected. `alwaysLoad` can override deferral for a tool that must be visible on the first model request.

## 10. Tool lifecycle in concrete terms

For a request such as:

```json
{
  "name": "Edit",
  "input": {
    "file_path": "/repo/src/app.ts",
    "old_string": "const port = 80",
    "new_string": "const port = 8080"
  }
}
```

the runtime:

1. finds `Edit` by primary name or alias;
2. parses the strict Zod input schema;
3. runs tool-specific validation;
4. builds permission-rule matching from the normalized absolute path;
5. runs pre-tool and permission hooks;
6. resolves allow/ask/deny, possibly through the UI;
7. executes the replacement and tracks file history;
8. returns original content, structured patch, and optional Git diff;
9. runs post-tool hooks;
10. converts the output to an API `tool_result`;
11. persists oversized results when the tool’s threshold permits;
12. renders a separate terminal result representation;
13. appends the result and asks the model what to do next.

See [04 — Tools, permissions, and security](04-tools-permissions-security.md) for the security decisions inside these stages.

## 11. Exhaustiveness statement

This catalog includes:

- every entry and conditional spread in [`getAllBaseTools()`](../../../src/tools.ts);
- the dedicated `Glob`/`Grep` substitution branch;
- the MCP resource helpers;
- arbitrary dynamic MCP tools;
- the separately injected `StructuredOutput` tool;
- simple, REPL, coordinator, async-agent, and teammate filters;
- known compatibility aliases;
- test-only and generated-stub registrations.

It does **not** claim static names for arbitrary MCP servers or private implementations absent from the checkout. Doing so would be fabricated rather than exhaustive.

“Exhaustive” here means every registration path visible in this checkout, not every action a shell command, plugin, hook, skill, or external MCP server could possibly perform.
