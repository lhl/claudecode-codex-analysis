# Claude Code ERRATA

This file is intentionally narrower than `ANALYSIS.md`. These are the issues that look most worth reporting or following up on.

## Likely current/reportable issues

### 1. Published artifact appears to embed recoverable source via inline sourcemaps

Severity: critical release hygiene issue

Evidence:

- `src/replLauncher.tsx:23` contains an inline `sourceMappingURL=data:application/json...`.
- `src/commands/session/session.tsx:140` does too.
- Decoding the embedded map in `src/commands/session/session.tsx` shows it contains `sourcesContent`.
- I found 552 files in `src/` with inline data-URI sourcemaps.

Why it matters:

- This is likely the direct reason readable source was recoverable from the published package.
- Even if Anthropic intended to ship sourcemaps, embedding `sourcesContent` in inline data URIs is much worse than shipping stripped maps or no maps.

What to report:

- Which published package versions included inline sourcemaps with `sourcesContent`.
- Whether npm publication should strip sourcemaps entirely or at least strip embedded sources.

### 2. Full-compaction fallback exposes tools to the summarizer

Severity: high

Evidence:

- `src/services/compact/compact.ts` prefers a no-tools cache-sharing summarizer.
- When that path fails, the fallback streaming path explicitly constructs a tool list.
- The fallback always includes `FileReadTool`.
- When tool search is enabled, it also includes `ToolSearchTool` and MCP tools.

Why it matters:

- Compaction should be the most context-reduction-focused and injection-resistant path in the system.
- Giving the summarizer tools creates extra failure modes, extra noise, and potentially lets summarization behavior depend on tool availability.
- It also weakens the clean separation between "summarize prior state" and "go gather more state".

What to report:

- Fallback compaction should remain tool-free.
- If a toolful fallback is truly required, it should be explicit and separately instrumented as a degraded mode.

### 3. Session-memory compaction has no real summary boundary after resume

Severity: high for cost/quality, medium for correctness

Evidence:

- `src/services/compact/sessionMemoryCompact.ts` handles a special "resumed session" case.
- If session memory exists but `lastSummarizedMessageId` is missing, the code sets the summary boundary to the last message, then backs into a keep-window from there.
- It logs `tengu_sm_compact_resumed_session`.

Why it matters:

- This means the first post-resume compaction cannot reliably know what was already summarized.
- That likely contributes to "resume is expensive" complaints and weak first compact results after resume.
- The client clearly knows this is a degraded state; it has a dedicated code path for it.

What to report:

- Resume should persist or reconstruct a stronger summarized-boundary marker.
- If that is impossible, resumed sessions may need a more deterministic re-baselining step than the current heuristic.

### 4. Post-compact rehydration undoes a large fraction of compaction

Severity: medium-high product-quality issue

Evidence:

- After generating a compact summary, `src/services/compact/compact.ts` re-adds recently read files, plan files, plan-mode state, invoked skills, deferred tool deltas, agent-listing deltas, MCP instruction deltas, and session-start hook outputs.
- `createPostCompactFileAttachments()` alone can restore up to five recent files under a token budget.

Why it matters:

- The system is paying the cost of compaction and then immediately re-expanding the prompt with a lot of reattached state.
- This is a plausible root cause for user-visible "Claude Code compaction is bad" reports even when the summarizer itself succeeds.

What to report:

- Attachments restored after compaction should be much more selective.
- Some of these should probably be diff-based or lazily recoverable rather than eagerly reattached.

### 5. Destructive-command warnings are informational only

Severity: medium safety/usability gap

Evidence:

- `src/tools/BashTool/destructiveCommandWarning.ts` explicitly says the detection is "purely informational" and "doesn't affect permission logic or auto-approval".
- Call sites are permission-dialog UI components like `src/components/permissions/BashPermissionRequest/BashPermissionRequest.tsx`.

Why it matters:

- Users may read a destructive warning and assume it is an enforcement layer.
- It is not. It is just a warning string attached to permission UI.
- This is fine as a UX feature, but it is easy to overstate what it protects against.

What to report:

- Consider either stronger enforcement hooks for especially destructive classes or clearer product wording that this is advisory, not blocking.

### 6. Resume/cache correctness appears extremely brittle

Severity: medium-high

Evidence:

- `src/utils/sessionRestore.ts` has an explicit fork-session path that reseeds content-replacement records into the new transcript.
- The comment says that without this seed, tool results become `FROZEN`, full content is resent, and the result is "cache miss, permanent overage".
- `src/utils/toolResultStorage.ts` reconstructs replacement state on resume specifically to preserve byte-identical decisions.

Why it matters:

- This is a sign that prompt-cache stability depends on exact transcript-level reconstruction, not just semantic equivalence.
- Systems this brittle tend to regress on resume, rewind, compact, fork, or partial-history edge cases.

What to report:

- The cache-resume contract should be made more explicit and more robust.
- Cache correctness should not hinge on replaying replacement decisions from sidecar records with such high precision.

### 7. External transcript persistence drops attachment-based deltas, likely breaking cache on resume

Severity: high cost/performance issue for `--resume`, medium correctness

Evidence:

- For non-`ant` users, Claude Code deliberately does not persist most `attachment` messages in transcripts (`isLoggableMessage` returns false), citing “sensitive info for training” (`src/utils/sessionStorage.ts:4351-4366`).
- Several cache-stability signals are attachment-based “system reminders” (`deferred_tools_delta`, `agent_listing_delta`, `mcp_instructions_delta`, etc.). They become meta user messages in API normalization and can affect both message merging and the single per-request `cache_control` marker placement (`src/utils/messages.ts:2269-2290`, `src/utils/messages.ts:4178-4231`, `src/services/api/claude.ts:3062-3106`).
- The deferred-tools delta system exists specifically to replace earlier “ephemeral prepends” that busted cache, so it is inherently part of the cache contract (`src/services/api/claude.ts:1327-1345`, `src/utils/attachments.ts:1455-1475`).

Why it matters:

- If attachment-derived prompt prefix state is included in cached API requests but not written to disk, `--resume` cannot reconstruct a byte-identical prefix and will force a full cache miss (one-turn `cache_creation` reprocess) on resume.

What to report:

- Persist enough cache-critical attachment state (even redacted/hashes) to reproduce the cached prefix on resume, or ensure these deltas are injected in a way that is stable across sessions.

### 8. Native client attestation rewrites `cch=00000` in request bytes

Severity: medium (could be high if the replacement is brittle)

Evidence:

- The “attribution header” is embedded as a system-prompt text block, and when `NATIVE_CLIENT_ATTESTATION` is enabled it includes a `cch=00000` placeholder; comments state Bun’s native HTTP stack overwrites the zeros in the serialized request body with a computed hash, pointing to a Zig implementation (`src/constants/system.ts:59-72`, `src/constants/system.ts:81-91`).

Why it matters:

- Post-serialization rewriting is a potential source of byte-level nondeterminism that can break prompt-cache hits and make debugging resume/cache behavior harder.
- If the replacement algorithm is not strict about matching only the intended placeholder, user/system content that includes the sentinel could be mutated.

What to report:

- Confirm the replacement is an exact, unambiguous match and is robust to user/system text containing the sentinel elsewhere.

### 9. Token estimation uses blanket 4/3 multiplier hiding 10-30% errors

Severity: medium (cost/quality)

Evidence:

- `src/services/compact/microCompact.ts:204` pads all token estimates by 4/3 "to be conservative since we're approximating."
- The underlying `estimateMessageTokens()` counts text + thinking + tool_use block character lengths but applies uniform character-to-token density. Text is ~0.25-0.33 tokens/char for English, but tool schemas have highly variable compression and thinking block metadata overhead is not counted.
- Additionally, `IMAGE_MAX_TOKEN_SIZE` is hardcoded at 2000 (`src/services/compact/microCompact.ts:38`), but actual image token costs range 500-4000+.
- `contentSize()` in `src/utils/toolResultStorage.ts:518-529` only counts text block lengths, ignoring JSON structure overhead (~5% undercount).
- `tokenCountWithEstimation` is called BEFORE `stripImagesFromMessages` / `stripReinjectedAttachments`, so the sent-to-API payload is smaller than estimated.

Why it matters:

- Autocompact threshold decisions based on these estimates can fire too early or too late.
- Tool-heavy sessions and image-heavy sessions are the most affected.
- Multiple estimation errors compound — the 4/3 pad doesn't consistently cover them.

### 10. Streaming compaction fallback rate is 2.79% on model 4.6 (vs 0.01% on 4.5)

Severity: medium (architecture/cost)

Evidence:

- `src/services/compact/prompt.ts:19-26` documents that the no-tools forked summarizer sometimes calls tools despite being told not to. With `maxTurns: 1`, one failed tool call = silent failure, triggering the streaming fallback.
- The `NO_TOOLS_PREAMBLE` is repeated as both preamble AND trailer specifically because the 4.6 failure rate is 279× higher than 4.5.

Why it matters:

- The streaming fallback has tool access (FileReadTool, optionally ToolSearchTool and MCP tools), increasing injection surface and making compaction behavior nondeterministic.
- ~3% of all compactions on model 4.6 go through this degraded path.

What to report:

- Consider enforcing tool-free compaction at the API layer rather than relying on prompt instructions.

### 11. GrowthBook can return null/NaN/string values that bypass default checks

Severity: medium (reliability)

Evidence:

- `src/utils/toolResultStorage.ts:65-77` includes defensive checks: "GrowthBook's cache returns `cached !== undefined ? cached : default`, so a flag served as `null` leaks through."
- The code uses optional chaining and `typeof` checks throughout. Similar guards appear in multiple files.

Why it matters:

- GrowthBook has historically shipped corrupted configs. Any code path that trusts GrowthBook values without explicit null/type guards gets wrong behavior silently.
- This is not a Claude Code bug per se, but it means feature-gate values throughout the system are fragile.

### 12. Cache break detection ignores drops under 2,000 tokens

Severity: low-medium

Evidence:

- `src/services/api/promptCacheBreakDetection.ts:117-120` defines `MIN_CACHE_MISS_TOKENS = 2_000`: "Small drops (e.g., a few thousand tokens) can happen due to normal variation and aren't worth alerting on."

Why it matters:

- In a 50K-token session, a 2K drop is 4% of context silently undetected.
- Repeated small drops can compound without triggering any diagnostics.

### 13. Cached microcompact state can be polluted by subagents

Severity: medium

Evidence:

- `src/services/compact/microCompact.ts:271-276`: "Only run cached MC for the main thread to prevent forked agents (session_memory, prompt_suggestion, etc.) from registering their tool_results in the global cachedMCState, which would cause the main thread to try deleting tools that don't exist in its own conversation."

Why it matters:

- If the guard (checking querySource for main thread) is bypassed or refactored incorrectly, `cache_edits` will reference `tool_use_id`s that don't exist in the main thread's transcript, causing unpredictable cache behavior.
- The guard is a string comparison that has already been wrong once (the output-style latent bug).

## Historical/likely-fixed issues documented in comments

These may already be fixed, but they are worth knowing because they show where the system has broken before.

### Cached microcompact used to exclude non-default output styles

Evidence:

- `src/services/compact/microCompact.ts` contains a comment saying cached microcompact's `=== 'repl_main_thread'` check was a latent bug and silently excluded users with non-default output styles.

### Prompt-cache break detection used to grow memory without bounds

Evidence:

- `src/services/api/promptCacheBreakDetection.ts` caps tracked sources to 10 because each entry stores a large diffable string and many subagents otherwise caused indefinite memory growth.

### Cold-cache ant-model alias resolution could crashloop

Evidence:

- `src/main.tsx` contains a long comment explaining why GrowthBook initialization must be awaited for ant model aliases such as `capybara-fast`, otherwise unresolved aliases could 404 and crashloop on fresh pods.

### Autocompact had no circuit breaker, allowing 3,272 consecutive failures per session

Evidence:

- `src/services/compact/autoCompact.ts:257-260` adds `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3` with a comment referencing BQ data from 2026-03-10: 1,279 sessions had 50+ consecutive compaction failures (up to 3,272 in a single session), wasting ~250K API calls/day globally.

### Circular import in compaction caused CI race condition

Evidence:

- `src/services/compact/grouping.ts:18-21` was extracted to break a `compact.ts ↔ compactMessages.ts` cycle (CC-1180). The comment says: "the cycle shifted module-init order enough to surface a latent ws CJS/ESM resolution race in CI shard-2."

## Non-issues or analysis limitations

### Missing Kairos core files in the leak

This is important for analysis, but it is not necessarily a Claude Code product bug. The leaked tree is incomplete; `src/assistant/index.js` and `src/assistant/gate.js` are imported by `src/main.tsx` but absent from this artifact.

### No direct Mythos hits

I found no `mythos` strings in this tree. That absence should not be overinterpreted because the artifact is incomplete and appears closer to an external build than an internal ant build.
