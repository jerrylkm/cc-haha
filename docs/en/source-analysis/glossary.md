# Glossary

| Term | Simple meaning |
|---|---|
| AI provider | The company or server that receives model requests and returns AI responses. |
| API | A structured way for two programs to communicate. Here it often means the network request to a model provider. |
| Agentic turn | One user request plus all model/tool/model iterations needed to finish it. |
| Attachment message | Session metadata represented in the message stream but not ordinary user/assistant chat. |
| Auto-compaction | Automatic summarization of old context when the conversation approaches the model limit. |
| Bash | A command shell. The Bash tool can run general terminal commands and is therefore powerful. |
| CLI | Command-line interface: a program controlled through text in a terminal. |
| Context collapse | Structured preservation/replay of collapsed context, separate from ordinary summarization. |
| Context window | The limited amount of text and other input a model can consider at one time. |
| Deferred tool | A tool whose full schema is loaded only after ToolSearch discovers it. |
| Elicitation | A request from MCP for additional user input, often authorization or a URL action. |
| Feature gate | A build/runtime switch that can include, exclude, or alter behavior. |
| Headless | Running without the normal interactive screen, often for scripts or another application. |
| Headless/SDK mode | Running without the interactive terminal UI and yielding structured SDK events. |
| Hook | User/project/plugin-defined logic invoked at lifecycle points such as before a tool call. |
| JSONL | A text format with one JSON record per line. Session transcripts use this style. |
| MCP | Model Context Protocol, used to connect external tools, resources, prompts, and auth flows. |
| Meta message | Model-visible runtime guidance normally hidden from the user-facing transcript. |
| Micro-compaction | Small cache-aware context reduction, lighter than full summarization. |
| Model | The remote or local AI system that produces responses and tool requests. |
| Orphaned permission | A permission request that survives a disconnect/reconnect and must be resolved later. |
| Permission mode | A policy preset controlling when actions are allowed, denied, or shown for approval. |
| Permission passthrough | A tool asks the central permission system to make the decision. |
| Plugin | An installed extension that can add commands, skills, hooks, or integrations. |
| Prompt | Instructions or content sent to the AI model. It includes more than the newest user sentence. |
| Query | The async-generator state machine for one agentic turn. |
| QueryEngine | Conversation owner that persists state across multiple turns. |
| Reactive compaction | Compaction triggered after the API rejects an already-too-large prompt. |
| Sandbox | A restricted execution environment intended to limit what a command can access. It is not a guarantee of correctness. |
| Schema | A machine-readable description of which input fields a tool accepts. |
| SDK | Software development kit: structured interfaces used by other programs to control or receive events from the runtime. |
| Session | One resumable conversation and its associated state. |
| Sidechain | A subagent/background activity stream linked to a main session. |
| Skill | A reusable instruction/workflow package that guides the agent through a type of task. |
| Snipping | Selectively removing low-value historical material from model context. |
| Task budget | API-level output budget shared across all requests in one agentic turn. |
| Telemetry | Operational events or measurements sent to an analytics/diagnostic system when configured. |
| Token | A unit of model input/output, roughly part of a word. Providers often measure limits and cost in tokens. |
| Tombstone | A message instructing consumers to remove an orphaned partial message. |
| Tool | A specific action the model can request from the local runtime or an external integration. |
| Tool result budget | Limit after which large tool output is persisted and replaced with a preview/reference. |
| ToolUseContext | Runtime dependencies and state supplied to a tool call. |
| Trajectory | Assistant thinking/text/tool request, tool result, and continuation that must remain structurally valid together. |
| Transcript | The saved session record, including more than the visible chat text. |
| TUI | Terminal user interface: an interactive screen drawn inside a terminal. |
| Worktree session | A session associated with an isolated Git worktree and branch state. |
