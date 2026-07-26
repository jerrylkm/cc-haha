# 10 — User questions and practical answers

This page answers the questions a user is likely to ask before studying the implementation.

## Is this just a chatbot?

No. It combines a chatbot-like model with local and remote tools. It can answer in text, but it can also request file reads, edits, terminal commands, web operations, subagents, and extension-provided actions.

## Does it have unrestricted access to my whole computer?

Not automatically in every mode, but do not treat it as a strict container by default.

Normal path permissions focus on the working directory and additional allowed directories. Sensitive paths receive extra treatment, and shell commands may be sandboxed. However:

- permission and sandbox settings vary;
- bypass modes may exist when authorized;
- a shell command is a powerful general-purpose capability;
- user-approved commands can affect locations or services outside the project;
- plugins/MCP services create additional boundaries.

The safe assumption is: **review what it is asking to do, not merely which folder you opened.**

## Can it control my mouse, keyboard, or screen?

Only if the Computer Use feature is enabled and you grant it operating-system permission. It is off by default and does nothing until you turn it on and approve Accessibility/Screen-Recording access.

When enabled, it can take screenshots and move the mouse, click, type, and scroll. There are layered safeguards: a master kill switch, per-app permission tiers (browsers are screenshot-only, terminals cannot be typed into), a single-session lock, and a clipboard guard against paste-injection. Even so, a fully granted app can be broadly controlled, so treat authorizing Computer Use like handing over your mouse and keyboard.

For the full mechanism and gates, see [16 — Computer Use and browser control](16-computer-use.md).

## Can it read passwords or API keys?

The code has sensitive-path checks and policies, but no program can infer every secret location or prevent a user-approved command from reading one.

Avoid asking it to inspect secret stores. Do not paste secrets into chat. Review file paths and shell commands, especially those mentioning home-directory settings, environment variables, credentials, keychains, browser profiles, or cloud configuration.

## Does a permission prompt show exactly what will happen?

It shows the tool and proposed input, often with a human-readable description or diff. The runtime also parses and normalizes inputs for security checks.

Still verify:

- the full file path;
- whether a write creates, edits, or overwrites;
- every command in a compound shell expression;
- network destinations;
- whether “remember this choice” broadens future permission;
- whether the action is destructive or externally visible.

## Why did it not ask me?

Possible reasons include:

- the tool is read-only;
- an allow rule matched;
- the current permission mode pre-approves that category;
- a hook or classifier approved it;
- the operation ran inside an allowed sandbox;
- you already approved a matching rule;
- the action was internal bookkeeping rather than a tool;
- the visible operation was performed by an already approved broader command.

Use [04 — Tools, permissions, and security](04-tools-permissions-security.md) to trace the decision.

## Why did it ask for something harmless?

The permission system is deliberately conservative. It may ask because parsing was uncertain, a path is sensitive, a command contains too many subcommands, a rule explicitly says “ask,” sandboxing is unavailable, or an automated check could not confidently approve it.

False positives are safer than silently running a command the parser misunderstood. For the exact list of checks and why each one exists, see [11 — Bash security checks in depth](11-bash-security-checks.md).

## Why is a tool missing?

A tool can be absent because:

- its build feature was excluded;
- its environment variable is off;
- `isEnabled()` rejected the current mode/platform;
- a blanket deny rule removed it;
- simple, REPL, coordinator, or agent mode filtered it;
- it is deferred behind ToolSearch;
- an MCP server has not connected;
- its implementation is only a stub in this reconstructed source.

See [08 — Complete tool catalog](08-complete-tool-catalog.md).

## Why does the assistant say it cannot do something even though a related tool exists?

The model sees only the final session-specific tool pool. A source file can exist without its schema being exposed. The current agent may also have a smaller allowlist than the main conversation.

## Where is my conversation saved?

Normally in a project-specific JSONL transcript under the Claude configuration directory, commonly below `~/.claude/projects/`. Exact paths can change with configuration.

A local SQLite index helps search/list sessions, but the JSONL transcript is the durable record.

## What else is saved besides chat text?

Potentially:

