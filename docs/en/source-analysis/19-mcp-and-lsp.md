# 19 — MCP and LSP integration

This chapter explains how Claude Code connects to two kinds of external helpers: **MCP** servers (which add tools, resources, and prompts) and **LSP** language servers (which answer code-intelligence questions like “go to definition”). It analyzes the real source so the coverage is self-contained; it expands the summary in [05 §9–§10](05-context-persistence-services.md).

Relevant source: [`src/services/mcp/`](../../../src/services/mcp/), [`src/services/lsp/`](../../../src/services/lsp/), [`src/tools/MCPTool/`](../../../src/tools/MCPTool/), [`src/tools/ListMcpResourcesTool/`](../../../src/tools/ListMcpResourcesTool/), [`src/tools/ReadMcpResourceTool/`](../../../src/tools/ReadMcpResourceTool/), and [`src/tools/LSPTool/`](../../../src/tools/LSPTool/).

## 1. What MCP is, in plain terms

The Model Context Protocol (MCP) is a standard way to plug an external program or service into the agent so it can offer extra tools (actions), resources (readable data), and prompts. Because these come from a server you configure, their names and behavior are dynamic — the built-in [tool catalog](08-complete-tool-catalog.md) cannot list them all.

## 2. MCP configuration model

Config types live in [`src/services/mcp/types.ts`](../../../src/services/mcp/types.ts). A server config (`McpServerConfig`, a union) can use several transports:

- **stdio** — run a local `command` with `args`/`env` (~lines 28–34);
- **SSE** and **HTTP (streamable)** — a `url` with optional `headers`/`headersHelper` and OAuth (~lines 58–97);
- **WebSocket** — a `url` transport (~lines 99–106);
- **IDE variants** (SSE-IDE, WebSocket-IDE) for editor extensions;
- **SDK/in-process** — a named in-process server (~lines 108–113);
- **Claude.ai proxy** — a managed connector.

Each config carries a **scope** (`local`, `user`, `project`, `dynamic`, `enterprise`, `claudeai`, `managed`). OAuth details (`McpOAuthConfigSchema`, ~line 43) include `clientId`, `callbackPort`, and an `authServerMetadataUrl`, plus an **XAA** (cross-app-access) flag; the shared identity-provider details come from `settings.xaaIdp`.

## 3. Where configs come from and how they merge

[`src/services/mcp/config.ts`](../../../src/services/mcp/config.ts) assembles configs in `getClaudeCodeMcpConfigs()`:

- if an **enterprise** managed config exists, it is **exclusive** — only enterprise servers are used and other scopes are ignored;
- otherwise scopes merge in precedence **plugin < user < project < local** (closer/more-specific wins), and `project` scope walks the directory hierarchy merging `.mcp.json` files;
- plugin servers are namespaced `plugin:{name}:{server}` and de-duplicated against manually configured servers (manual wins), matched by a signature (`stdio:{cmd}` or `url:{...}`);
- Claude.ai connector servers that duplicate an **enabled** manual server are dropped.

A policy layer (`filterMcpServersByPolicy`) then applies allowlists (by name, command, or URL pattern with `*` wildcards) and denylists (deny wins), returning allowed vs blocked sets.

## 4. Connecting and discovering

[`src/services/mcp/client.ts`](../../../src/services/mcp/client.ts) connects via `connectToServer()`, a memoized factory keyed by a server cache key. It selects a transport (SSE, streamable HTTP, WebSocket, stdio, in-process, or Claude.ai proxy) and creates an MCP SDK `Client` that declares `roots` and `elicitation` capabilities.

Once connected it discovers:

- **tools** (`fetchToolsForClient`, LRU-cached) via `tools/list`;
- **resources** (`fetchResourcesForClient`) via `resources/list`;
- **prompts** via `prompts/list`;
- **commands** via `commands/list`.

A connection is represented by a state: connected, failed, needs-auth, pending (reconnecting), or disabled.

## 5. How an MCP tool becomes a usable tool

Each discovered MCP tool is wrapped into a `Tool` object (the base is [`MCPTool.ts`](../../../src/tools/MCPTool/MCPTool.ts), populated at runtime by the client):

- **name**: `mcp__{server}__{tool}` (each part normalized so only `[A-Za-z0-9_-]` remain), unless an SDK no-prefix mode is active;
- **`mcpInfo`**: `{ serverName, toolName }` for permission matching;
- **`isMcp: true`** and an `inputJSONSchema` taken directly from the server;
- **`checkPermissions()`** returns `behavior: 'passthrough'` — the central permission system decides, and it can suggest an `mcp__server__tool` allow rule saved to local settings;
- **`call()`** invokes the server and handles URL elicitation (see below).

A known limitation: a server name containing double underscores can be parsed incorrectly, because `__` is the delimiter.

## 6. Resources and auth tools

- [`ListMcpResourcesTool`](../../../src/tools/ListMcpResourcesTool/ListMcpResourcesTool.ts) lists resources (optionally filtered by server) from the cached discovery.
- [`ReadMcpResourceTool`](../../../src/tools/ReadMcpResourceTool/ReadMcpResourceTool.ts) reads one `server` + `uri` via `resources/read`; binary blobs are persisted to a file and returned as a path.
- [`McpAuthTool`](../../../src/tools/McpAuthTool/McpAuthTool.ts) triggers OAuth for servers in the needs-auth state.

