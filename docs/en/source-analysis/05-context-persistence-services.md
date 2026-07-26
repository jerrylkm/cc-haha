# 05 — Context, persistence, and services

## What matters most to a user

This chapter explains what the program remembers, what it sends to model/provider services, and how a session can be resumed.

Key points:

- the model receives assembled context, not only your newest sentence;
- conversation transcripts are normally stored locally;
- provider, MCP, web, plugin, telemetry, and remote features can communicate over the network;
- old conversation detail may be summarized to fit model limits;
- the SQLite index is for speed; the JSONL transcript is the main local session record;
- restoring a session also restores operational metadata, not only chat text.

## 1. Context assembly

[`getSystemContext()`](../../../src/context.ts) takes a memoized snapshot that includes Git/environment information. [`getUserContext()`](../../../src/context.ts) discovers project instructions such as `CLAUDE.md` content and includes the current date.

Memoization prevents repeated filesystem/Git work and avoids cycles in classifier/context generation. A consequence is that some startup facts, such as Git status, are intentionally stale later in the conversation. This is the same startup-time snapshot behavior introduced in [02 — Startup and one turn](02-startup-and-one-turn.md) (Step C, context assembly): `getSystemContext()` captures the machine/project snapshot once and reuses it, so a fact gathered at launch is not silently refreshed on every later turn.

In practical terms, if the repository changes outside the assistant during a long session, some background description captured at startup may not immediately update.

[`buildSystemInitMessage()`](../../../src/utils/messages/systemInit.ts) serializes session/tool/model/MCP/permission/command metadata for SDK consumers. Compatibility mapping preserves legacy tool names where external clients require them.

## 2. Configuration layers

The core configuration implementation is [`src/utils/config.ts`](../../../src/utils/config.ts), supplemented by settings modules under [`src/utils/settings/`](../../../src/utils/settings/).

Configuration can come from:

- defaults and bundled product behavior;
- user settings;
- project and local settings;
- managed enterprise/device policy;
- CLI flags and environment;
- session/runtime overrides.

Unknown user-owned fields must be preserved. Managed scopes can be read-only and can constrain user/project behavior.

## 3. Provider abstraction

Provider types live in [`src/server/types/provider.ts`](../../../src/server/types/provider.ts). A saved provider describes scope, endpoint/model information, API format, and an authentication strategy.

Supported request families include native Anthropic and OpenAI Chat/Responses compatibility. Transform code under [`src/server/proxy/transform/`](../../../src/server/proxy/transform/) maps requests, streaming responses, errors, thinking/reasoning data, and tool calls across protocols.

Persistent provider schema migrations live in [`persistentStorageMigrations.ts`](../../../src/server/services/persistentStorageMigrations.ts).

The selected provider is a trust boundary: prompts and included context are sent according to that provider configuration. Protocol conversion does not make a third-party provider equivalent in privacy, billing, or behavior to another provider.

## 4. API streaming and retry

[`src/services/api/claude.ts`](../../../src/services/api/claude.ts) builds and streams model requests. [`withRetry.ts`](../../../src/services/api/withRetry.ts) applies source-aware retry:

- foreground user work receives bounded retry support;
- background suggestions/titles avoid amplifying gateway failures;
- unattended behavior may use gated persistent retry with heartbeat output.

[`streamWatchdog.ts`](../../../src/services/api/streamWatchdog.ts) distinguishes waiting for the first event, waiting for useful content, and a stalled mid-stream response. It records whether any tool/side-effect boundary was crossed so retry logic can avoid duplicating effects.

[`streamAssistantCommitBuffer.ts`](../../../src/services/api/streamAssistantCommitBuffer.ts) buffers side-effect-free assistant blocks until it is safe to commit them. Failed attempts can then be discarded cleanly.

## 5. Transcript format

[`src/utils/sessionStorage.ts`](../../../src/utils/sessionStorage.ts) stores sessions as JSON Lines under the Claude configuration project area. Each line is independently parseable and includes identifiers, timestamps, working directory, entrypoint/version/Git metadata, and a typed message or metadata entry.

Besides visible chat messages, logs can contain:

- compact boundaries and summaries;
- custom/AI titles and last prompts;
- task summaries and tags;
- agent name/color/settings;
- PR links and permission modes;
- worktree state;
- content-replacement metadata;
- file-history and attribution snapshots;
- context-collapse commits.

This is why a transcript is more than a chat export.

Deleting or exporting only visible messages may not account for every related local artifact. Large tool results, history, indexes, caches, and extension storage can exist separately.

