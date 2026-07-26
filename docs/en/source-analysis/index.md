# Claude Code CLI source analysis

This folder explains the recovered Claude Code CLI code in this repository. No programming knowledge is required for the beginner path.

> **Provenance limitation:** The repository maintainers state that the project was repaired from source exposed through Anthropic's npm registry on 2026-03-31. This analysis can describe the checked-in code, but it cannot independently verify that historical claim or attribute every current file to Anthropic. The desktop app, local server, messaging adapters, and later fixes include substantial project-specific work.

## Choose your reading path

### I use the program but do not write code

Start with:

1. [00 — Beginner guide](00-beginner-guide.md)
2. [10 — User questions and practical answers](10-user-faq.md)
3. [04 — Tools, permissions, and security](04-tools-permissions-security.md)
4. [08 — Complete tool catalog](08-complete-tool-catalog.md)

These pages explain capabilities, approval, privacy, local storage, network activity, costs, stopping, recovery, and extension risks.

### I want to understand how it works

Read chapters 01 through 06, then use the tool catalog.

### I am debugging or auditing the source

Use [09 — Core function reference](09-core-function-reference.md) and [07 — Source atlas](07-source-atlas.md). They are reference pages and are not intended as the beginner introduction.

## What “complete” means here

Reprinting every source line would be neither useful nor appropriate. Instead, this guide provides:

1. a plain-language explanation of every major runtime path;
2. direct links to the relevant implementation;
3. subtle behavior, failure handling, and compatibility details;
4. a mechanically generated atlas containing every code-like file in the repository.

Use the atlas to account for individual files, and the explanatory chapters to understand how they work together.

## Reading order

| Chapter | Question answered |
|---|---|
| [00 — Beginner guide](00-beginner-guide.md) | What can the program do, what can leave my computer, and how do I stay in control? |
| [01 — Orientation](01-orientation.md) | What is this repository, and which parts belong to which runtime? |
| [02 — Startup and one turn](02-startup-and-one-turn.md) | What happens from launching the command to receiving an answer? |
| [03 — Agent loop](03-agent-loop.md) | How does the model/tool/recovery loop actually work? |
| [04 — Tools, permissions, and security](04-tools-permissions-security.md) | How are actions validated, approved, sandboxed, and executed? |
| [05 — Context, persistence, and services](05-context-persistence-services.md) | How are prompts, sessions, providers, MCP/LSP, cost, and telemetry handled? |
| [06 — Terminal UI and commands](06-terminal-ui-commands.md) | How do input, slash commands, history, plugins, skills, and Ink rendering work? |
| [07 — Source atlas](07-source-atlas.md) | Which code-like files exist, where are they, and what area owns them? |
| [08 — Complete tool catalog](08-complete-tool-catalog.md) | Which tools exist, what does each do, and why can the visible list change? |
| [09 — Core function reference](09-core-function-reference.md) | Which exact symbols own startup, turns, streaming, tools, permissions, and persistence? |
| [10 — User FAQ](10-user-faq.md) | Where is data saved, why did it ask/not ask, what can go wrong, and how do I check? |
| [Glossary](glossary.md) | What do the recurring terms mean? |

## The shortest possible mental model

Claude Code is a loop around a language model:

```text
collect instructions and conversation
              |
              v
       ask the model to act
              |
       +------+------+
       |             |
   text answer    tool request
       |             |
     finish      check permission
                     |
                  run tool
                     |
              return tool result
                     |
              ask model again
```

The surrounding code makes this small loop reliable: it builds context, streams partial output, protects dangerous operations, persists transcripts, compacts long conversations, retries recoverable failures, and presents everything in a terminal or desktop UI.

For a concrete inventory rather than a framework summary, see [the complete tool catalog](08-complete-tool-catalog.md). It covers every registration branch in the built-in source, dynamic MCP tools, structured output, aliases, feature gates, agent-specific filtering, and internal modules whose implementations are missing from this checkout.

## Scope boundaries

- **Recovered CLI/runtime core:** primarily [`src/`](../../../src/), [`bin/claude-haha`](../../../bin/claude-haha), and [`preload.ts`](../../../preload.ts).
- **Local API and protocol bridge:** [`src/server/`](../../../src/server/), part of the current runtime but heavily extended for the desktop app.
- **Project extensions:** [`desktop/`](../../../desktop/), [`adapters/`](../../../adapters/), and many later services.
- **Supporting code:** [`runtime/`](../../../runtime/), [`scripts/`](../../../scripts/), [`tests/`](../../../tests/), and [`site/`](../../../site/).

The chapters focus on the CLI/runtime core while calling out these boundaries whenever behavior crosses them.
