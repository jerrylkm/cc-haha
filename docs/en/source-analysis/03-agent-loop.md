# 03 — Agent loop

> **Beginner note:** You can skip this chapter unless you want to know why one request can produce many steps, why the program retries, or why a long conversation seems to “forget” details.

## The simple version

The AI repeatedly chooses between two things:

1. answer you;
2. ask the local program to perform an action and then look at the result.

The “agent loop” is the code that keeps this process organized. It also prevents endless retries, shortens conversations that exceed model limits, and makes sure every requested tool receives a corresponding result.

## 1. The state machine

[`src/query.ts`](../../../src/query.ts) implements one turn as an explicit loop. Its carried state includes:

- messages;
- current `ToolUseContext`;
- auto-compaction tracking;
- max-output-token recovery count;
- whether reactive compaction was already attempted;
- temporary max-token override;
- pending tool-use summary;
- stop-hook state;
- turn count and the last transition reason.

Keeping these fields together prevents retry branches from accidentally resetting one guard while preserving another.

## 2. Before each model call

The loop performs a sequence of context-size and compatibility operations:

1. select queued input at the allowed priority;
2. normalize API-visible messages;
3. budget or persist oversized tool results;
4. optionally snip old history;
5. apply cached micro-compaction;
6. apply context-collapse state;
7. trigger automatic compaction when thresholds demand it;
8. attach current user/system context;
9. select model, thinking, task budget, tools, and request options.

Several mechanisms can coexist because they solve different problems. They run in a deliberate order so that the cheapest, most granular reductions are tried first and the most lossy summarization is used only as a last resort:

1. **first, tool-result budgeting** controls individual large results;
2. **then snipping** removes low-value old material;
3. **then micro-compaction** makes small cache-aware reductions;
4. **then context collapse** stores/replays a more structured collapsed history;
5. **finally, auto-compaction** summarizes old conversation as a fallback. It runs last on purpose: if the earlier steps already brought the conversation under the threshold, auto-compaction is a no-op and the granular context is preserved instead of being replaced by a single summary.

For a user, all five mechanisms mean the same broad thing: the model cannot hold unlimited information, so the program decides what to keep in active working context. The original local transcript may contain more detail than the model currently remembers.

## 3. Streaming

The API layer emits request-start, message-start, content-block, delta, stop, and completed message events. The query loop both yields events and accumulates complete assistant messages for later tool/recovery decisions.

Recoverable errors are sometimes retained internally but not yielded yet. This is important for desktop/SDK consumers that terminate a session as soon as they see an error.

## 4. Tool scheduling

[`toolOrchestration.ts`](../../../src/services/tools/toolOrchestration.ts) partitions tool calls in original order:

- adjacent concurrency-safe calls share a batch;
- an unsafe call occupies its own batch;
- safe calls after an unsafe call start a new batch.

This preserves model order while allowing independent reads/searches to overlap. The default parallel limit is ten and can be configured with `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY`.

You may therefore see several read/search indicators at once. Changes are treated more cautiously and are normally run one after another.

If `isConcurrencySafe()` throws, the call is treated as unsafe. This conservative fallback avoids turning parser or tool bugs into concurrent mutation.

Serial execution may produce a context modifier. The modified context is passed to the next tool, which matters when an earlier call grants a permission, changes file state, or alters another turn-scoped facility.

## 5. Streaming tool execution

[`StreamingToolExecutor`](../../../src/services/tools/StreamingToolExecutor.ts) can start a tool while later model blocks are still streaming. It maintains cancellation and fallback bookkeeping so results are discarded if the stream is retried. A tool-abort signal can bubble to the query controller unless the reason is merely a sibling failure.

This feature improves latency but has stricter correctness requirements than waiting for the complete assistant message.

## 6. Recovery paths

### Prompt too long

The loop can:

1. drain an available context-collapse result;
2. attempt reactive compaction once;
3. surface the original error if recovery is exhausted.

Flags prevent repeatedly draining or compacting the same failure.

### Maximum output tokens

The runtime can first increase the output limit when the model supports it. If the response is still cut off, it appends a hidden instruction asking the model to resume directly and split the remaining work. Multi-turn recovery is capped at three attempts.

The intermediate error is withheld from clients until this recovery either succeeds or is exhausted.

This can look like the assistant paused and then continued without showing an error. The hidden continuation is deliberate.

### Oversized media

When enabled, media recovery strips or compacts problematic image content and retries. If recovery cannot produce a valid request, the original error is surfaced.

### Model fallback

Rate-limit/provider logic can throw [`FallbackTriggeredError`](../../../src/services/api/withRetry.ts). The loop switches to the configured fallback once, strips model-specific thinking signatures when required, emits a user-visible system notice, and retries. It does not recurse indefinitely.

### Streaming fallback

If a streamed attempt must be replaced, tombstones remove partial assistant content and any prematurely started tool results are discarded. This prevents two contradictory versions of one answer from entering the transcript.

## 7. Stop hooks

After an apparently final answer, [`handleStopHooks()`](../../../src/query/stopHooks.ts) can:

- allow completion;
- prevent further continuation;
- return blocking errors that are appended as messages, causing another model iteration.

The reactive-compaction guard is deliberately preserved through a blocking stop-hook retry. Resetting it could create a compact/error/hook loop.

## 8. Token budgets

Two similarly named mechanisms are distinct:

- API `task_budget` tracks output allowance across the whole agentic turn;
- the feature-gated turn token budget can inject a continuation nudge or stop when work shows diminishing returns.

Usage spent before compaction is still accounted for; compaction is not a way to reset the budget.

## 9. Completion reasons

The loop returns a terminal reason rather than assuming every exit is success. Reasons cover normal completion, blocking/max-turn limits, model errors, streaming aborts, prompt length, image errors, and stop-hook prevention. The outer `query()` marks consumed queued-command UUIDs complete only when the inner generator returns normally.

The UI may turn these internal reasons into a friendly message, retry, or ordinary completion. They are useful when diagnosing why a turn ended early.

## 10. Important invariants

- A tool request must receive a corresponding tool result or explicit interruption.
- Thinking blocks must stay with the trajectory in which they were produced.
- API-bound input is not mutated for observer compatibility; observable copies may receive backfilled fields.
- Recoverable errors remain hidden until recovery has genuinely failed.
- Unsafe tools never gain concurrency because of an exception.
- Retry guards survive the branches where resetting them would permit a spiral.