## 7. Elicitation (interactive requests from a server)

An MCP tool can ask for more input mid-call. When a tool returns error code **-32042** with an `elicitations` array, the client runs a retry loop (up to a few attempts): it runs elicitation hooks, either resolves programmatically or shows the request to the user via `handleElicitation()`, polls for completion, submits the result with `elicitations/complete`, and retries the tool call. This is how a server can, for example, ask you to authorize something or open a URL.

## 8. MCP error handling

The client defines named errors so failures are precise and telemetry-safe:

- `McpAuthError` for OAuth failures (carries the server name);
- `McpSessionExpiredError` when it sees HTTP 404 + JSON-RPC `-32001`;
- a telemetry-safe tool-call error wrapper whose long name asserts it contains no code or file paths.

## 9. LSP integration

Language-server support lives in [`src/services/lsp/`](../../../src/services/lsp/). `initializeLspServerManager()` ([`manager.ts`](../../../src/services/lsp/manager.ts)) creates a singleton via `createLSPServerManager()` ([`LSPServerManager.ts`](../../../src/services/lsp/LSPServerManager.ts)) and starts async initialization in the background; `isLspConnected()` reports whether at least one server is healthy.

The manager keeps three maps in a closure: `servers` (name → instance), `extensionMap` (file extension → server names), and `openedFiles` (file URI → server). It exposes `initialize`, `shutdown`, `getServerForFile`, `ensureServerStarted`, `sendRequest`, and the document-sync operations `openFile`/`changeFile`/`saveFile`/`closeFile` (which send `didOpen`/`didChange`/`didSave`/`didClose`). It answers `workspace/configuration` requests with null items to satisfy the protocol without exposing config.

## 10. The LSP tool

[`LSPTool.ts`](../../../src/tools/LSPTool/LSPTool.ts) exposes language operations to the model: `goToDefinition`, `findReferences`, `hover`, `documentSymbol`, `workspaceSymbol`, `goToImplementation`, and call-hierarchy operations. Its input is `{ operation, filePath, line, character }`. It is enabled only when `isLspConnected()` is true, skips UNC paths (to avoid credential leaks), and caps file size (~10 MB). A request routes through `ensureServerStarted(filePath)` and the extension map to the right server (for example `textDocument/definition`).

Note the LSP **tool** is additionally gated by `ENABLE_LSP_TOOL` in the main tool list (see [08](08-complete-tool-catalog.md)); the LSP **service** can still run for other internal uses.

## 11. What this means for you as a user

- An MCP server is an installed integration: it can add powerful tools and can read/receive whatever you send to it. Trust it like any extension, restrict it with permission rules, and remember its tools are dynamic.
- MCP tools always go through the central permission system (passthrough), so you still approve their actions; a saved allow rule applies to that `mcp__server__tool`.
- An MCP elicitation is the server asking you for something (often authorization) — read it before responding.
- LSP is read-only code intelligence; it improves navigation/understanding and does not modify files.

## 12. Source reference (line-level)

| Concern | Symbol / constant | Location |
|---|---|---|
| Config union + scopes + OAuth/XAA | `McpServerConfig`, `ConfigScope`, `McpOAuthConfigSchema` | [`services/mcp/types.ts`](../../../src/services/mcp/types.ts):10–169 |
| Config assembly + policy | `getClaudeCodeMcpConfigs`, `filterMcpServersByPolicy` | [`services/mcp/config.ts`](../../../src/services/mcp/config.ts) |
| Connect + discover | `connectToServer`, `fetchToolsForClient`, `fetchResourcesForClient` | [`services/mcp/client.ts`](../../../src/services/mcp/client.ts) |
| Tool wrapping + passthrough | `MCPTool`, `mcp__server__tool`, `checkPermissions` | [`tools/MCPTool/MCPTool.ts`](../../../src/tools/MCPTool/MCPTool.ts) |
| Resource tools | `ListMcpResourcesTool`, `ReadMcpResourceTool` | [`tools/ListMcpResourcesTool/`](../../../src/tools/ListMcpResourcesTool/), [`tools/ReadMcpResourceTool/`](../../../src/tools/ReadMcpResourceTool/) |
| Elicitation | error `-32042`, retry loop | [`services/mcp/client.ts`](../../../src/services/mcp/client.ts) |
| MCP errors | `McpAuthError`, `McpSessionExpiredError` | [`services/mcp/client.ts`](../../../src/services/mcp/client.ts) |
| LSP manager | `initializeLspServerManager`, `createLSPServerManager`, `isLspConnected` | [`services/lsp/manager.ts`](../../../src/services/lsp/manager.ts), [`services/lsp/LSPServerManager.ts`](../../../src/services/lsp/LSPServerManager.ts) |
| LSP tool | `LSPTool` operations, `ENABLE_LSP_TOOL` gate | [`tools/LSPTool/LSPTool.ts`](../../../src/tools/LSPTool/LSPTool.ts) |

Line numbers are approximate; symbol names are the durable anchors.

## 13. Real vs stub

The MCP client/config/types, the MCP resource/auth tools, and the LSP manager and tool are real source in this checkout (with one stub inside `services/lsp`). Some behavior is feature-gated (for example MCP-provided skills and the LSP tool’s `ENABLE_LSP_TOOL` flag).
