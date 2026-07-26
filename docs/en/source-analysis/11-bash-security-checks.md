# 11 — Bash security checks in depth

This chapter expands the “Security checks” subsection of [04 — Tools, permissions, and security](04-tools-permissions-security.md) into a complete breakdown.

Everything here concerns one question:

> When the model asks to run a shell command, how does the program decide whether the command is safe, must ask you, or must be blocked?

## 1. Why shell commands need special checks

Most tools accept simple, structured input. A shell command is different: it is a small program in a language full of ways to hide meaning. The same visible text can:

- run more than one command;
- expand into a different command than it appears to;
- read secret files;
- reach the network;
- be parsed one way by a safety checker and another way by the real shell.

A permission rule such as “allow `git *`” is only safe if the checker and the real shell agree on what the command actually is. Most of these checks exist to close that gap, especially tricks that make an allowlisted command smuggle in a second, dangerous command.

For non-programmers: this chapter explains why the program sometimes asks for approval on a command that looks harmless. It is usually being cautious about a parsing trick, not about the obvious command.

## 2. Two checking paths

There are two implementations of the same idea.

| Path | Location | Role |
|---|---|---|
| Primary (AST) | [`src/utils/bash/ast.ts`](../../../src/utils/bash/ast.ts) via `parseForSecurity` / `parseForSecurityFromAst` | Preferred gate. Parses the command into a structured tree and reasons about real command structure. |
| Legacy (regex/shell-quote) | [`bashSecurity.ts`](../../../src/tools/BashTool/bashSecurity.ts) via `bashCommandIsSafe_DEPRECATED` and its async variant | Fallback used when tree parsing is unavailable; also reused by some read-only checks. |

The Bash tool calls `parseForSecurity()` from [`BashTool.tsx`](../../../src/tools/BashTool/BashTool.tsx). The command parser that produces the tree is [`src/utils/bash/bashParser.ts`](../../../src/utils/bash/bashParser.ts), which limits parse time and node count so a hostile command cannot exhaust resources.

The check-by-check list below comes from the legacy validator suite because that file enumerates each check explicitly with a numeric ID. The AST path mirrors the same intent with more accurate parsing, and the two are deliberately kept consistent. Where the AST path adds its own semantic checks, they are described in section 7.

## 3. Three possible outcomes

Each validator returns one of three results:

| Result | Meaning |
|---|---|
| `passthrough` | This validator found nothing; continue to the next check and the normal permission flow. |
| `ask` | Stop and require user approval. |
| `allow` | Positively assert the command is safe (used only by a few early validators). |

An `ask` result can also carry a flag, `isBashSecurityCheckForMisparsing`. This flag means “the checker and the real shell may disagree about this command,” and it causes an early hard stop at the permission gate rather than a normal prompt. Two categories of parsing trick are treated as misparsing; ordinary newlines and redirections are not.

## 4. Order of checking

The legacy suite runs in a deliberate order.

### Pre-checks (before parsing)

1. **Control characters** — non-printable bytes that can hide content. `ask`, flagged as misparsing.
2. **Single-quoted backslash bug** — a known quoting bug that a naive parser mishandles. `ask`, flagged as misparsing.

Then quoted heredoc bodies are stripped so their literal text does not trigger false alarms, and the command is separated into quoted/unquoted views for analysis.

### Early validators (an `allow` short-circuits)

3. `validateEmpty` — an empty command is safe.
4. `validateIncompleteCommands` — fragments (for example, starting with a stray tab) are treated as suspicious.
5. `validateSafeCommandSubstitution` — recognizes specific safe heredoc/substitution shapes.
6. `validateGitCommit` — allows normal `git commit -m "message"` while rejecting messages that hide command substitution.

If an early validator returns `allow`, the command is treated as safe and the remaining checks are skipped. This is why these four are written very defensively: a false “allow” here would bypass everything after it.

### Main validators (an `ask` can stop)

The remaining checks run in a fixed sequence. Most `ask` results are treated as misparsing and stop immediately. Two checks—**newlines** and **redirections**—are “non-misparsing”: their `ask` result is deferred, so a later misparsing check can take priority. This ordering prevents an attacker from making a harmless-looking redirection fire first to hide a real parsing trick later in the same command.

## 5. The complete check list

