# 12 — Context limits and compaction in depth

This chapter explains one of the most common user questions:

> Why did the assistant seem to “forget” earlier parts of a long conversation?

The short answer is that a model can only consider a limited amount of text at once (its **context window**). When a conversation approaches that limit, the program shortens what it sends back to the model. This chapter breaks down exactly how.

It expands the summaries in [03 — Agent loop](03-agent-loop.md) and [05 — Context, persistence, and services](05-context-persistence-services.md).

## 1. The core problem

Every message, tool result, file, and instruction sent to the model costs **tokens** (roughly, pieces of words). The model has a fixed budget. A long session—many files read, many commands run—steadily fills that budget.

Two important facts:

- the limit applies to what is **sent to the model**, not to what is stored locally;
- your on-disk transcript can keep far more than the model currently “sees.”

So “forgetting” usually means “this detail was summarized or dropped from the model’s working context,” not “this detail was deleted from your computer.”

## 2. The budget line: effective context window

The program does not fill the window to 100%. It reserves room for the model’s own reply and for a summary.

From [`autoCompact.ts`](../../../src/services/compact/autoCompact.ts):

- `MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20,000` — space reserved so a summary can be produced.
- `getEffectiveContextWindowSize(model)` = the model’s context window minus the reserved output tokens.

Everything below is measured against this **effective** window, not the raw window.

## 3. The thresholds

As token usage rises, the runtime crosses several lines. All values come from [`autoCompact.ts`](../../../src/services/compact/autoCompact.ts) and are measured relative to the effective window.

| Stage | Where it sits | What happens |
|---|---|---|
| Warning | effective window − `WARNING_THRESHOLD_BUFFER_TOKENS` (20,000) | The UI can warn that context is getting full. |
| Error/near-full | effective window − `ERROR_THRESHOLD_BUFFER_TOKENS` (20,000) | A stronger warning state. |
| Auto-compact | effective window − `AUTOCOMPACT_BUFFER_TOKENS` (13,000) | Automatic compaction may run before the next model call. |
| Blocking limit | effective window − `MANUAL_COMPACT_BUFFER_TOKENS` (3,000) | Treated as effectively full; the turn cannot simply keep growing. |

`calculateTokenWarningState()` returns `percentLeft` plus booleans for each stage. `percentLeft` is what a “context remaining” indicator reflects.

These are proactive thresholds. If the provider still rejects a request as too long, a separate **reactive** recovery runs (see [03 — Agent loop](03-agent-loop.md) and section 7 below).

## 4. What automatic compaction does

When usage crosses the auto-compact threshold, `autoCompactIfNeeded()` may run before the next model request. The main path is `compactConversation()` in [`compact.ts`](../../../src/services/compact/compact.ts).

Compaction is itself a model call: the runtime asks the model to summarize the conversation so far, then replaces the long history with that summary plus a preserved recent tail.

### The summary is highly structured

The summarization prompt in [`prompt.ts`](../../../src/services/compact/prompt.ts) asks for a private `<analysis>` block followed by a `<summary>` block with numbered sections, including:

1. primary request and intent;
2. key technical concepts;
3. files and code sections examined or changed;
4. errors and fixes;
5. problem solving;
6. **all user messages** (not tool results);
7. pending tasks;
8. current work;
9. an optional next step, with verbatim quotes to avoid drift.

The prompt deliberately preserves user intent and recent work in detail, because those are what the model most needs to continue correctly. The `<analysis>` scratchpad is stripped before the summary enters context.

### It runs as a constrained sub-call

The prompt begins with a strict “respond with text only, do not call tools” preamble and runs with a single turn. This prevents the summary step from wandering off and calling tools, which would waste its one turn.

## 5. What is preserved and restored after compaction

Compaction is not only a summary. `buildPostCompactMessages()` assembles the new conversation as:

```text
compact boundary marker
  + summary messages
  + preserved recent messages (a "tail")
  + attachments (e.g. re-injected files/skills/plan)
  + hook results
```

To avoid losing the most useful concrete context, the runtime re-injects some material under strict budgets (from [`compact.ts`](../../../src/services/compact/compact.ts)):

| Budget constant | Value | Meaning |
|---|---|---|
| `POST_COMPACT_MAX_FILES_TO_RESTORE` | 5 | At most five recently relevant files are re-attached. |
| `POST_COMPACT_MAX_TOKENS_PER_FILE` | 5,000 | Per-file cap for that re-injection. |
| `POST_COMPACT_TOKEN_BUDGET` | 50,000 | Overall budget for restored file content. |
| `POST_COMPACT_MAX_TOKENS_PER_SKILL` | 5,000 | Per-skill cap for preserved skill content. |
| `POST_COMPACT_SKILLS_TOKEN_BUDGET` | 25,000 | Overall budget for preserved skills. |

Files already visible in the preserved tail are skipped so the same content is not injected twice. A plan and invoked-skill content are also preserved through dedicated attachments so compaction does not erase an approved plan or an active skill.

The compact boundary is recorded in the transcript with metadata (`preservedSegment` head/anchor/tail UUIDs) so the session can be reloaded with the correct message chain.

## 6. Lighter-weight alternatives to full compaction

Full compaction is the heaviest option. The runtime has cheaper mechanisms that reduce context without a full summary. Several are feature-gated, and in this checked-in source some are only generated stubs, so their real behavior cannot be verified here.

