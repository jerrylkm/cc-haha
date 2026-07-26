# 16 — Computer Use and browser control

This chapter explains how the agent can control a computer’s screen — take screenshots, move the mouse, click, type, scroll — and control a browser. It analyzes the real source so the coverage is self-contained.

Relevant source: [`src/vendor/computer-use-mcp/`](../../../src/vendor/computer-use-mcp/), [`src/native-ts/`](../../../src/native-ts/), [`src/utils/computerUse/`](../../../src/utils/computerUse/), [`src/utils/claudeInChrome/`](../../../src/utils/claudeInChrome/), and the Python helpers in [`runtime/`](../../../runtime/).

Computer Use is powerful and risky: it lets an AI operate your real desktop. It is gated behind explicit settings and permissions, described below.

## 1. What Computer Use is

Computer Use exposes desktop control as MCP tools ([`toolCalls.ts`](../../../src/vendor/computer-use-mcp/toolCalls.ts)). Actions include:

- `screenshot` (optionally cropped/filtered);
- `left_click`, `right_click`, `double_click`, drag, `mouse_move`, `mouse_down`/`mouse_up`;
- `type` (character by character), `key` chords like `cmd+shift+q`, `hold_key`;
- `scroll`, display switching, clipboard read/write, opening apps;
- `computer_batch` to run a sequence.

Coordinates are either image pixels or a normalized 0–100 scale, frozen at server start. `scaleCoord()` ([`toolCalls.ts`](../../../src/vendor/computer-use-mcp/toolCalls.ts):~206–290) handles both: in `normalized_0_100` mode it computes `(x / 100) * display.width` (~line 228), and in `pixels` mode it maps image-space coordinates using the geometry captured **at screenshot time** (~line 236), so a click lands where the model actually saw the target. A pixels-mode coordinate with no prior screenshot is rejected (~line 257).

## 2. The four mandatory gates

Every tool call passes through `handleToolCall()` ([`toolCalls.ts`](../../../src/vendor/computer-use-mcp/toolCalls.ts):3460) which enforces, in order (documented in the file header, ~lines 2–35):

1. **Kill switch** — `adapter.isDisabled()`; a Settings toggle can disable Computer Use entirely.
2. **OS permission (TCC on macOS)** — `adapter.ensureOsPermissions()`; Accessibility and Screen Recording permissions must be granted (a `request_access` flow is exempt so it can ask).
3. **Global lock** — `acquireCuLock()` (~line 3582); at most one session may use Computer Use at a time (file lock on the CLI, in-memory elsewhere).
4. **Per-tool handlers** — the specific action, with more checks below.

Input actions add further checks: hiding non-allowlisted apps and defocusing Claude before acting, verifying the frontmost app is allowed (so it cannot type into its own window), and a pixel-staleness check comparing a small patch at the click location to the last screenshot.

## 3. Tiered per-app permissions

Apps are classified into tiers by `categorizeApp()` in [`deniedApps.ts`](../../../src/vendor/computer-use-mcp/deniedApps.ts) (browser at ~line 310, terminal at ~line 311, using `BROWSER_BUNDLE_IDS` ~line 55 and `TERMINAL_BUNDLE_IDS` ~line 102):

| Tier | Allowed | Typical apps |
|---|---|---|
| `read` | screenshot only, no interaction | browsers, trading apps |
| `click` | left-click and scroll, no typing/modifiers/right-click/drag | terminals, IDEs |
| `full` | click, type, keys, modifiers, drag | general apps (default) |

`tierSatisfies` blocks a mismatched action, and the error explicitly forbids AppleScript/`osascript` workarounds. Sentinel apps ([`sentinelApps.ts`](../../../src/vendor/computer-use-mcp/sentinelApps.ts)) — shells, Finder, System Settings — are flagged as extra-sensitive. Grant flags for clipboard read/write and system key combos are opt-in separately.

A specific mitigation: for a `click`-tier terminal, the agent cannot `type`, but a UI paste button is clickable. To prevent injecting a dangerous clipboard command, the session stashes and repeatedly clears the clipboard while in click tier (`session.cuClipboardStash`, referenced ~line 361 of [`toolCalls.ts`](../../../src/vendor/computer-use-mcp/toolCalls.ts)) and restores it on exit.

## 4. Dynamic access requests

`request_access` ([`toolCalls.ts`](../../../src/vendor/computer-use-mcp/toolCalls.ts)) resolves requested app names against installed apps, checks tiers and deny lists, splits them into needs-dialog / already-granted / policy-denied, and fires a permission callback so the host UI can prompt the user. Grants are merged into session state.

## 5. The native bridge

The TypeScript side talks to a Python helper through [`src/utils/computerUse/pythonBridge.ts`](../../../src/utils/computerUse/pythonBridge.ts). On first use it prepares `~/.claude/.runtime/` (`runtimeStateRoot`, ~line 17), copies the helper and `requirements.txt` from [`runtime/`](../../../runtime/), creates a virtual environment at `~/.claude/.runtime/venv` (`venvRoot`, ~line 18), installs dependencies, and caches a digest at `requirements.sha256` (~line 19) to skip re-installs. The helper file is `win_helper.py` on Windows and `mac_helper.py` otherwise (~line 25). Each action runs the helper as a subprocess with a JSON payload and reads a JSON result.