- tool requests and results;
- titles and tags;
- cost/token summaries;
- task state;
- permission mode;
- worktree information;
- file history and attribution snapshots;
- compaction/collapse metadata;
- background-agent activity;
- large tool-result file references.

## If I delete the visible chat, is all related data gone?

Do not assume so. History, transcript, indexes, tool-result files, caches, plugins, remote providers, MCP servers, and integrations can have separate storage or retention. Deletion behavior must be checked for each configured component.

## Does it send my entire project to the AI provider?

Not necessarily. The runtime selects context and reads files as needed. However, any relevant file content or command output placed into model messages is sent to the configured provider.

Large projects also have context limits, so the program uses targeted reads, search, result limits, and compaction rather than fitting the entire repository into every request.

## What is compaction? Does it delete my transcript?

Compaction shortens what is sent back to the model by summarizing or removing older low-value context. The visible/local transcript can retain more than the model’s current working context.

Compaction can lose detail from the model’s active memory even when the original transcript still exists on disk. For the exact thresholds, the summary format, what is preserved, and how to tune or disable it, see [12 — Context limits and compaction in depth](12-context-and-compaction.md).

## Can the assistant remember something forever?

Not automatically.

- Conversation context is limited.
- Compaction can summarize older details.
- Session transcripts allow resume.
- Memory files or skills can provide longer-lived instructions.
- Plugins may add their own storage.

Persistent memory should be treated as explicit stored data, not human-like recollection.

## What does “one turn” mean?

It starts with one user message and ends when the assistant finishes that request. Inside it, the model may be called many times:

```text
you ask
  -> model asks to read a file
  -> file is read
  -> model asks to run a test
  -> test runs
  -> model gives final answer
```

## Why did it use several tools at once?

Independent read-only actions can run concurrently to save time. File changes and uncertain actions are normally serialized so later actions see the state produced by earlier ones.

## Can two tools change the same file at the same time?

The orchestration system asks tools whether concurrency is safe. Mutating operations are intended to run serially. A thrown or uncertain concurrency check defaults to serial execution.

This reduces risk but does not replace reviewing the final file state.

## What is a subagent?

A subagent is another AI task runner given a focused prompt. It may work synchronously or in the background and may use a restricted tool set.

Subagents can improve parallelism, but they can also increase model usage and make the activity harder to follow. Ordinary async agents are prevented from recursively using several main-thread control tools. For how subagents, forks, and teams work in detail, see [14 — Multi-agent, subagents, and teams](14-multi-agent-and-teams.md).

## What is an MCP server, and should I trust it?

An MCP server is an external program or service that supplies tools/resources. Trust it as you would any installed extension:

- inspect who provides it;
- understand its network and file access;
- review authentication;
- restrict its permission rules;
- remember that its tool names and schemas are dynamic.

The built-in catalog cannot enumerate arbitrary MCP tools.

## What is a skill? Can a Markdown file be dangerous?

A skill is largely instructions and workflow metadata, but those instructions can encourage the model to call powerful tools. A plugin-provided skill can also be part of a larger extension.

Review untrusted skills before installing them. “It is only Markdown” does not mean its requested workflow is harmless.

## What is a hook?

A hook is configured logic that runs at lifecycle points such as before a tool, after a tool, on session start, or when a prompt is submitted.

Hooks can block, approve, modify inputs, add context, or run commands. A project with hooks can behave differently from a clean project even when the user types the same request.

## Does sandboxing make every command safe?

No. Sandboxing limits selected filesystem/network effects when enabled and supported. It does not prove that:

- the command is logically correct;
- allowed files cannot be damaged;
- an allowed network request is harmless;
- credentials were not explicitly supplied;
- a user-approved unsandboxed command is safe.

The source itself treats excluded-command configuration as convenience, not the security boundary.

## Can it spend money?

Potentially:

- model calls can incur provider charges;
- web/provider services may be billable;
- subagents and retries add usage;
- commands can invoke paid cloud tools;
- externally visible actions can create resources.

The local cost tracker estimates/model-reports usage but cannot prevent every external command from spending money.

## Can it publish, push, send messages, or modify remote services?

Only when a suitable tool, shell command, plugin, MCP server, or integration is available and permission allows it. These actions can be irreversible and externally visible.

