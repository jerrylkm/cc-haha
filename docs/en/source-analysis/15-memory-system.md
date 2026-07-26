# 15 — Memory system

This chapter explains how Claude Code can remember things across a conversation and across sessions. It analyzes the real source so the coverage is self-contained.

Relevant source: [`src/memdir/`](../../../src/memdir/), [`src/services/SessionMemory/`](../../../src/services/SessionMemory/), [`src/services/extractMemories/`](../../../src/services/extractMemories/), [`src/services/autoDream/`](../../../src/services/autoDream/), [`src/services/teamMemorySync/`](../../../src/services/teamMemorySync/), [`src/utils/memory/`](../../../src/utils/memory/), and [`sessionMemoryCompact.ts`](../../../src/services/compact/sessionMemoryCompact.ts).

Most of this system is **feature-gated**, so it may be inactive in a given build. Treat it as “stored files on disk,” not human-like recall.

## 1. What “memory” means here

Memory is plain files on disk, not a hidden database. The auto-memory area is a directory named `memory` (`AUTO_MEM_DIRNAME`, [`paths.ts`](../../../src/memdir/paths.ts):92) with an entry file `MEMORY.md` (`AUTO_MEM_ENTRYPOINT_NAME`, [`paths.ts`](../../../src/memdir/paths.ts):93) plus topic files. Paths are resolved in [`paths.ts`](../../../src/memdir/paths.ts) (`getAutoMemPath`, `getAutoMemEntrypoint`); loading and prompt construction are in [`memdir.ts`](../../../src/memdir/memdir.ts).

Key limits from the source:

- `MAX_ENTRYPOINT_LINES` = 200 and `MAX_ENTRYPOINT_BYTES` = 25,000 bound the `MEMORY.md` entry file ([`memdir.ts`](../../../src/memdir/memdir.ts):35,38); content beyond either limit is truncated with a note;
- `MAX_MEMORY_FILES` = 200 bounds how many memory files are scanned into the manifest ([`memoryScan.ts`](../../../src/memdir/memoryScan.ts):21).

There are four cooperating layers, described below.

## 2. Persistent auto-memory (cross-session)

At session start, `loadMemoryPrompt()` ([`memdir.ts`](../../../src/memdir/memdir.ts):419) reads `MEMORY.md` and injects it into the system prompt as a memory section with behavioral rules. This is how long-lived project/user knowledge is carried into a new session.

During a turn, `startRelevantMemoryPrefetch()` ([`src/utils/attachments.ts`](../../../src/utils/attachments.ts):2357) runs asynchronously and, through `getRelevantMemoryAttachments()` (~line 2192), calls `findRelevantMemories()` ([`findRelevantMemories.ts`](../../../src/memdir/findRelevantMemories.ts):39). That function asks a Sonnet model (via `selectRelevantMemories()`, ~line 77) to pick the files that “will clearly be useful … (up to 5)”, excluding `MEMORY.md` (already loaded), and attaches them as a `<system-reminder>` block. This surfaces only what looks relevant instead of dumping all memory into context.

The main agent can also **write** memory directly, because file edit/write permission is granted for auto-memory paths (`isAutoMemPath` in [`paths.ts`](../../../src/memdir/paths.ts)).

## 3. Memory extraction (turn end)

After a model response with no further tool calls, an extraction step can run. [`extractMemories.ts`](../../../src/services/extractMemories/extractMemories.ts) forks an agent with limited permissions (read plus memory-only writes via `createAutoMemCanUseTool`) that scans the memory manifest and updates `MEMORY.md` and topic files. It only runs when there is relevant new material to record.

## 4. Session memory (for compaction)

Session memory summarizes the current session into a file (under `~/.claude/session-memory/…`). [`sessionMemory.ts`](../../../src/services/SessionMemory/sessionMemory.ts) gates extraction (`shouldExtractMemory`, ~line 134) and forks an extraction agent; the prompt enforces `MAX_TOTAL_SESSION_MEMORY_TOKENS` = 12,000 ([`SessionMemory/prompts.ts`](../../../src/services/SessionMemory/prompts.ts):9), instructing the model to condense when the file grows past that budget.

This feeds compaction: `trySessionMemoryCompaction()` ([`sessionMemoryCompact.ts`](../../../src/services/compact/sessionMemoryCompact.ts)) waits for extraction, reads the session summary, and computes which recent messages to keep (respecting minimum/maximum token bounds and API invariants) before replacing older detail with the summary. This is the session-memory path referenced in [12 — Context limits and compaction](12-context-and-compaction.md).