| Mechanism | Location | Status in this checkout | Idea |
|---|---|---|---|
| Micro-compaction | [`microCompact.ts`](../../../src/services/compact/microCompact.ts) | Real | Clears/shrinks old, low-value tool results (only for a set of “compactable” tools such as Read, shell, Grep, Glob, web, Edit, Write) instead of summarizing everything. |
| Time-based micro-compaction config | [`timeBasedMCConfig.ts`](../../../src/services/compact/timeBasedMCConfig.ts) | Real | Tunes when old tool-result content is cleared, replaced by a short “content cleared” placeholder. |
| Session-memory compaction | [`sessionMemoryCompact.ts`](../../../src/services/compact/sessionMemoryCompact.ts) | Real | An alternative that prunes messages via a session-memory approach; tried before legacy compaction when enabled. |
| Cached micro-compaction | [`cachedMicrocompact.ts`](../../../src/services/compact/cachedMicrocompact.ts) | Stub | Cache-aware micro-compaction (`CACHED_MICROCOMPACT`), implementation not present here. |
| Reactive compaction | [`reactiveCompact.ts`](../../../src/services/compact/reactiveCompact.ts) | Stub | Fires after the provider rejects an over-limit prompt (`REACTIVE_COMPACT`), implementation not present here. |
| History snipping | [`snipCompact.ts`](../../../src/services/compact/snipCompact.ts), [`snipProjection.ts`](../../../src/services/compact/snipProjection.ts) | Stub | Removes selected low-value history (`HISTORY_SNIP`), implementation not present here. |
| Context collapse | [`src/services/contextCollapse/`](../../../src/services/contextCollapse/) | Present | A structured commit/replay system that, when enabled, owns the headroom problem and suppresses ordinary auto-compact. |

“Compactable tools” are defined by the `COMPACTABLE_TOOLS` set in [`compact.ts`](../../../src/services/compact/compact.ts). Micro-compaction targets bulky tool output (like large file reads) because that is where most reclaimable space usually is.

## 7. Proactive versus reactive

There are two moments compaction can happen:

- **Proactive** — before a request, when usage crosses the auto-compact threshold. This is `shouldAutoCompact()` / `autoCompactIfNeeded()`.
- **Reactive** — after the provider rejects a request as too long. The agent loop then attempts recovery (context-collapse drain or reactive compaction) before surfacing the error. See [03 — Agent loop](03-agent-loop.md), section 6.

When context collapse or a reactive-only mode is active, ordinary proactive auto-compact is intentionally suppressed so the two systems do not fight over the same headroom.

## 8. Safety guards

Compaction has to avoid making things worse. Guards in [`autoCompact.ts`](../../../src/services/compact/autoCompact.ts) include:

- **Circuit breaker.** `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`. After three consecutive failures (for example, context that is irrecoverably over the limit), it stops retrying so it does not hammer the provider every turn.
- **Recursion guards.** Compaction and session-memory work run as forked sub-agents; those sources (`session_memory`, `compact`, and the context-collapse agent) do not themselves trigger auto-compact, which would deadlock.
- **Post-compact cleanup.** `runPostCompactCleanup()` resets related state, and cache-break detection is notified so the expected post-compact cache drop is not misreported.

## 9. Turning it off or tuning it

Several environment variables and one setting affect this system (from [`autoCompact.ts`](../../../src/services/compact/autoCompact.ts)):

| Control | Effect |
|---|---|
| `autoCompactEnabled` setting | Master switch for automatic compaction. |
| `DISABLE_COMPACT` | Disables compaction entirely. |
| `DISABLE_AUTO_COMPACT` | Disables automatic compaction but keeps manual `/compact`. |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | Overrides the context-window size used for the calculation. |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | Triggers auto-compact at a percentage of the effective window (testing/tuning). |
| `CLAUDE_CODE_BLOCKING_LIMIT_OVERRIDE` | Overrides the blocking limit. |

Manual `/compact` remains available even when automatic compaction is off, and it uses a smaller buffer (`MANUAL_COMPACT_BUFFER_TOKENS = 3,000`).

## 10. What this means for you as a user

- Long sessions naturally lose fine detail from the model’s active memory; this is expected, not a malfunction.
- Your local transcript usually still contains the original detail even after the model’s context was compacted.
- Recent messages, your stated intent, pending tasks, an approved plan, and a few relevant files are the things the system tries hardest to keep.
- If precise older detail matters, restate it, or point the assistant back to the file or message rather than assuming it still remembers verbatim.
- Compaction costs an extra model call. Very long sessions can therefore be slower and more expensive around a compaction event.

## 11. Where to look in the source

| Topic | File |
|---|---|
| Thresholds, effective window, circuit breaker, env controls | [`autoCompact.ts`](../../../src/services/compact/autoCompact.ts) |
| Summarize-and-preserve, budgets, boundary metadata | [`compact.ts`](../../../src/services/compact/compact.ts) |
| Summarization prompt structure | [`prompt.ts`](../../../src/services/compact/prompt.ts) |
| Micro-compaction of tool results | [`microCompact.ts`](../../../src/services/compact/microCompact.ts) |
| Session-memory compaction | [`sessionMemoryCompact.ts`](../../../src/services/compact/sessionMemoryCompact.ts) |
| Post-compact cleanup | [`postCompactCleanup.ts`](../../../src/services/compact/postCompactCleanup.ts) |
| Context collapse | [`src/services/contextCollapse/`](../../../src/services/contextCollapse/) |
| Loop integration and reactive recovery | [`src/query.ts`](../../../src/query.ts) |
