# 00 — Beginner guide: what this program actually does

This chapter is for readers who do not write code. You do not need to understand TypeScript, APIs, or terminal commands.

## 1. What is Claude Code?

Claude Code is not only a chat box. It is an AI assistant that can be connected to a folder on your computer.

Depending on its settings and your approval, it can:

- read files;
- search through folders;
- explain a project;
- create, edit, or overwrite files;
- run terminal commands;
- search or fetch information from the web;
- ask you questions;
- start smaller helper agents;
- connect to extra services through plugins or MCP;
- remember a conversation by saving a local transcript.

The most important difference from an ordinary chatbot is this:

> An ordinary chatbot usually tells you what to do. Claude Code can sometimes do it on your computer.

The built-in capabilities are listed in [the complete tool catalog](08-complete-tool-catalog.md).

## 2. What happens after I type a request?

Imagine you ask:

> “Please find why this application does not start.”

The program roughly does this:

1. **Collects context.** It looks at your request, previous messages, project instructions, available tools, and basic workspace information.
2. **Sends a request to the selected AI provider.** The provider returns either an answer or a request to use a tool.
3. **Checks the proposed action.** The program validates the tool input and checks your permission rules.
4. **Asks you when required.** A risky command or file change may produce an approval screen.
5. **Runs the approved tool.** For example, it may read a log file or run a test command.
6. **Sends the result back to the AI.** The AI uses that result to decide the next step.
7. **Repeats until finished.** One sentence from you can lead to several reads, searches, commands, and AI requests.
8. **Saves the conversation locally.** Unless persistence is disabled, the messages and operational metadata are written to a transcript.

For the detailed version, see [02 — Startup and one turn](02-startup-and-one-turn.md).

## 3. Is the AI itself reading my files?

There are two parts:

- **The local program** reads a file by running a tool on your computer.
- **The AI provider** receives the file content that the program includes in the conversation or tool result.

This distinction matters. A file can remain local if it is never placed in a model request, but content read for the model normally becomes part of the request sent to the configured provider.

The code also builds context from project instructions and may include workspace information needed to perform the task. Do not open a sensitive folder and assume that “local tool” means “nothing is sent to a provider.”

## 4. What can leave my computer?

Depending on the feature and configuration, network traffic can include:

| Destination | Possible information |
|---|---|
| Selected AI provider | Your prompt, relevant conversation history, system instructions, tool definitions, and file/command results included for the model |
| Web search/fetch service | Search terms, requested URLs, and fetched page processing |
| MCP server | Arguments sent to tools hosted by that server |
| Plugin or integration service | Data required by that plugin, such as messages or repository actions |
| Telemetry/analytics sink | Operational event metadata, subject to configuration and the code’s privacy conventions |
| Remote/IM features | Conversation and approval information needed for the selected remote channel |

The exact destination depends on provider, plugins, MCP servers, remote features, and settings. This source analysis cannot guarantee the privacy policy of a third-party provider or plugin.

## 5. What stays on my computer?

The runtime normally stores state under the Claude configuration area, commonly `~/.claude`, although configuration can redirect some paths.

Local state can include:

- settings and provider configuration;
- project/session transcripts;
- command history;
- cached plugin or skill information;
- task and worktree metadata;
- large tool results that were too big to keep inside the conversation;
- a local search index derived from transcripts;
- cost and usage summaries.

The transcript is more than the visible chat. It can contain tool results, compaction markers, titles, task information, permission mode, file history, and resume metadata.