- macOS: [`runtime/mac_helper.py`](../../../runtime/mac_helper.py) uses AppKit/Quartz/pyautogui/mss/PIL for screenshots, input synthesis, frontmost-app detection, app hiding, and permission checks.
- Windows: [`runtime/win_helper.py`](../../../runtime/win_helper.py) uses win32/psutil/screeninfo/pyautogui/mss, with registry lookups for installed apps and key-name remapping (`command`→`win`, `option`→`alt`).

[`src/native-ts/`](../../../src/native-ts/) provides TypeScript enums/bindings used alongside this bridge.

## 6. The MCP server and CLI wiring

[`mcpServer.ts`](../../../src/vendor/computer-use-mcp/) builds the Computer Use MCP server. `bindSessionContext()` returns a dispatcher that holds the last screenshot and enforces the lock gate before `handleToolCall()`. The host implements a session-context interface (allowed apps, grant flags, selected display, lock primitives, permission callbacks, OS notifications). The CLI wires this via [`src/utils/computerUse/`](../../../src/utils/computerUse/), timeboxing app enumeration so listing installed apps cannot hang startup. Tools are named `mcp__computer-use__*`, which the backend recognizes to add a system-prompt hint.

## 7. Browser control (Claude in Chrome)

[`src/utils/claudeInChrome/`](../../../src/utils/claudeInChrome/) integrates a Chrome extension for browser control. Enablement is off by default for non-interactive sessions and can be turned on by flag (`--claude-in-chrome`), environment variable, or saved config; auto-enable is limited to internal users with the extension installed. It registers a native-messaging host manifest and connects either through a WebSocket bridge (internal users) or native messaging. Its permission modes include ask-per-action, follow-the-plan, and an insecure skip-all mode for development. A dedicated system-prompt section ([`prompt.ts`](../../../src/utils/claudeInChrome/prompt.ts)) tells the model how to use the browser tools.

## 8. What this means for you as a user

- Computer Use can operate real applications; treat authorizing it like handing over mouse and keyboard.
- Nothing works until you enable it and grant OS-level Accessibility/Screen Recording permission.
- Per-app tiers limit what can happen in sensitive apps (browsers are screenshot-only; terminals cannot be typed into), and the clipboard guard reduces paste-injection risk — but a `full`-tier app is broadly controllable.
- Only one session can drive Computer Use at a time.
- Browser control is a separate, off-by-default integration with its own permission prompts.

## 9. Source reference (line-level)

| Concern | Symbol / constant | Location |
|---|---|---|
| Gate sequence | `handleToolCall` (kill switch → TCC → lock → handlers) | [`toolCalls.ts`](../../../src/vendor/computer-use-mcp/toolCalls.ts):3460 (header ~2–35) |
| Global lock | `acquireCuLock` | [`toolCalls.ts`](../../../src/vendor/computer-use-mcp/toolCalls.ts):~3582 |
| Coordinate scaling | `scaleCoord` (`normalized_0_100` / `pixels`) | [`toolCalls.ts`](../../../src/vendor/computer-use-mcp/toolCalls.ts):~206–290 |
| Clipboard guard | `session.cuClipboardStash` | [`toolCalls.ts`](../../../src/vendor/computer-use-mcp/toolCalls.ts):~361 |
| App tiers | `categorizeApp`, `BROWSER_BUNDLE_IDS`, `TERMINAL_BUNDLE_IDS` | [`deniedApps.ts`](../../../src/vendor/computer-use-mcp/deniedApps.ts):55,102,310 |
| Sentinel apps | shell/filesystem/system-settings sets | [`sentinelApps.ts`](../../../src/vendor/computer-use-mcp/sentinelApps.ts) |
| Python bridge | `runtimeStateRoot`, `venvRoot`, `requirements.sha256`, `helperFileName` | [`pythonBridge.ts`](../../../src/utils/computerUse/pythonBridge.ts):17,18,19,25 |
| macOS helper | `take_screenshot`, `click`, `key`, `prepare_for_action` | [`runtime/mac_helper.py`](../../../runtime/mac_helper.py) |
| Windows helper | registry app lookup, key remapping | [`runtime/win_helper.py`](../../../runtime/win_helper.py) |
| Browser control | setup/enablement, permission modes | [`src/utils/claudeInChrome/`](../../../src/utils/claudeInChrome/) |

Line numbers are approximate; symbol names are the durable anchors.

## 10. Real vs stub

The `computer-use-mcp` vendor code, `native-ts`, `utils/computerUse`, `utils/claudeInChrome`, and the Python helpers in `runtime/` are real source in this checkout. The related `WebBrowserTool`/`TerminalCaptureTool` entries are feature-gated and appear as stubs in the [tool catalog](08-complete-tool-catalog.md).
