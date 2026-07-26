# 18 — Local server and provider proxy

This chapter explains the local HTTP/WebSocket server that sits between graphical/remote clients and the CLI, and the **provider proxy** that lets the Anthropic-shaped CLI talk to non-Anthropic model providers. It analyzes the real source so the coverage is self-contained; it expands the summary in [05 §11](05-context-persistence-services.md).

Relevant source: [`src/server/index.ts`](../../../src/server/index.ts), [`src/server/ws/`](../../../src/server/ws/), [`src/server/api/`](../../../src/server/api/), [`src/server/services/`](../../../src/server/services/), [`src/server/proxy/`](../../../src/server/proxy/), [`src/server/middleware/`](../../../src/server/middleware/), [`src/server/config/`](../../../src/server/config/), and [`src/server/types/`](../../../src/server/types/).

## 1. Why there is a local server

The terminal CLI can run on its own. But the desktop app, phone (H5), and IM adapters are separate programs that need a stable way to drive a session, watch its output, and answer permission prompts. The local server is that hub: it accepts client connections, launches and supervises CLI subprocesses, relays messages both ways, and can proxy model requests to different providers.

## 2. Startup

`startServer()` ([`index.ts`](../../../src/server/index.ts):205) calls Bun’s `Bun.serve()` (~line 239) with an HTTP `fetch` handler and a `websocket` handler (`handleWebSocket`, ~line 576). Defaults are port **3456** and host **127.0.0.1** (loopback), overridable by `--port`/`SERVER_PORT` and `--host`/`SERVER_HOST` (~lines 67–68). Because the default host is loopback, the server is not exposed to the network unless configured.

## 3. What it serves (request routing)

The `fetch` handler dispatches by path:

| Path | Purpose |
|---|---|
| `/health` | Startup probe (used by the desktop host) |
| `/ws/:sessionId` | Client WebSocket for a UI/remote controller |
| `/sdk/:sessionId` | Internal WebSocket the CLI subprocess connects back on (randomized token) |
| `/callback`, `/callback/openai` | OAuth redirect handlers |
| `/api/*` | REST API dispatcher |
| `/proxy/v1/messages`, `/proxy/providers/:id/v1/messages` | Provider proxy |
| `/preview-fs/:sessionId/*`, `/local-file/*` | Sandboxed file access for previews |
| static H5 assets | Bootstrap shell served after other checks (so a phone can read the QR token) |

## 4. The two-socket model

A running session uses **two** WebSockets:

- **client ↔ server** at `/ws/:sessionId` — the desktop/phone/IM controller;
- **CLI ↔ server** at `/sdk/:sessionId` — the CLI subprocess’s stream-json I/O, authorized by a random token.

This split lets several client viewers watch the same CLI session, and it keeps the CLI’s I/O channel internal. The CLI is the source of truth: it emits assistant output and permission requests; the server queues and broadcasts them; a client answers; the server forwards the answer to the CLI.

## 5. Session lifecycle

The WebSocket handler ([`ws/handler.ts`](../../../src/server/ws/handler.ts)) and conversation service ([`services/conversationService.ts`](../../../src/server/services/conversationService.ts)) manage the lifecycle:

1. a client connects to `/ws/:sessionId`;
2. on the first user message, `conversationService` spawns the CLI (`Bun.spawn`) and waits for it to connect back on `/sdk/:sessionId` (readiness polled every `CONTROL_READY_POLL_MS` = 50 ms);
3. user messages are written to the CLI; CLI stdout JSON lines are parsed and broadcast;
4. a permission request from the CLI is stored and broadcast; a client’s response is forwarded back and the CLI continues;
5. when the last client disconnects, a cleanup timer is armed for `PENDING_PERMISSION_DISCONNECT_CLEANUP_MS` = 30 minutes (~line 93); reconnecting cancels it;
6. on cleanup, the CLI is sent SIGTERM, then SIGKILL after a ~6 s grace.

