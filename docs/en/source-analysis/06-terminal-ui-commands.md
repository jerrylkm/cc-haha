# 06 — Terminal UI and commands

## What a user sees

This chapter describes the terminal version of the product: the text box, conversation view, suggestions, slash commands, history, progress indicators, settings screens, and approval dialogs.

The most important user-facing distinction is:

- **ordinary text** normally enters the AI conversation;
- **slash commands** may control the local application;
- **Bash mode** treats input as a shell action;
- **remote input** can be restricted so it cannot silently invoke unsafe local commands.

## 1. Rendering stack

[`launchRepl()`](../../../src/replLauncher.tsx) mounts the application. [`src/screens/REPL.tsx`](../../../src/screens/REPL.tsx) is the main interactive screen, while [`src/ink/ink.tsx`](../../../src/ink/ink.tsx) implements the custom terminal renderer.

The renderer uses React reconciliation, front/back frame buffering, focus state, selection, search highlights, scrolling, keyboard and mouse input, and optional alternate-screen behavior. Core terminal primitives live under [`src/ink/components/`](../../../src/ink/components/).

## 2. Prompt input

[`PromptInput.tsx`](../../../src/components/PromptInput/PromptInput.tsx) composes:

- normal or Vim text input;
- suggestions/autocomplete;
- queued command display;
- footer/status indicators;
- global and command keybinding handlers.

Input modes include normal prompts, Bash mode, orphaned permission responses, and task notifications. Cursor offset, line, and column are tracked separately.

External insertion, such as speech-to-text or paste, is distinguished from internal typing so the cursor can move predictably.

## 3. Submission

[`handlePromptSubmit.ts`](../../../src/utils/handlePromptSubmit.ts) performs the main split:

- immediate local/JSX slash commands execute in the UI process;
- ordinary prompts and non-immediate commands enter the message queue.

Before queueing, paste placeholders are expanded and images are converted to content blocks.

Queued entries record value, mode, priority, UUID, pasted content, origin, hidden/meta status, and optional target agent. Priorities allow “now,” “next,” and “later” behavior around a running turn.

Remote/bridge-originated messages can disable slash-command interpretation or restrict commands to bridge-safe entries. This prevents text received from another channel from silently becoming a privileged local command.

Before pressing Enter, check which input mode is active. The same visible text can mean “ask the model” or “run this shell command” depending on mode.

## 4. Command model

[`src/types/command.ts`](../../../src/types/command.ts) distinguishes:

- prompt commands that become model input;
- local commands that run JavaScript;
- local JSX commands that mount an interactive dialog.

Commands are lazy-loaded to reduce startup cost. The registry in [`src/commands.ts`](../../../src/commands.ts) combines built-ins, filesystem skills, plugins, managed commands, and MCP-provided commands.

## 5. Skills and plugins

[`loadSkillsDir.ts`](../../../src/skills/loadSkillsDir.ts) discovers Markdown skills/commands with frontmatter from bundled, managed, plugin, project, and user locations. It resolves symlinks for deduplication and caches results. Change detectors invalidate caches when relevant files change.

[`loadPluginCommands.ts`](../../../src/utils/plugins/loadPluginCommands.ts) adds plugin commands lazily. MCP servers can also expose command-like skills.

The command source and execution context are retained so telemetry, permission, and subagent behavior know whether a command was bundled, user-defined, managed, plugin-provided, or MCP-provided.

Skills and plugins can change what slash commands do. Do not assume a command with a familiar-looking name is built in; its source matters.

## 6. History and paste storage

[`src/history.ts`](../../../src/history.ts) writes project/session-aware JSONL history. It keeps a bounded recent list and deduplicates display strings for navigation.

Small pastes can remain inline. Large text is hashed and externalized; history stores a numbered placeholder that resolves lazily. Replacement works from later offsets backward so inserted content cannot shift the remaining offsets.

Removing a just-submitted entry handles a race:

- if still buffered, remove it directly;
- if already flushed, remember its timestamp as skipped when reading.

Large pasted text may be stored separately and represented by a placeholder in history. Pasting sensitive data can therefore create both provider-visible content and local history artifacts.

## 7. Keybindings and focus

Global and command handlers live in [`useGlobalKeybindings.tsx`](../../../src/hooks/useGlobalKeybindings.tsx) and [`useCommandKeybindings.tsx`](../../../src/hooks/useCommandKeybindings.tsx). A keybinding context marks which nested surface currently owns input.

Examples include exit/EOF, transcript paging, history search, autocomplete, command hints, fast-mode toggle, Vim behavior, and double-Escape actions.

Dialogs can temporarily own Escape. For example, settings defer closing while a child search/menu is handling the same key.

## 8. REPL state

The REPL coordinates:

- message transcript and virtualized scrolling;
- prompt/queue state;
- tool progress and permission dialogs;
- session resume and compaction;
- tips, status, token/cost display;
- background tasks, agents, teams, and notifications;
- plugin update checks;
- remote/bridge state;
- transcript mode and search.

Its size reflects orchestration of many UI states, not the model logic itself. Model iteration remains in [`src/query.ts`](../../../src/query.ts).

## 9. Important subtle behavior

- Large paste references are not expanded by naive global string replacement.
- Agent-targeted queued commands are isolated from other agents.
- Vim and normal input swap implementations while preserving surrounding prompt state.
- Settings and nested dialogs coordinate key ownership to avoid accidental closure.
- Input can be temporarily suppressed around loading transitions to avoid duplicate submission.
- Tool renderers provide separate compact, verbose, progress, rejected, and searchable text representations.
- Transcript search indexes what the terminal actually renders, not merely the model-facing serialized result.
