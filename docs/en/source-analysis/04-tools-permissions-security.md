# 04 — Tools, permissions, and security

## What matters most to a user

- A tool is an action, not just an explanation.
- The visible tool list changes by mode, settings, provider, platform, permissions, extensions, and agent type.
- Approval decisions are based on the proposed input, such as a command, file path, or website.
- Sandboxing and permissions reduce risk but cannot guarantee that an approved action is correct or reversible.
- Plugins, hooks, skills, shell commands, and MCP servers can extend behavior beyond the built-in list.

For safe-use questions, read [00 — Beginner guide](00-beginner-guide.md) and [10 — User FAQ](10-user-faq.md) first.

## 1. Tool registration

[`src/tools.ts`](../../../src/tools.ts) assembles the available tool list. Some tools are statically present, some are feature-gated, and some team-related tools are lazy-loaded to break dependency cycles.

Agent types receive subsets from [`src/constants/tools.ts`](../../../src/constants/tools.ts). Background or in-process teammates do not automatically inherit every main-thread capability.

This chapter explains the shared framework and security path. For the name, purpose, inputs, aliases, gate, and availability of every registration, see [08 — Complete tool catalog](08-complete-tool-catalog.md).

The phrase “tool list” has several meanings:

1. [`getAllBaseTools()`](../../../src/tools.ts) lists every built-in that could exist in the current build/environment.
2. [`getTools()`](../../../src/tools.ts) applies simple/REPL mode, blanket deny rules, and each tool’s `isEnabled()`.
3. [`assembleToolPool()`](../../../src/tools.ts) adds dynamic MCP tools, sorts partitions for prompt-cache stability, and deduplicates names.
4. Agent/coordinator filters remove capabilities unsuitable for that execution context.
5. The API layer may defer full schemas behind ToolSearch.

Thus a source-level tool can exist without being shown to the current model.

## 2. The execution pipeline

For each complete `tool_use` block:

```text
find tool by name or alias
  -> parse schema
  -> validateInput()
  -> global + tool-specific permission checks
  -> permission hooks / optional user dialog
  -> call()
  -> progress events
  -> model-facing result conversion
  -> persistence/truncation and UI rendering
```

Aliases in [`src/Tool.ts`](../../../src/Tool.ts) preserve renamed-tool compatibility. Deferred tools expose only search metadata until the model discovers them through ToolSearch.

## 3. Permission modes

Permission types are defined in [`src/types/permissions.ts`](../../../src/types/permissions.ts). User-visible modes include:

- `default`: rules decide; uncertain operations ask;
- `plan`: restricts action while planning;
- `acceptEdits`: pre-approves editing behavior within its rules;
- `bypassPermissions`: bypasses prompts when policy allows it;
- `dontAsk`: converts would-ask outcomes to denial;
- `auto`: classifier-assisted behavior behind a feature gate.

`bubble` is internal and used when a decision must be delegated to an outer context.

If you do not recognize a mode, the practical choice is to stay with the normal/default behavior. Bypass-style modes trade fewer interruptions for more responsibility on the user.

## 4. Rule sources and precedence

Rules may come from user, project, local, policy, command-line, command, or session scope. A rule combines:

- behavior: allow, ask, or deny;
- tool name;
- optional tool-specific content such as a Bash pattern, path, or domain.

[`src/utils/permissions/permissions.ts`](../../../src/utils/permissions/permissions.ts) checks whole-tool denials, explicit asks, tool-specific logic, sensitive-resource rules, mode transformations, and automated checks. Managed-policy-only operation can discard non-policy rule sources.

An “allow” can still become “ask” when a classifier or sensitive-path check requires review. In `dontAsk`, “ask” becomes “deny,” never silent approval.

Rules can be broad. Approving or configuring a pattern such as an entire command family may affect future calls, so read any “always allow” choice more carefully than a one-time approval.

## 5. Bash defenses

### Parsing

[`src/utils/bash/bashParser.ts`](../../../src/utils/bash/bashParser.ts) uses a TypeScript parser that produces tree-sitter-like nodes. It limits parse time and node count to resist pathological input and tracks UTF-8 byte offsets rather than JavaScript character offsets.

### Security checks

[`bashSecurity.ts`](../../../src/tools/BashTool/bashSecurity.ts) looks for command substitution, process substitution, malformed quoting/tokens, IFS manipulation, dangerous zsh modules, environment leakage, control characters, Unicode whitespace, and parser-desynchronization patterns.