## 6. Loading and recovery

[`sessionStoragePortable.ts`](../../../src/utils/sessionStoragePortable.ts) can read small head/tail portions for fast session listing. Full resume logic in [`sessionRestore.ts`](../../../src/utils/sessionRestore.ts) reconstructs messages, todos, file history, attribution, compaction, and worktree state.

Compact boundaries allow sufficiently old pre-compact messages to be skipped. Tombstone rewrites are size-capped to avoid loading arbitrarily large logs into memory. Worktrees are restored only when their paths still exist.

## 7. Local index

The local server indexes transcripts in SQLite. Migrations in [`src/server/services/localIndex/migrations.ts`](../../../src/server/services/localIndex/migrations.ts) define:

- source transcript files and indexed sessions;
- ordinal JSONL entries;
- activity sources such as subagent sidechains;
- schema/backfill state.

The index accelerates listing/search but the JSONL transcript remains the durable conversation record.

## 8. Cost and usage

[`src/cost-tracker.ts`](../../../src/cost-tracker.ts) tracks cost, API/tool duration, changed lines, requests, and per-model input/output/cache/web-search usage.

The stored cost state includes total cost in USD, total API duration (with and without retries), total tool duration, lines added/removed, the last request duration, and a per-model usage map. Each model entry can record input tokens, output tokens, cache-read and cache-creation tokens, web-search requests, and a computed cost. This is what the UI uses to show token and cost summaries.

Two cautions matter for users:

- these numbers are the runtime’s own accounting, partly from model-reported usage; the authoritative charge comes from your provider, not this counter;
- a local command can consume compute, disk, or network independently of any model cost.

Session restoration only hydrates stored cost when the saved session ID matches (`restoreCostStateForSession`). This avoids applying the previous session’s totals to a different conversation.

## 9. MCP

MCP configuration types in [`src/services/mcp/types.ts`](../../../src/services/mcp/types.ts) support stdio, SSE, streamable HTTP, WebSocket, headers, OAuth, and cross-app-access identity settings.

[`src/services/mcp/client.ts`](../../../src/services/mcp/client.ts) handles connection, authentication, transport, session expiration, tool/resource/command discovery, and safe error wrapping. Scope rules prevent desktop clients from editing managed or enterprise-owned definitions.

Because MCP tools come from connected servers, their names and actions cannot be completely listed in a static document. Treat each MCP server as an installed integration with its own trust and data-handling implications.

## 10. LSP

[`createLSPServerManager()`](../../../src/services/lsp/LSPServerManager.ts) keeps extension-to-server routing and file-open state in a closure. It initializes/shuts down servers, opens/changes/saves/closes files, and sends typed requests. Tools use this service instead of each implementing an editor protocol.

## 11. Local server and WebSocket

[`src/server/index.ts`](../../../src/server/index.ts) starts the local HTTP/WebSocket boundary. [`src/server/ws/handler.ts`](../../../src/server/ws/handler.ts) tracks per-session commands, runtime model/provider overrides, title generation, active turns, disconnect cleanup, and permission requests.

[`conversationService.ts`](../../../src/server/services/conversationService.ts) manages CLI subprocess conversations for desktop/remote clients and retains bounded crash diagnostics. Permission requests carry tool name/input and request identity across the process boundary.

## 12. Analytics and privacy boundaries

[`src/services/analytics/index.ts`](../../../src/services/analytics/index.ts) queues events until a sink is attached. Metadata marker types require call sites to distinguish reviewed non-code data from PII-tagged data. `_PROTO_*` fields are removed before sending to sinks that must not receive privileged fields.

Errors intended for telemetry use explicitly named safe wrappers, reducing accidental inclusion of code or file paths. This is a convention enforced through types and naming rather than a magical sanitizer.

For a user, three practical points follow:

- analytics only flow when a sink is configured/attached; before that, events are queued rather than sent;
- the code separates ordinary reviewed metadata from PII-tagged metadata at the type level, so callers must consciously mark sensitive values;
- because the protection is a coding convention, a custom build, plugin, or provider integration could still send more than the core intends. Treat privacy as dependent on your configuration and installed extensions, not guaranteed by these markers alone.

## 13. Migrations

[`src/migrations/`](../../../src/migrations/) contains narrow migrations for model names/defaults, permission acceptance, update settings, MCP enablement, and remote-control settings. Persistence migrations are forward transformations: old state must remain loadable without erasing unknown fields.