The legacy suite assigns each check a numeric ID (numbers are logged instead of the raw command, to avoid recording command text). The identifiers below come directly from [`bashSecurity.ts`](../../../src/tools/BashTool/bashSecurity.ts).

| ID | Name | Plain-language purpose |
|---|---|---|
| 1 | Incomplete commands | Flags fragments/partial commands that suggest broken or crafted input. |
| 2 | jq system function | Blocks `jq` using its system-execution feature to run commands. |
| 3 | jq file arguments | Blocks `jq` argument shapes that could read unexpected files. |
| 4 | Obfuscated flags | Detects hidden or disguised flags (for example ANSI‑C `$'...'` quoting and quote-splitting) used to smuggle dangerous options past allowlist patterns. |
| 5 | Shell metacharacters | Detects metacharacters that can chain or redirect beyond the apparent command. |
| 6 | Dangerous variables | Detects variable usage that can alter parsing or expand into other commands. |
| 7 | Newlines | Detects unquoted newlines that could separate multiple commands (non-misparsing). |
| 8 | Dangerous patterns: command substitution | Detects `$( )`, backticks, and related substitution that runs nested commands. |
| 9 | Dangerous patterns: input redirection | Detects input redirection that could read unexpected sources. |
| 10 | Dangerous patterns: output redirection | Detects output redirection that could write unexpected files (non-misparsing). |
| 11 | IFS injection | Detects use of the `IFS` variable, which can change how words split and defeat validation. |
| 12 | Git commit substitution | Detects command substitution hidden inside a commit message. |
| 13 | /proc environ access | Detects reads of `/proc/*/environ`, which can leak environment secrets. |
| 14 | Malformed token injection | Detects token shapes that parse incorrectly and could hide a second command. |
| 15 | Backslash-escaped whitespace | Detects escaped spaces used to disguise separate words/commands. |
| 16 | Brace expansion | Detects `{...}` expansion that can generate multiple arguments or commands. |
| 17 | Control characters | Detects non-printable characters used to bypass checks. |
| 18 | Unicode whitespace | Detects unusual Unicode spaces that a checker and shell may treat differently. |
| 19 | Mid-word hash | Detects `#` positioned to create a comment/quote desync that hides content. |
| 20 | Zsh dangerous commands | Detects zsh module builtins (see below) that enable file, socket, or command execution outside normal binaries. |
| 21 | Backslash-escaped operators | Detects escaped shell operators (for example `\;`) that a checker may miss but the shell honors. |
| 22 | Comment/quote desync | Detects `#` comment handling that can desynchronize quote tracking and hide a newline-separated command. |
| 23 | Quoted newline | Detects newlines inside quotes used to split a command across lines so line-based filtering drops sensitive content. |

Carriage return (`\r`) is handled by a dedicated validator and reuses the newline ID with a sub-identifier. It is treated as a misparsing concern because a checker may split on `\r` while the shell does not.

### Detail: the jq checks (IDs 2–3)

`jq` is a JSON tool, but it can execute programs through a system feature and can be pointed at files. The checks separate a normal filter from a `jq` invocation that reaches outside JSON processing.

### Detail: command/redirection patterns (IDs 8–10)

These detect the classic building blocks of “run something else”: nested command substitution, reading from an unexpected input, and writing to an unexpected output. Output redirection is “non-misparsing” because a plain `>` is normal and the surrounding code validates real write paths elsewhere; its prompt is deferred so a stronger misparsing check can win.

### Detail: parser-differential checks (IDs 14, 17–19, 21–23, plus CR)

This is the most important cluster. Each targets a case where the safety checker and the real shell could disagree:

- malformed tokens;
- control characters and unusual Unicode whitespace;
- a `#` placed to break quote tracking;
- escaped operators such as `\;`;
- newlines hidden inside quotes;
- carriage returns treated as separators by one parser but not the other.

Because a disagreement means “the rule you approved might not describe what actually runs,” these produce a misparsing `ask` that stops early.

### Detail: zsh module commands (ID 20)

The code keeps an explicit set of dangerous zsh builtins, including `zmodload`, `emulate`, `sysopen`/`sysread`/`syswrite`/`sysseek`, `zpty`, `ztcp`, `zsocket`, `mapfile`, and the `zf_*` file builtins. These can perform file input/output, open sockets, or run commands without invoking normal binaries, so they are checked against each command segment’s base word.

