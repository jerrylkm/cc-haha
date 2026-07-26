# 17 — Remote access and IM bridge

This chapter explains how you can control a Claude Code session from somewhere other than the local terminal: a web client, a phone browser (H5), or a messaging app (Telegram, Feishu, WeChat, DingTalk, WhatsApp). It analyzes the real source so the coverage is self-contained.

Relevant source: [`src/bridge/`](../../../src/bridge/), [`src/remote/`](../../../src/remote/), [`src/upstreamproxy/`](../../../src/upstreamproxy/), the WebSocket server under [`src/server/ws/`](../../../src/server/ws/), the H5 service under [`src/server/`](../../../src/server/), and the messaging adapters in [`adapters/`](../../../adapters/).

## 1. The bridge: connecting a session to a remote channel

The bridge ([`src/bridge/`](../../../src/bridge/)) is a relay that connects a local CLI session to a remote control channel. Its HTTP client is built by `createBridgeApiClient()` ([`bridgeApi.ts`](../../../src/bridge/bridgeApi.ts):68) and exposes `registerBridgeEnvironment()` (~line 142) to register the machine as a bridge environment, `pollForWork()` (~line 199) to long-poll for incoming sessions, `acknowledgeWork()` (~line 249), `heartbeatWork()` (~line 387) to keep a lease, and `deregisterEnvironment()` (~line 301) on shutdown, plus permission-response events.

Authentication requires a claude.ai subscription. If no access token is present, the bridge throws with `BRIDGE_LOGIN_INSTRUCTION` ([`src/bridge/types.ts`](../../../src/bridge/types.ts):5) telling the user to `/login`. An elevated “trusted device” token can be required behind a feature gate.

Inbound and outbound messages are relayed by `handleIngressMessage()` ([`bridgeMessaging.ts`](../../../src/bridge/bridgeMessaging.ts):132): inbound user messages, control requests (permission prompts, interrupts, model changes), and control responses are parsed, de-duplicated (an echo filter), and routed to the REPL; outbound messages are converted to SDK messages and sent over the transport (HTTP/SSE or WebSocket). Inbound attachments are fetched and written under `~/.claude/uploads/…` and referenced in the message.

## 2. Message safety for remote input

Remote input is deliberately constrained so a message from another channel cannot silently run a privileged local command:

- Only user/assistant messages and `local_command` system messages are eligible to cross the bridge; internal “virtual” messages, tool results, progress, and errors are filtered out (`isEligibleBridgeMessage`, [`bridgeMessaging.ts`](../../../src/bridge/bridgeMessaging.ts):77, with the `local_command` allowance at ~line 86).
- Mutating control requests (set model, set permission mode, etc.) are gated; in outbound-only mode they reply with an error rather than a false success.
- This complements the slash-command restrictions described in [06 — Terminal UI and commands](06-terminal-ui-commands.md): remote/bridge messages can disable or limit slash-command interpretation.

## 3. Remote session manager

[`RemoteSessionManager.ts`](../../../src/remote/RemoteSessionManager.ts) (`class RemoteSessionManager`, ~line 95) manages a single remote session: a WebSocket subscription for inbound messages plus HTTP for outbound, with reconnect handling. It relays SDK control requests — permission requests (`can_use_tool`, ~line 192, tracked in a `pendingPermissionRequests` map at ~line 97), cancels (`control_cancel_request`, ~line 160), and interrupts — and `respondToPermissionRequest()` (~line 247) maps a remote approver’s decision back to the correct tool call. This is the transport behind remote-isolation agents in [14 — Multi-agent, subagents, and teams](14-multi-agent-and-teams.md).

## 4. Upstream proxy relay

[`upstreamproxy/relay.ts`](../../../src/upstreamproxy/relay.ts) tunnels HTTP `CONNECT` requests over a WebSocket to the CCR server, which proxies the connection and injects organization-configured credentials. It runs a small local TCP server, parses the `CONNECT host:port` header, opens a WebSocket, and pumps bytes both ways in bounded chunks (`MAX_CHUNK_BYTES` = 512 KiB, ~line 51) with a keepalive (`PING_INTERVAL_MS` = 30,000 ms, ~line 54) sent as an empty chunk (`encodeChunk`, ~line 66). It has Node and Bun code paths and guards against malformed headers and double-close. WebSocket is used instead of raw CONNECT because CCR ingress is GKE L7 with path-prefix routing (~lines 10–12). This lets tools reach approved upstreams through a managed proxy without embedding credentials locally.

