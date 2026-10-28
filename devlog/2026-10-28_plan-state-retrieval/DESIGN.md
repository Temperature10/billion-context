# Design

## Extraction

Keep the existing latest-per-pattern scan. At its source-selection boundary, recognize only `TodoWrite` and `todowrite`. Parse a complete JSON object containing a `todos` array, bounded to the existing 4,000-character source limit. Accept nonblank string `content`; `activeForm` and other fields are irrelevant to retrieval. Validate every entry before using any structured content. Claude TodoWrite supports pending/in_progress/completed; OpenCode todowrite also supports cancelled.

Select all in_progress contents if present, otherwise all pending contents. Completed/cancelled entries produce no terms. Valid empty/all-terminal arrays produce an empty source, not fallback and not an earlier snapshot. Other planning tools and the latest user text remain independent sources. Unknown schema, invalid entries, malformed JSON and oversize input fall back to the old capped raw-text tokenization. Only a legitimate leading ACP reference decoration may be removed in this read-only projection; never modify stored or forwarded arguments.

Use the existing task weight 2, user weight 1, tokenizer and block overlap score. Return an optional status-aware marker for recognized snapshots; return null when no terms remain. This avoids a new configuration surface and preserves legacy callers.

## Ranking

Candidate generation remains `core.search(query, state)`. Only when plan-aware is enabled for the current session and candidates exceed the requested limit, re-rank as before. For recognized status snapshots the ordering is:

1. Number of distinct lowercased whitespace-separated query terms present in the block topic/summary, descending.
2. Existing plan overlap score, descending.
3. Original core search order.

This lexical guard prevents an active task from displacing a historical record that matches more of an explicit query. It preserves contiguous CJK phrases and punctuation and counts each query term once, without term-frequency weights. If coverage is tied (including zero), plan ranking decides. Terminal-task documents are never removed from the candidate pool. Fuzzy matches and dependencies not expressible in the plan remain limitations; this is not semantic relevance scoring.

The initial design used a boolean full-query match. Independent review found four new regressions: adding `?`, `behavior`, or `details` could disable that guard for every candidate and let the active task override otherwise clear historical-query evidence. The original 28-case corpus was kept frozen; four separately recorded adversarial cases and a repeated-query-term regression drove the change to distinct-term coverage before any live model run.

## Validation

Tests cover strict status swaps, active-versus-pending, terminal noise, latest empty/all-terminal snapshots, invalid/unknown/custom/incremental/oversized fallbacks, explicit history queries, CJK/punctuation, immutable inputs, disabled/foreign/small-pool paths, and an Anthropic HTTP tool-use fixture with unchanged forwarded arguments and cache breakpoints.

A separately authored fixed corpus compares three arms using real core search. Rank metrics (P@1, recall and MRR at 3) are diagnostic, not task-completion metrics. An optional real-model experiment uses hidden task facts and deterministic output checks, separating fixed-query continuation from autonomous search. No fixture label is revised to hide a regression.

## Sources and tradeoffs

- Claude Agent SDK TodoWrite schema: https://code.claude.com/docs/en/agent-sdk/typescript#todowrite
- OpenCode todowrite snapshot/status schema: https://github.com/anomalyco/opencode/blob/b94a30547f7f677b9b9a7e36d7c66ca1ac14787d/packages/core/src/tool/plugin/todo.ts
- #1419 / #1336 established the existing opt-in deterministic ranker.

Parsing a whole snapshot above 4,000 characters could recover late active tasks but expands the prior processing bound; this first change deliberately retains the bound and records the fallback. Incremental task tools need call/result identity reconstruction and belong in a separate proposal. Reusing existing weights keeps this fix interpretable; it does not establish that those weights are optimal.
