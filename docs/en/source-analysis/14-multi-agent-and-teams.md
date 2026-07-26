# 14 — Multi-agent, subagents, and teams

This chapter explains how one Claude Code session can spawn other AI workers: quick helper **subagents**, isolated **worktree/remote** agents, and coordinated **teams**. It analyzes the actual source, so it lives here rather than deferring to the repository’s separate agent guide.

Relevant source: [`src/tools/AgentTool/`](../../../src/tools/AgentTool/), [`src/utils/swarm/`](../../../src/utils/swarm/), [`src/coordinator/`](../../../src/coordinator/), [`src/remote/`](../../../src/remote/), and the team/task tools under [`src/tools/`](../../../src/tools/).

## 1. What a subagent is

A subagent is a second AI run, given a focused prompt and its own tool set, launched by the `Agent` tool ([`AgentTool.tsx`](../../../src/tools/AgentTool/AgentTool.tsx)). Its primary name is `Agent` and its compatibility alias is `Task` (`AGENT_TOOL_NAME` / `LEGACY_AGENT_TOOL_NAME` in [`constants.ts`](../../../src/tools/AgentTool/constants.ts)). Its result is returned to the parent conversation.

The `Agent` tool input schema (defined across `baseInputSchema`, `fullInputSchema`, and `inputSchema` in [`AgentTool.tsx`](../../../src/tools/AgentTool/AgentTool.tsx)) includes:

| Field | Schema line | Meaning |
|---|---|---|
| `description` | ~83 | 3–5 word label shown in the UI |
| `prompt` | ~84 | The task the subagent should perform |
| `subagent_type` | ~85 | Which specialized agent to use (omit to “fork” when that gate is on) |
| `model` | ~86 | Optional override: `fable`, `sonnet`, `opus`, or `haiku` |
| `run_in_background` | ~87 | Run asynchronously and notify on completion |
| `name`, `team_name` | ~94–95 | Make the agent addressable within a team via `SendMessage({to: name})` |
| `mode` | ~96 | Permission mode for the spawned agent (e.g. `plan`) |
| `isolation` | ~99 | `worktree` (isolated git copy) or `remote` (internal-only) |
| `cwd` | ~100 | Run in a specific directory (mutually exclusive with worktree) |

The `cwd` and `run_in_background` fields are stripped from the model-facing schema when their gates are off (via `.omit()` around lines 111–124), which is why the effective schema varies by build.

## 2. Sync, background, worktree, remote

Four execution shapes, dispatched in [`AgentTool.tsx`](../../../src/tools/AgentTool/AgentTool.tsx) and [`runAgent.ts`](../../../src/tools/AgentTool/runAgent.ts):

- **Sync (foreground):** default. Runs inline; the parent waits and receives the full result.
- **Background (async):** `run_in_background: true`, or an agent defined with `background: true`, or a long run auto-backgrounded. The decision is the `shouldRunAsync` expression (~line 567 of [`AgentTool.tsx`](../../../src/tools/AgentTool/AgentTool.tsx)), which is also forced on when fork mode, coordinator mode, or proactive mode is active. Auto-background uses `getAutoBackgroundMs()` (default `120_000` ms — about two minutes). Async returns `agentId` and an `outputFile` to poll; completion arrives as a user-role `<task-notification>` message. Async agents commonly default to the cheaper `fable` model.
- **Worktree isolation:** creates a temporary git worktree so the agent edits an isolated copy of the repository; the worktree is cleaned up if no changes were made, otherwise its path is returned.
- **Remote isolation:** internal-only; launches in a remote CCR environment (always async) and returns a session URL. See [17 — Remote access and IM bridge](17-remote-access-and-bridge.md).

## 3. How agents are defined

Custom agents are Markdown files with YAML frontmatter, discovered by [`loadAgentsDir.ts`](../../../src/tools/AgentTool/loadAgentsDir.ts) from user (`~/.claude/agents/`) and project (`agents/`) locations. Frontmatter fields include `name`, `description`, `model` (or `inherit`), `effort`, `tools` (allowlist), `disallowedTools` (denylist), `permissionMode`, `maxTurns`, `background`, `memory`, `isolation`, `color`, `skills`, `mcpServers`, and `hooks`; the body is the system prompt.