### Detail: command-substitution patterns list

Beyond the numbered checks, the file keeps a pattern list for substitution/expansion shapes, including process substitution `<()` / `>()`, zsh `=()`, zsh “equals expansion” (`=cmd`, which can expand to a full path and dodge a `curl` deny rule), `$()`, `${}`, legacy `$[ ]` arithmetic, several zsh-specific expansions, and even PowerShell comment syntax as defense in depth. This is why a command containing `$(...)` commonly triggers an approval prompt.

## 6. Extra safeguards around the checks

- **Subcommand cap.** [`bashPermissions.ts`](../../../src/tools/BashTool/bashPermissions.ts) defines `MAX_SUBCOMMANDS_FOR_SECURITY_CHECK = 50`. If a command explodes into more than fifty subcommands through the legacy split path, the runtime returns `ask` instead of trying to analyze each one. Very large compound commands are treated as “too complex to auto-approve.”
- **Wrapper stripping.** Permission matching can peel wrappers such as `timeout`, `time`, `nice`, `stdbuf`, and `nohup`, plus simple safe environment assignments, repeating until stable. Flag/value syntax is constrained and only horizontal whitespace is peeled, so a newline is never swallowed as if it were an ordinary space.
- **Safe redirection stripping.** Harmless redirections like `2>&1` and `>/dev/null` are removed with strict trailing boundaries so a lookalike such as `/dev/nullo` cannot be mistaken for `/dev/null`.
- **Heredoc handling.** Quoted heredoc bodies (literal text) are stripped before checking; unquoted heredocs, which the shell would expand, are left in place so their contents are still inspected.
- **Analytics.** When a check fires, the code logs a `tengu_bash_security_check_triggered` event carrying the numeric ID (and a sub-identifier), not the command text.

## 7. What the AST path adds

The AST path in [`ast.ts`](../../../src/utils/bash/ast.ts) parses the command first, then classifies it (for example “simple” versus “too complex”). After a simple classification, `checkSemantics` runs post-argument checks and performs the same wrapper stripping described above, keeping behavior aligned with the legacy validators. Because it works from real structure rather than regular expressions, it is less prone to the parser-differential tricks the misparsing checks defend against—which is why it is the primary gate and the regex suite is the fallback.

## 8. How this connects to permission rules

Passing these checks does not mean the command runs without approval. The security checks are one input to the wider decision described in [04 — Tools, permissions, and security](04-tools-permissions-security.md):

```text
parse command (limits on time/size)
  -> security checks (this chapter)
  -> wrapper/redirection normalization
  -> match against your allow/ask/deny rules and policy
  -> sensitive-path and sandbox considerations
  -> possibly ask you
  -> run (optionally sandboxed)
```

A misparsing result can force a stop even when a rule would otherwise allow the command, precisely because the rule might be matching something different from what the shell would execute.

## 9. What this means for you as a user

- A prompt on a “simple” command is usually the parser being careful about a trick, not a bug.
- Commands containing `$(...)`, backticks, `${...}`, `|`, `;`, `&`, `<`, `>`, escaped punctuation, unusual spacing, or `IFS` are more likely to require approval.
- Approving such a command means you accept that it may do more than the obvious first word suggests.
- These checks reduce risk from disguised commands. They do not judge whether an obvious, correctly understood command is a good idea. `rm -rf` on real files is “understood,” not “safe.”

## 10. Where to look in the source

| Topic | File |
|---|---|
| Primary gate and semantics | [`src/utils/bash/ast.ts`](../../../src/utils/bash/ast.ts) |
| Command parser | [`src/utils/bash/bashParser.ts`](../../../src/utils/bash/bashParser.ts) |
| Check list and validators | [`src/tools/BashTool/bashSecurity.ts`](../../../src/tools/BashTool/bashSecurity.ts) |
| Permission matching, wrapper stripping, subcommand cap | [`src/tools/BashTool/bashPermissions.ts`](../../../src/tools/BashTool/bashPermissions.ts) |
| Sandbox decision | [`src/tools/BashTool/shouldUseSandbox.ts`](../../../src/tools/BashTool/shouldUseSandbox.ts) |
| Bash tool entry point | [`src/tools/BashTool/BashTool.tsx`](../../../src/tools/BashTool/BashTool.tsx) |
