# 02. The Agent Loop & Query Engine

The **Agent Loop** is the cognitive engine of Claude Code CLI. It receives user prompts, builds rich system context, streams completions from Claude, dispatches tool execution calls, and recurses until the user's task is fulfilled.

---

## 🔄 The Complete Agent Turn Cycle

```
                       +-------------------------+
                       | User Submits Prompt     |
                       +-------------------------+
                                    |
                                    v
                       +-------------------------+
                       | QueryEngine.submit()    |
                       | [src/QueryEngine.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/QueryEngine.ts:187) |
                       +-------------------------+
                                    |
                                    v
                       +-------------------------+
                       | Assemble System Prompt  |
                       | - Git status, CLAUDE.md |
                       | - Active tools schema   |
                       | [src/context.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/context.ts:116)         |
                       +-------------------------+
                                    |
                                    v
                       +-------------------------+
                       | Invoke query() Stream   |
                       | [src/query.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/query.ts:222)        |
                       +-------------------------+
                                    |
            +-----------------------+-----------------------+
            |                                               |
            v                                               v
  (Model Streams Text)                            (Model Emits Tool Call)
            |                                               |
            v                                               v
  +-------------------+                           +-------------------------+
  | Render text to    |                           | Check Permission Rules  |
  | Ink UI stream     |                           | [src/hooks/useCanUseTool.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/hooks/useCanUseTool.ts)|
  +-------------------+                           +-------------------------+
            |                                               |
            v                                               v
  +-------------------+                           +-------------------------+
  | Completion Done?  |                           | Execute Tool via        |
  | Return control    |                           | tool.call(input)        |
  +-------------------+                           +-------------------------+
                                                            |
                                                            v
                                                  +-------------------------+
                                                  | Append Tool Result Block|
                                                  | to Conversation Messages|
                                                  +-------------------------+
                                                            |
                                                            v
                                                  +-------------------------+
                                                  | RECURSE (Next LLM Turn) |
                                                  +-------------------------+
```

---

## 🧠 Core Components Explained

### 1. `QueryEngine` Class
Located in [src/QueryEngine.ts](../src/QueryEngine.ts#L187), `QueryEngine` maintains conversation state, manages memory, coordinates permissions, and provides controls like `interrupt()` to stop running streams.
Key responsibilities:
- **State Maintenance**: Holds the array of `Message` objects (`getMessages()`).
- **File Read Cache**: Maintains file content snapshots (`getReadFileState()`) to avoid re-reading identical files within a session.
- **Model & Cost Tracking**: Tracks token usage and monetary cost via `cost-tracker.ts`.
- **Interruption Handling**: Cancels running streams via `AbortController` when the user presses `Ctrl+C`.

### 2. The `query()` Generator Stream
Located at [src/query.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/query.ts:222), `query()` is an async generator function (`async function* query(...)`) that drives API streaming and recursive tool loops.

Step-by-step workflow inside `query()`:
1. **Message Normalization**: Filters and normalizes conversation messages into API-compliant blocks using `normalizeMessagesForAPI()`.
2. **Context Compaction Check**: Inspects token usage. If context exceeds thresholds, it triggers auto-compaction ([src/services/compact/autoCompact.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/services/compact/autoCompact.ts)).
3. **Streaming API Request**: Sends prompt and tool schemas to the Anthropic API.
4. **Event Processing**: As chunks arrive from Claude (`text_delta`, `tool_use`), events are yielded to the UI for live rendering.
5. **Tool Dispatch**:
   - When a `tool_use` block is completed, `query()` looks up the matching tool definition using `findToolByName()`.
   - Checks permission rules (`useCanUseTool`).
   - Calls `tool.call(input, context)`.
   - Formats the return value as a `ToolResultBlockParam`.
   - Appends the result to message history and triggers another turn automatically.

---

## 🗜️ Context Management & Auto-Compaction

When working in large repositories, conversation history can quickly consume hundreds of thousands of tokens. Claude Code employs a multi-tiered compaction strategy:

```
[Conversation Growth] ──> [Token Threshold Warning] ──> [Microcompaction] ──> [Auto-Compaction]
```

1. **Token Warning Limits**:
   Calculated in [src/services/compact/autoCompact.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/services/compact/autoCompact.ts). Warns when prompt tokens reach ~75% of context window limits.

2. **Microcompaction**:
   Summarizes intermediate tool call outputs (e.g. large file view outputs or bash stdout) into compact tombstone messages using `createMicrocompactBoundaryMessage()`.

3. **Auto-Compaction**:
   Summarizes earlier conversation turns into a structured summary checkpoint (`buildPostCompactMessages()`), preserving user instructions while dropping redundant intermediate tool outputs.

---

## 🔒 Interruptions & Recovery

If the model is executing a long-running bash command or streaming an extensive response, pressing `Ctrl+C`:
- Triggers `queryEngine.interrupt()` ([src/QueryEngine.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/QueryEngine.ts:1199)).
- Aborts active HTTP streams and child process handles.
- Inserts a `createUserInterruptionMessage()` into history, letting Claude know the user cancelled the previous operation.

---

## 🔑 Key Code Locations

- **`QueryEngine` Class**: [src/QueryEngine.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/QueryEngine.ts:187)
- **`query()` Stream Function**: [src/query.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/query.ts:222)
- **Message Types**: [src/types/message.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/types/message.ts)
- **Auto-Compaction Service**: [src/services/compact/autoCompact.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/services/compact/autoCompact.ts)
- **Cost Tracking**: [src/cost-tracker.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/cost-tracker.ts)