## 5. H5 remote access (phone/browser)

H5 access lets a phone or another browser connect to the local server. The service ([`src/server/`](../../../src/server/) H5 modules) stores settings including an enable flag, a token (with a short preview), allowed CORS origins, a public base URL, an optional fixed port, and a disconnect grace period.

Access is classified by `classifyH5Request()` in [`h5AccessPolicy.ts`](../../../src/server/h5AccessPolicy.ts) into one of three kinds (`H5RequestKind`, line 1):

- **local-trusted** — loopback connections need no token (~lines 140);
- **h5-browser** — remote browsers must present the token and pass CORS plus a Fetch-Metadata (CSRF) check (~lines 132/143);
- **internal-sdk** — server-to-server with its own auth (~line 136).

The API ([`server/api/h5-access.ts`](../../../src/server/api/h5-access.ts)) supports getting/updating settings and enabling, disabling, regenerating (rotating, ~line 79), and verifying the token. The desktop app typically shows a QR code so a phone can scan and connect, and the grace period keeps the session briefly alive after the phone disconnects.

## 6. IM adapters

The messaging adapters in [`adapters/`](../../../adapters/) let you drive a session from chat apps. A shared layer ([`adapters/common/`](../../../adapters/common/)) provides a WebSocket bridge to the local server (`/ws/:sessionId`), a persistent chat↔session mapping, message buffering/dedup, attachment handling, and permission parsing. Permissions can be answered with inline buttons or slash-style shortcuts (`/allow`, `/always`, `/deny`, with numeric and multilingual variants), where “always” persists the permission for the session.

Each platform adapter (Telegram, Feishu, WeChat, DingTalk, WhatsApp) wraps its own SDK, formats streaming/thinking output to that platform’s limits, and uploads attachments to the local server. These adapters live at the repository root, outside `src/`, but connect through the same server/bridge boundary analyzed here.

## 7. What this means for you as a user

- Remote control requires signing in; the bridge refuses without a claude.ai account.
- A remote or chat message is constrained so it cannot silently become a dangerous local command, but you are still approving real actions — review permission prompts the same way you would locally.
- H5 access is protected by a token and origin/CSRF checks; treat the token like a password and rotate it if exposed. Loopback connections are trusted without a token.
- Every extra channel (web, phone, each IM platform, the upstream proxy) is an additional trust and network boundary; enable only the ones you need.
- Attachments sent from remote channels are written to your local disk under `~/.claude/`.

## 8. Source reference (line-level)

| Concern | Symbol / constant | Location |
|---|---|---|
| Bridge API client | `createBridgeApiClient`, `registerBridgeEnvironment`, `pollForWork`, `acknowledgeWork`, `heartbeatWork`, `deregisterEnvironment` | [`bridgeApi.ts`](../../../src/bridge/bridgeApi.ts):68,142,199,249,387,301 |
| Login requirement | `BRIDGE_LOGIN_INSTRUCTION` | [`types.ts`](../../../src/bridge/types.ts):5 |
| Message eligibility | `isEligibleBridgeMessage`, `handleIngressMessage` | [`bridgeMessaging.ts`](../../../src/bridge/bridgeMessaging.ts):77,86,132 |
| Upstream relay | `MAX_CHUNK_BYTES` (512 KiB), `PING_INTERVAL_MS` (30 s), `encodeChunk` | [`relay.ts`](../../../src/upstreamproxy/relay.ts):51,54,66 |
| Remote session | `RemoteSessionManager`, `pendingPermissionRequests`, `can_use_tool`, `control_cancel_request`, `respondToPermissionRequest` | [`RemoteSessionManager.ts`](../../../src/remote/RemoteSessionManager.ts):95,97,192,160,247 |
| H5 classification | `H5RequestKind`, `classifyH5Request` | [`h5AccessPolicy.ts`](../../../src/server/h5AccessPolicy.ts):1,132–143 |
| H5 token API | enable/disable/regenerate/verify | [`server/api/h5-access.ts`](../../../src/server/api/h5-access.ts):79 |
| IM shared layer | WebSocket bridge, permission parsing | [`adapters/common/`](../../../adapters/common/) |

Line numbers are approximate; symbol names and constants are the durable anchors.

## 9. Real vs stub

The bridge, remote session manager, upstream proxy, WebSocket server, H5 service, and IM adapters are real source in this checkout (with a couple of stubs inside `src/bridge/`). Some behavior is feature-gated (for example elevated auth enforcement and multi-session modes).