## 5. autoDream (background consolidation)

[`autoDream.ts`](../../../src/services/autoDream/autoDream.ts) is a background consolidation step gated by both time and session count. The defaults come from a remote config (`tengu_onyx_plover`): `minHours: 24` (~line 64) and a minimum session count, with a scan throttle `SESSION_SCAN_INTERVAL_MS` = 10 minutes (~line 56) so the time-gate does not re-fire every turn. When the gate opens, it acquires a consolidation lock (mutual exclusion), forks a consolidation agent that reads recent session transcripts, and distills them into `MEMORY.md` and topic files, rolling back the lock on failure. Conceptually, it periodically “tidies” accumulated session history into durable memory.

## 6. Team memory sync

[`teamMemorySync/`](../../../src/services/teamMemorySync/) can synchronize memory across an organization, scoped to a repository. Before uploading, [`secretScanner.ts`](../../../src/services/teamMemorySync/secretScanner.ts) scans for credentials and blocks the push if secrets are detected. Team memory is opt-in and authenticated.

## 7. Safety and trust boundaries

- Auto-memory paths configured in settings are trusted only from user/local/flag/policy sources; **project-committed settings are excluded** so a malicious repository cannot redirect memory writes ([`paths.ts`](../../../src/memdir/paths.ts)).
- Path handling supports `~/` expansion but rejects unsafe forms.
- Extraction/consolidation agents run with **read plus memory-write only**, not full tool access.
- Team memory upload is gated behind secret scanning.

## 8. What this means for you as a user

- Memory is stored as readable files (mostly under `~/.claude/`); you can inspect or delete them.
- The assistant does not truly “remember” — it re-reads memory files that were written earlier; if something important was never written to memory, it will not persist.
- Relevant-memory prefetch means the model may receive memory content you did not paste; that content is sent to the provider like any other context.
- Team memory shares knowledge with your organization; the secret scanner reduces but does not eliminate the risk of leaking sensitive data, so avoid putting secrets in memory files.
- Because the whole system is feature-gated, a given build may not run some or all of these layers.

## 9. Source reference (line-level)

| Concern | Symbol / constant | Location |
|---|---|---|
| Memory dir + entry names | `AUTO_MEM_DIRNAME` (`memory`), `AUTO_MEM_ENTRYPOINT_NAME` (`MEMORY.md`) | [`paths.ts`](../../../src/memdir/paths.ts):92,93 |
| Entry-file limits | `MAX_ENTRYPOINT_LINES` (200), `MAX_ENTRYPOINT_BYTES` (25,000) | [`memdir.ts`](../../../src/memdir/memdir.ts):35,38 |
| Manifest cap | `MAX_MEMORY_FILES` (200) | [`memoryScan.ts`](../../../src/memdir/memoryScan.ts):21 |
| Load into prompt | `loadMemoryPrompt` | [`memdir.ts`](../../../src/memdir/memdir.ts):419 |
| Relevant-memory select | `findRelevantMemories`, `selectRelevantMemories` (up to 5, Sonnet) | [`findRelevantMemories.ts`](../../../src/memdir/findRelevantMemories.ts):39,77 |
| Prefetch + attach | `startRelevantMemoryPrefetch`, `getRelevantMemoryAttachments` | [`attachments.ts`](../../../src/utils/attachments.ts):2357,2192 |
| Session-memory budget | `MAX_TOTAL_SESSION_MEMORY_TOKENS` (12,000) | [`SessionMemory/prompts.ts`](../../../src/services/SessionMemory/prompts.ts):9 |
| Session-memory compaction | `trySessionMemoryCompaction` | [`sessionMemoryCompact.ts`](../../../src/services/compact/sessionMemoryCompact.ts) |
| autoDream gates | `minHours` (24), `SESSION_SCAN_INTERVAL_MS` (10 min) | [`autoDream.ts`](../../../src/services/autoDream/autoDream.ts):64,56 |
| Team-memory secret scan | `scanForSecrets` | [`secretScanner.ts`](../../../src/services/teamMemorySync/secretScanner.ts) |

Line numbers are approximate; symbol names and constants are the durable anchors.

## 10. Real vs stub

The `memdir` files, session memory, extract memories, autoDream, team memory sync, and session-memory compaction are real source in this checkout (with one stub inside `memdir`). Their activation depends on feature gates and settings.