[`bashPermissions.ts`](../../../src/tools/BashTool/bashPermissions.ts) caps the number of subcommands subjected to automatic analysis. Very large compounds fall back to asking rather than attempting an expensive or unreliable classification.

> **Full breakdown:** [11 — Bash security checks in depth](11-bash-security-checks.md) documents all twenty-three individual check IDs, the primary tree-sitter gate versus the legacy fallback, validator ordering, the misparsing rule that can force an early stop, the subcommand cap, and what each check means for a user.

### Wrapper normalization

Permission matching can peel known wrappers such as `timeout`, `time`, `nice`, `stdbuf`, `nohup`, and simple environment assignments. It repeats until stable, because wrappers can be nested. Flag/value syntax is constrained, and horizontal whitespace is used deliberately so a newline cannot be swallowed as harmless spacing.

## 6. Sandboxing

[`sandbox-adapter.ts`](../../../src/utils/sandbox/sandbox-adapter.ts) integrates `@anthropic-ai/sandbox-runtime` with project settings and policy.

The decision in [`shouldUseSandbox.ts`](../../../src/tools/BashTool/shouldUseSandbox.ts) considers:

1. whether sandboxing is enabled;
2. whether policy permits explicitly disabling it;
3. excluded-command configuration;
4. normalized command structure.

Excluded commands are a convenience control, not the security boundary. Permission checks remain authoritative.

The adapter has project-specific path semantics: double-leading-slash paths are filesystem-root absolute, while some single-leading-slash configuration paths resolve relative to a settings base.

## 7. File safety

[`src/utils/permissions/filesystem.ts`](../../../src/utils/permissions/filesystem.ts) normalizes and checks paths against the working directory, additional allowed directories, settings locations, and skill scopes.

Sensitive examples include shell profiles, Git configuration/metadata, editor settings, MCP configuration, and `.claude` state. Case-insensitive normalization prevents casing tricks on case-insensitive platforms.

Edit tools use this shared boundary instead of implementing unrelated path checks.

## 8. Hooks

Hook schemas live in [`src/types/hooks.ts`](../../../src/types/hooks.ts), with execution in [`src/utils/hooks.ts`](../../../src/utils/hooks.ts) and specialized helpers under [`src/utils/hooks/`](../../../src/utils/hooks/).

Hook events cover pre/post tool execution, failures, permission requests, session/setup, user prompt submission, notifications, file/cwd changes, worktrees, elicitation, and subagent lifecycle.

Hooks may:

- block;
- add model context;
- update tool input or MCP output;
- approve/deny a permission request;
- emit system messages.

Multiple results are aggregated. Some fields merge, while updated inputs have defined winner behavior.

## 9. User decision versus classifier race

The permission UI can race a non-blocking classifier against human interaction. [`PermissionContext.ts`](../../../src/hooks/toolPermission/PermissionContext.ts) uses resolve-once state so exactly one result wins. A late classifier cannot overwrite a user click, and a late click cannot resolve an already approved request twice.

## 10. MCP and web tools

[`MCPTool.ts`](../../../src/tools/MCPTool/MCPTool.ts) is a runtime-overridden base. The connected MCP server supplies actual name, description, schema, and call behavior. MCP tools return a permission “passthrough” so the central permission system decides.

[`WebFetchTool.ts`](../../../src/tools/WebFetchTool/WebFetchTool.ts) validates URLs, converts permission matching to `domain:<hostname>`, recognizes preapproved URLs, defers loading, and allows large bounded results.

## 11. Result size and cancellation

Each tool declares `maxResultSizeChars`. Oversized output is persisted with a preview, except tools such as Read that already self-bound and would create a Read-to-file-to-Read loop.

Each tool may declare:

- `cancel`: stop and discard on interruption;
- `block`: finish before handling new input.

`block` is the default because it is safer for writes and irreversible operations.

## 12. Security interpretation

No single check is the boundary. Safety is defense in depth:

```text
schema + parser limits
  + tool validation
  + scoped rules and managed policy
  + sensitive path/domain checks
  + hooks/classifier
  + explicit user approval
  + sandbox
  + abort/result bookkeeping
```

Bypass mode is intentionally constrained by policy and availability state; merely selecting a mode name is not enough to make it available.

Defense in depth does not mean “nothing bad can happen.” It means several independent checks must fail or be bypassed before many unsafe actions can occur. User review remains one of those layers.