See [05 — Context, persistence, and services](05-context-persistence-services.md) for details, or the [practical toolkit in the FAQ](10-user-faq.md#practical-toolkit-how-do-i-find-and-control-my-own-data) for the concrete file locations and the commands (`/status`, `/cost`, `/export`, `/clear`) you can use to find, inspect, and clear your own data.

## 6. Does it always ask before changing something?

No single answer applies to every session. Permission behavior depends on:

- the selected permission mode;
- user, project, local, managed-policy, command-line, and session rules;
- the tool and its input;
- whether the path or command is considered sensitive;
- hooks or automated security checks;
- whether sandboxing is available;
- whether you previously allowed a matching action.

In the normal/default mode, uncertain or risky actions can ask you. However:

- an allow rule can avoid a prompt;
- a deny rule can block the action;
- `dontAsk` turns “ask” into “deny”;
- an authorized bypass mode can remove many prompts;
- some read-only actions may run without a dialog;
- plugins and MCP tools have their own behavior plus the central permission layer.

If you are unsure, review the exact command, file path, and proposed change instead of approving only because the message came from an AI.

The permission mode is not fixed for all time. It can come from a saved `defaultMode` setting, be chosen at launch with a command-line flag such as `--permission-mode`, or be changed during a session (for example through the in-app permission controls). Because the mode can shift, do not assume the behavior you saw earlier still applies later in the same session—check the current mode if a prompt appears or disappears unexpectedly.

See [04 — Tools, permissions, and security](04-tools-permissions-security.md).

## 7. What is the safest way to use it?

For important work:

1. **Use version control.** Commit or otherwise back up important files first.
2. **Open only the project you intend to work on.** The working directory defines much of the normal file scope.
3. **Keep the default permission mode until you understand the rules.**
4. **Read command approvals carefully.** Look for deletion, overwrite, upload, credential, installation, and network behavior.
5. **Use a worktree for isolated changes** when the feature is available.
6. **Do not expose secret files unnecessarily.**
7. **Review the resulting diff.** A successful command does not prove the change is correct.
8. **Run tests or ask the assistant to run them.**
9. **Treat third-party plugins, skills, hooks, MCP servers, and providers as additional trusted parties.**

Permissions reduce risk; they do not replace backups, review, and judgment.

## 8. Can it undo a mistake?

Sometimes, but not universally.

- File changes may be reviewable or recoverable through Git and file-history features.
- A worktree can isolate changes from your main working tree.
- Some edits include structured diffs and original content.
- A command can still delete data, change external services, send a message, or perform another irreversible action.

There is no general “undo everything the agent did” guarantee. The tool contract even marks some operations as destructive, but that label cannot reverse an operation afterward.

## 9. How do I stop it?

The UI supports interruption, but the outcome depends on the active tool:

- some tools can be cancelled and their result discarded;
- write-like operations default to finishing before a new message is handled;
- background tasks can continue until stopped with task controls;
- subprocesses may need graceful termination;
- a request already sent to a remote service cannot necessarily be recalled.

If an approval dialog is open, denying it prevents that proposed call. It does not retroactively undo earlier calls in the same turn.

## 10. Why does it sometimes continue after looking finished?

The model may have:

- requested another tool;
- reached an output limit and been told to continue;
- received blocking feedback from a stop hook;
- compacted a long conversation and retried;
- switched to a fallback model;
- received a tool result and started the next reasoning step.

The program hides some recoverable intermediate errors so the user sees one continuous attempt rather than a false failure followed by more output.

## 11. Why can the answer become slower or more expensive?

One request from you can involve:

- multiple model calls;
- large files or long conversation history;
- tool-result processing;
- web searches;
- subagents using additional model calls;
- retries after network/provider errors;
- compaction or summarization;
- a more expensive model.

The code tracks tokens, duration, and model usage, but actual billing rules come from the configured provider. A local command can also consume computer/network resources independently of model cost.

## 12. What are tools, skills, plugins, and MCP?

| Term | Everyday explanation |
|---|---|
| Tool | A specific action the AI may request, such as Read, Edit, Bash, or WebSearch |
| Skill | A reusable instruction/workflow package that teaches the agent how to handle a type of task |
| Plugin | An installed extension that can provide commands, skills, hooks, or integrations |
| MCP server | A separate program/service that advertises additional tools or resources |
| Hook | User- or project-defined logic that runs at moments such as before/after a tool |
| Agent | A helper AI session assigned a narrower task |

Adding an extension can add behavior that is not fully described by the built-in tool catalog.

## 13. Does this repository exactly equal official Claude Code?

That cannot be established from the files alone.

The repository maintainers say it was repaired from source exposed through Anthropic’s npm registry. The checked-in project also contains:

- generated stubs for missing internal modules;
- fixes needed to make the recovered source run;
- a local server;
- desktop applications;
- messaging adapters;
- later project-specific features.

This guide describes what is checked in. It does not certify provenance, completeness, security, or equivalence to an official release.

## 14. What should a non-programmer read?

Recommended order:

1. this beginner guide;
2. [10 — User questions and practical answers](10-user-faq.md);
3. [02 — Startup and one turn](02-startup-and-one-turn.md);
4. [04 — Tools, permissions, and security](04-tools-permissions-security.md);
5. [08 — Complete tool catalog](08-complete-tool-catalog.md) when you want to know a specific capability;
6. [Glossary](glossary.md) whenever a term is unfamiliar.

You can skip:

- [07 — Source atlas](07-source-atlas.md), unless locating a file;
- [09 — Core function reference](09-core-function-reference.md), unless debugging or studying implementation;
- most low-level details in [03 — Agent loop](03-agent-loop.md).