Do not treat a successful local diff review as proof that no remote side effect occurred. If you control the session remotely (web, phone, or a chat app), see [17 — Remote access and IM bridge](17-remote-access-and-bridge.md) for how those channels are secured.

## How can I tell what it changed?

Use:

- the tool transcript;
- file diffs;
- Git status/diff;
- worktree isolation;
- test results;
- command output;
- session activity panels.

Always distinguish “the command ran” from “the requested behavior is correct.”

## What if the program crashes?

Session transcripts, queued writes, resume logic, task state, and local indexing support recovery. Some in-flight output can still be lost, and an external action may have completed even if its final result was not recorded.

After a crash, inspect file and external-service state before repeating a destructive operation.

## Practical toolkit: how do I find and control my own data?

The earlier answers explain *what* is stored and sent. This section is the concrete "how." Commands are typed in the input box starting with `/`; exact wording can vary by version, so use `/help` if a name differs.

### Where is my data on disk?

Local state lives under the Claude configuration home directory, commonly `~/.claude` (it can be redirected with the `CLAUDE_CONFIG_DIR` environment variable). Useful locations inside it include:

| What | Typical location | Notes |
|---|---|---|
| Conversation transcripts | `~/.claude/projects/<encoded-project-path>/<session-id>.jsonl` | The durable per-session record. One JSON object per line. |
| Search/index database | Local SQLite index maintained by the server | Speeds up listing/search; rebuilt from the transcripts if lost. |
| Settings and auth | `~/.claude/` settings files (for example `~/.claude.json`) | Provider and configuration state. Treat as sensitive. |
| Backups | `~/.claude/backups/` | Kept so a bad write does not destroy a good configuration. |
| Command history, caches | `~/.claude/` history and cache files | Includes externalized large pastes. |

To open a transcript yourself, find the matching `.jsonl` file under `~/.claude/projects/` and read it in any text editor. Remember it contains more than the visible chat (tool results, titles, permission mode, and other metadata).

### How do I inspect a session without digging through files?

| Goal | Do this |
|---|---|
| See version, model, account, API connectivity, and tool status | `/status` |
| See cost and duration of the current session | `/cost` |
| Save the current conversation to a file or clipboard | `/export` |
| Reopen an earlier conversation | `/resume`, or start with `--resume` / `--continue` |
| Diagnose installation and settings problems | `/doctor` |
| View or change privacy settings | `/privacy-settings` |

### How do I see exactly what it changed?

Do not rely on the assistant's summary alone. Verify with:

- the tool transcript and per-tool diffs shown in the UI;
- `git status` and `git diff` in the project (the most reliable check for file changes);
- test or build output;
- for isolated work, a Git worktree so changes stay off your main branch until you review them.

A command reporting success is not proof the change is correct or complete.

### How do I clear or remove data?

- `/clear` clears the current conversation from active context (it frees the model's working memory; it does not by itself erase every on-disk artifact).
- To remove a stored conversation, delete the corresponding `.jsonl` file under `~/.claude/projects/`. The derived index can be rebuilt, so removing the transcript is the meaningful step.
- **Deleting the visible chat is not the same as deleting all related data.** As noted above, history, caches, exported files, large tool-result files, plugin/MCP storage, and remote/provider records can persist separately. To fully clean up, account for each configured component, not only the transcript.

### What should I check before doing something risky?

1. Back up or commit important files first (version control is the best undo).
2. Read the exact command, path, and diff in each approval — not just the fact that "the AI asked."
3. Watch for deletion, overwrite, credential access, installation, network, and "remember this choice" (rule-broadening) behavior.
4. Prefer a worktree for large or uncertain changes.
5. After the turn, review the diff and run tests before trusting the result.

## Which pages should I read if I only care about using it safely?

Read:

1. [00 — Beginner guide](00-beginner-guide.md);
2. [04 — Tools, permissions, and security](04-tools-permissions-security.md);
3. [08 — Complete tool catalog](08-complete-tool-catalog.md);
4. this FAQ.

Use [Glossary](glossary.md) for unfamiliar terms. The source atlas and function reference are optional.