If active work is running at disconnect, a disconnect watcher defers the stop until the work completes. Sessions can also be **prewarmed** (started without a user message) with a ~5-minute idle timeout.

The handler tracks a lot of per-session state: active turns, background task IDs, stop requests, terminal chat states, title-generation state, and cached slash commands. `getSessionChatActivityState()` (~line 242) collapses these into `idle` / `waiting` / `running` for the UI, with explicit stop and pending-permission taking priority.

## 6. REST API surface

[`src/server/api/`](../../../src/server/api/) (32 real files) exposes resources the desktop uses, including sessions, providers, MCP configuration, H5 access, and diagnostics. Requests pass through the API dispatcher after the auth gate (section 9). Provider and H5 endpoints are the security-sensitive ones; managed/enterprise-owned definitions are read-only to clients.

## 7. Session services and the local index

Under [`src/server/services/`](../../../src/server/services/) (79 real files):

- **`sessionService`** aggregates session metadata and usage, and applies context-window overrides from the environment.
- **`conversationService`** owns CLI subprocesses: spawning, stdin/stdout, permission relay, and **crash diagnostics** — it keeps the last ~80 stdout and ~80 stderr lines and the last ~40 SDK messages (bounded to 64 KB each, 512 KB total). Exit codes 0/null/143/137 are classified `info` (clean); anything else is `error`.
- **`localIndex`** is a SQLite index of sessions/transcripts for fast listing and search. Its schema is versioned: `LOCAL_INDEX_SCHEMA_VERSION` = 3 ([`localIndex/migrations.ts`](../../../src/server/services/localIndex/migrations.ts):3), with a marker for unsupported (newer) versions so a downgrade fails safe. The session-list projection is given first cold-start I/O priority so the UI list appears quickly; the search index warms in the background.
- **`persistentStorageMigrations`** versions the provider configuration (`CURRENT_PROVIDER_INDEX_SCHEMA_VERSION` = 2, [`persistentStorageMigrations.ts`](../../../src/server/services/persistentStorageMigrations.ts):10).

The JSONL transcript remains the durable record; the SQLite index is a rebuildable accelerator (see [05 §6–§7](05-context-persistence-services.md)).

## 8. The provider proxy

The proxy is what lets a CLI that speaks the Anthropic Messages API use OpenAI-compatible or other providers. Provider records ([`server/types/provider.ts`](../../../src/server/types/provider.ts)) carry an `apiFormat` of `anthropic` (passthrough, no proxy), `openai_chat` (Chat Completions), or `openai_responses` (Responses API) (~lines 24–26), plus built-in IDs like `claude-official`, `openai-official`, and `grok-official` (~lines 10–12).

When the format is not native Anthropic, the proxy handler ([`server/proxy/handler.ts`](../../../src/server/proxy/handler.ts)) transforms the request and response:

- **Request** ([`transform/anthropicToOpenaiChat.ts`](../../../src/server/proxy/transform/anthropicToOpenaiChat.ts)): converts messages, tool definitions, and reasoning-effort into the OpenAI shape; a vision-disabled endpoint gets a placeholder for images (`OMITTED_IMAGE_TEXT`).
- **Response** ([`transform/openaiChatToAnthropic.ts`](../../../src/server/proxy/transform/openaiChatToAnthropic.ts)): converts text, tool calls (parsing arguments via [`transform/toolArguments.ts`](../../../src/server/proxy/transform/toolArguments.ts)), and several provider-specific reasoning formats (`reasoning_content`, `reasoning`, `thinking_blocks`) back into Anthropic `thinking`/`text`/`tool_use` blocks.
- **Streaming**: a state machine parses the upstream SSE stream and re-emits ordered Anthropic events (`message_start`, `content_block_*`, `message_delta`, `message_stop`). An idle-timeout wrapper cancels a stalled stream.
- **Usage** ([`transform/usage.ts`](../../../src/server/proxy/transform/usage.ts)): maps upstream token counts (including cached tokens) into Anthropic’s `input_tokens`/`output_tokens`/cache fields so cost tracking stays consistent.
- **Effort/billing** ([`transform/effort.ts`](../../../src/server/proxy/transform/effort.ts), [`transform/billingHeader.ts`](../../../src/server/proxy/transform/billingHeader.ts)): normalizes reasoning-effort levels and strips an internal billing header before forwarding so upstream prompt caching is preserved.