Built-in agents live in [`builtInAgents.ts`](../../../src/tools/AgentTool/builtInAgents.ts) (for example `general-purpose`, plus gated `explore`/`plan`/`verification`/`claude-code-guide`). Loading precedence runs built-in → plugin → user → project → flag, with later sources overriding earlier ones and the nearest working directory winning within a source. Tool resolution (`resolveAgentTools` in [`agentToolUtils.ts`](../../../src/tools/AgentTool/agentToolUtils.ts)) applies the allowlist (`*` = all) then the denylist; invalid tool names are logged but do not fail the spawn.

## 4. Fork subagents and prompt-cache sharing

When the fork feature is enabled (`isForkSubagentEnabled()`, ~line 32) and `subagent_type` is omitted, the `Agent` tool creates a **fork** ([`forkSubagent.ts`](../../../src/tools/AgentTool/forkSubagent.ts)). The fork definition `FORK_AGENT` (~line 60) sets `tools: ['*']`, `model: 'inherit'`, `permissionMode: 'bubble'`, and `useExactTools: true`, so the child reuses the parent’s exact tool set and model. `buildForkedMessages()` (~line 107) reconstructs the history so that all fork children share a byte-identical request prefix, maximizing prompt-cache reuse: each parent tool_use block is filled with the identical placeholder `FORK_PLACEHOLDER_RESULT` (`"Fork started — processing in background"`, ~line 93), and only a trailing per-child directive (`buildChildMessage()`, ~line 171) differs. `isInForkChild()` (~line 78) detects fork boilerplate in history to prevent recursive forking.

For a user, forking is an optimization: it runs a focused sub-task that inherits full context cheaply, when the intermediate output is not worth keeping in the main thread.

## 5. Teams and the mailbox

Teams let several named agents coordinate. `TeamCreate` ([`TeamCreateTool.ts`](../../../src/tools/TeamCreateTool/TeamCreateTool.ts)) writes a team file (under `~/.claude/teams/{team}/team.json`) recording the lead agent and members (name, agent type, model, cwd, permission mode, backend, shared allowed paths).

Members communicate through a file-based mailbox ([`SendMessageTool.ts`](../../../src/tools/SendMessageTool/SendMessageTool.ts) with helpers in [`src/utils/swarm/`](../../../src/utils/swarm/)). `SendMessage` supports a direct `to: "name"` or a broadcast `to: "*"`, and structured messages such as shutdown or plan-approval responses. Team state is written under a lock: [`teamHelpers.ts`](../../../src/utils/swarm/teamHelpers.ts) uses `lockfile.lock()` (~line 206) around the team file and cleans up the team directory `~/.claude/teams/{team-name}/` (~line 705), so concurrent agents serialize safely.

Teammates can be spawned **in-process** ([`spawnInProcess.ts`](../../../src/utils/swarm/spawnInProcess.ts), driven by [`inProcessRunner.ts`](../../../src/utils/swarm/inProcessRunner.ts)) via `Agent` with `name` + `team_name`, isolated with an abort controller and async-local context, or in separate terminal panes (tmux/iTerm2). In-process teammates additionally receive the task tools (`TaskCreate`/`TaskGet`/`TaskList`/`TaskUpdate`) and `SendMessage`, per `IN_PROCESS_TEAMMATE_ALLOWED_TOOLS` (~line 77 of [`constants/tools.ts`](../../../src/constants/tools.ts)), so they can share a task list with dependencies.

## 6. Coordinator mode

Coordinator mode ([`coordinatorMode.ts`](../../../src/coordinator/coordinatorMode.ts)) is enabled by `CLAUDE_CODE_COORDINATOR_MODE` behind a feature gate (checked in [`main.tsx`](../../../src/main.tsx) ~line 1886 and [`AgentTool.tsx`](../../../src/tools/AgentTool/AgentTool.tsx) ~lines 223/553). The coordinator orchestrates but does not run ordinary tools itself: its pool is limited to `Agent`, `SendMessage`, `TaskStop`, and `StructuredOutput` (`COORDINATOR_MODE_ALLOWED_TOOLS`, ~line 107 of [`constants/tools.ts`](../../../src/constants/tools.ts)). Workers it spawns receive the restricted `ASYNC_AGENT_ALLOWED_TOOLS` set (~line 55 of the same file). Worker results return to the coordinator as `<task-notification>` messages rather than as the coordinator’s own output.

