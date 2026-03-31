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

## Non-issues or analysis limitations

### Missing Kairos core files in the leak

This is important for analysis, but it is not necessarily a Claude Code product bug. The leaked tree is incomplete; `src/assistant/index.js` and `src/assistant/gate.js` are imported by `src/main.tsx` but absent from this artifact.

### No direct Mythos hits

I found no `mythos` strings in this tree. That absence should not be overinterpreted because the artifact is incomplete and appears closer to an external build than an internal ant build.