The Responses-API pair ([`anthropicToOpenaiResponses.ts`](../../../src/server/proxy/transform/anthropicToOpenaiResponses.ts) / [`openaiResponsesToAnthropic.ts`](../../../src/server/proxy/transform/openaiResponsesToAnthropic.ts)) does the equivalent for that newer format.

## 9. Middleware and access control

[`src/server/middleware/`](../../../src/server/middleware/) provides auth and CORS. Access is layered in the `fetch` handler:

- loopback requests are trusted;
- H5/remote browser requests must pass the H5 token + origin/CSRF checks (see [17 §5](17-remote-access-and-bridge.md));
- the internal `/sdk/` socket is authorized by a per-session random token;
- an optional `--auth-required`/`SERVER_AUTH_REQUIRED` mode requires an API key;
- **pet** (companion) clients get a restricted, read-only, verb-limited token scoped to specific sessions.

## 10. What this means for you as a user

- The desktop app, phone, and IM control all funnel through this one local server; it is the trust boundary for those clients.
- By default it listens only on loopback (127.0.0.1:3456); exposing it more widely is a deliberate configuration choice with security consequences.
- Using a non-Anthropic provider means your prompts pass through the local transform proxy and then to that provider — a different privacy/billing domain than Anthropic.
- The server keeps bounded crash diagnostics (recent CLI output and SDK messages) to help debugging; these are local.
- A closed UI does not immediately kill the session; there is a 30-minute grace period so you can reconnect.

## 11. Source reference (line-level)

| Concern | Symbol / constant | Location |
|---|---|---|
| Server start | `startServer`, `Bun.serve`, `handleWebSocket` | [`index.ts`](../../../src/server/index.ts):205,239,576 |
| Port/host defaults | `3456`, `127.0.0.1` | [`index.ts`](../../../src/server/index.ts):67,68 |
| Session cleanup grace | `PENDING_PERMISSION_DISCONNECT_CLEANUP_MS` (30 min) | [`ws/handler.ts`](../../../src/server/ws/handler.ts):93 |
| Activity state | `getSessionChatActivityState` | [`ws/handler.ts`](../../../src/server/ws/handler.ts):~242 |
| CLI supervision + diagnostics | `conversationService`, `CONTROL_READY_POLL_MS` (50 ms) | [`services/conversationService.ts`](../../../src/server/services/conversationService.ts):68 |
| Local index schema | `LOCAL_INDEX_SCHEMA_VERSION` (3) | [`services/localIndex/migrations.ts`](../../../src/server/services/localIndex/migrations.ts):3 |
| Provider config schema | `CURRENT_PROVIDER_INDEX_SCHEMA_VERSION` (2) | [`services/persistentStorageMigrations.ts`](../../../src/server/services/persistentStorageMigrations.ts):10 |
| Provider records | `apiFormat`, official provider IDs | [`types/provider.ts`](../../../src/server/types/provider.ts):10–26 |
| Proxy transforms | anthropic↔openai chat/responses, usage, effort, billing | [`server/proxy/transform/`](../../../src/server/proxy/transform/) |

Line numbers are approximate; symbol names and constants are the durable anchors.

## 12. Real vs stub

The server entry, WebSocket handler, API, session/conversation/index services, provider proxy transforms, middleware, and config are real source in this checkout. A few backend files (for example under [`server/backends/`](../../../src/server/backends/)) are generated stubs; the [subsystem map](13-src-subsystem-map.md) records the per-area counts.