## 7. Remote agents

The `remote` isolation path and remote sessions are managed by [`RemoteSessionManager.ts`](../../../src/remote/RemoteSessionManager.ts). It maintains a WebSocket subscription for inbound messages plus HTTP for outbound messages, and it relays SDK control requests such as permission and cancel. Remote agent launches return a task id and a user-facing session URL. The transport details are covered in [17 — Remote access and IM bridge](17-remote-access-and-bridge.md).

## 8. What this means for you as a user

- One request can spawn several AI workers; each worker uses additional model calls, so multi-agent work can be faster but more expensive.
- A subagent’s tools may be narrower than the main conversation’s; ordinary async agents cannot recursively spawn agents or use main-thread control tools.
- Worktree isolation is the safest way to let an agent make changes without touching your main working tree.
- Background agents keep running until they finish or are stopped; use the task/stop controls and the reported output file to follow them.
- Team messages and inboxes are stored as local files under `~/.claude/teams/`.

## 9. Source reference (line-level)

| Concern | Symbol | Location |
|---|---|---|
| Tool name / alias | `AGENT_TOOL_NAME`, `LEGACY_AGENT_TOOL_NAME` | [`constants.ts`](../../../src/tools/AgentTool/constants.ts):1,3 |
| Input schema | `baseInputSchema`/`fullInputSchema`/`inputSchema` | [`AgentTool.tsx`](../../../src/tools/AgentTool/AgentTool.tsx):~82–125 |
| Async decision | `shouldRunAsync`, `getAutoBackgroundMs` (120,000 ms) | [`AgentTool.tsx`](../../../src/tools/AgentTool/AgentTool.tsx):~74,567 |
| Fork | `FORK_AGENT`, `buildForkedMessages`, `FORK_PLACEHOLDER_RESULT`, `isInForkChild` | [`forkSubagent.ts`](../../../src/tools/AgentTool/forkSubagent.ts):32,60,78,93,107,171 |
| Agent loading | frontmatter schema, `source` discriminants | [`loadAgentsDir.ts`](../../../src/tools/AgentTool/loadAgentsDir.ts):~78–184 |
| Built-in agents | `EXPLORE_AGENT`, `PLAN_AGENT`, `VERIFICATION_AGENT`, `STATUSLINE_SETUP_AGENT` | [`builtInAgents.ts`](../../../src/tools/AgentTool/builtInAgents.ts):6–10 |
| Team file + lock | `lockfile.lock`, team dir cleanup | [`teamHelpers.ts`](../../../src/utils/swarm/teamHelpers.ts):206,705 |
| In-process spawn | `spawnInProcess`, `inProcessRunner` | [`src/utils/swarm/`](../../../src/utils/swarm/) |
| Allowed-tool sets | `ASYNC_AGENT_ALLOWED_TOOLS`, `IN_PROCESS_TEAMMATE_ALLOWED_TOOLS`, `COORDINATOR_MODE_ALLOWED_TOOLS` | [`constants/tools.ts`](../../../src/constants/tools.ts):55,77,107 |
| Coordinator gate | `CLAUDE_CODE_COORDINATOR_MODE` | [`main.tsx`](../../../src/main.tsx):1886 |
| Remote session | `RemoteSessionManager` | [`RemoteSessionManager.ts`](../../../src/remote/RemoteSessionManager.ts):95 |

Line numbers are approximate and may drift as the source changes; the symbol names are the durable anchors.

## 10. Real vs stub

The agent, team, task, swarm, and remote files listed above are real source in this checkout. Parts of coordinator mode and some team backends are feature-gated; the tool catalog ([08](08-complete-tool-catalog.md)) marks any stubbed tools individually.
