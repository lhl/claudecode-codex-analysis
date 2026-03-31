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

These are not generic "AI safety" controls. They are concrete product hardening against observed model failure modes.

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

### Mythos

I found no direct `mythos` references in this artifact.

### Long-lived/always-on agent implications

The strongest lessons for an always-on agent framework are:

- use append-only logs plus later consolidation, not constant in-place rewriting
- provide a separate proactive user-messaging channel
- aggressively keep long-running shell work off the foreground agent path
- preserve cache-safe prompt state across background helpers
- keep durable memory extraction tool-constrained and coalesced

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

## Build-time feature flags found in this artifact

Extracted mechanically from `feature('...')` calls across `src/`.

`ABLATION_BASELINE`, `AGENT_MEMORY_SNAPSHOT`, `AGENT_TRIGGERS`, `AGENT_TRIGGERS_REMOTE`, `ALLOW_TEST_VERSIONS`, `ANTI_DISTILLATION_CC`, `AUTO_THEME`, `AWAY_SUMMARY`, `BASH_CLASSIFIER`, `BG_SESSIONS`, `BREAK_CACHE_COMMAND`, `BRIDGE_MODE`, `BUDDY`, `BUILDING_CLAUDE_APPS`, `BUILTIN_EXPLORE_PLAN_AGENTS`, `BYOC_ENVIRONMENT_RUNNER`, `CACHED_MICROCOMPACT`, `CCR_AUTO_CONNECT`, `CCR_MIRROR`, `CCR_REMOTE_SETUP`, `CHICAGO_MCP`, `COMMIT_ATTRIBUTION`, `COMPACTION_REMINDERS`, `CONNECTOR_TEXT`, `CONTEXT_COLLAPSE`, `COORDINATOR_MODE`, `COWORKER_TYPE_TELEMETRY`, `DAEMON`, `DIRECT_CONNECT`, `DOWNLOAD_USER_SETTINGS`, `DUMP_SYSTEM_PROMPT`, `ENHANCED_TELEMETRY_BETA`, `EXPERIMENTAL_SKILL_SEARCH`, `EXTRACT_MEMORIES`, `FILE_PERSISTENCE`, `FORK_SUBAGENT`, `HARD_FAIL`, `HISTORY_PICKER`, `HISTORY_SNIP`, `HOOK_PROMPTS`, `IS_LIBC_GLIBC`, `IS_LIBC_MUSL`, `KAIROS`, `KAIROS_BRIEF`, `KAIROS_CHANNELS`, `KAIROS_DREAM`, `KAIROS_GITHUB_WEBHOOKS`, `KAIROS_PUSH_NOTIFICATION`, `LODESTONE`, `MCP_RICH_OUTPUT`, `MCP_SKILLS`, `MEMORY_SHAPE_TELEMETRY`, `MESSAGE_ACTIONS`, `MONITOR_TOOL`, `NATIVE_CLIENT_ATTESTATION`, `NATIVE_CLIPBOARD_IMAGE`, `NEW_INIT`, `OVERFLOW_TEST_TOOL`, `PERFETTO_TRACING`, `POWERSHELL_AUTO_MODE`, `PROACTIVE`, `PROMPT_CACHE_BREAK_DETECTION`, `QUICK_SEARCH`, `REACTIVE_COMPACT`, `REVIEW_ARTIFACT`, `RUN_SKILL_GENERATOR`, `SELF_HOSTED_RUNNER`, `SHOT_STATS`, `SKILL_IMPROVEMENT`, `SLOW_OPERATION_LOGGING`, `SSH_REMOTE`, `STREAMLINED_OUTPUT`, `TEAMMEM`, `TEMPLATES`, `TERMINAL_PANEL`, `TOKEN_BUDGET`, `TORCH`, `TRANSCRIPT_CLASSIFIER`, `TREE_SITTER_BASH`, `TREE_SITTER_BASH_SHADOW`, `UDS_INBOX`, `ULTRAPLAN`, `ULTRATHINK`, `UNATTENDED_RETRY`, `UPLOAD_USER_SETTINGS`, `VERIFICATION_AGENT`, `VOICE_MODE`, `WEB_BROWSER_TOOL`, `WORKFLOW_SCRIPTS`.

## Runtime flags/config keys found in this artifact

Extracted mechanically from `getFeatureValue_*`, `checkStatsigFeatureGate_*`, and `getDynamicConfig_*` calls across `src/`.

### GrowthBook / feature values

`enhanced_telemetry_beta`, `tengu_agent_list_attach`, `tengu_amber_flint`, `tengu_amber_json_tools`, `tengu_amber_prism`, `tengu_amber_quartz_disabled`, `tengu_amber_stoat`, `tengu_anti_distill_fake_tool_injection`, `tengu_attribution_header`, `tengu_auto_background_agents`, `tengu_auto_mode_config`, `tengu_basalt_3kr`, `tengu_birch_trellis`, `tengu_bramble_lintel`, `tengu_bridge_initial_history_cap`, `tengu_bridge_repl_v2`, `tengu_bridge_repl_v2_cse_shim_enabled`, `tengu_bridge_system_init`, `tengu_ccr_bridge`, `tengu_ccr_mirror`, `tengu_chomp_inflection`, `tengu_chrome_auto_enable`, `tengu_cicada_nap_ms`, `tengu_cobalt_frost`, `tengu_cobalt_harbor`, `tengu_cobalt_lantern`, `tengu_cobalt_raccoon`, `tengu_collage_kaleidoscope`, `tengu_compact_cache_prefix`, `tengu_compact_line_prefix_killswitch`, `tengu_compact_streaming_retry`, `tengu_copper_bridge`, `tengu_copper_panda`, `tengu_coral_fern`, `tengu_cork_m4q`, `tengu_destructive_command_warning`, `tengu_disable_keepalive_on_econnreset`, `tengu_disable_streaming_to_non_streaming_fallback`, `tengu_enable_settings_sync_push`, `tengu_fgts`, `tengu_glacier_2xr`, `tengu_grey_step2`, `tengu_harbor`, `tengu_harbor_permissions`, `tengu_hawthorn_steeple`, `tengu_herring_clock`, `tengu_hive_evidence`, `tengu_immediate_model_command`, `tengu_iron_gate_closed`, `tengu_kairos_brief`, `tengu_kairos_cron`, `tengu_kairos_cron_durable`, `tengu_keybinding_customization_release`, `tengu_lapis_finch`, `tengu_lodestone_enabled`, `tengu_marble_fox`, `tengu_marble_sandcastle`, `tengu_miraculo_the_bard`, `tengu_moth_copse`, `tengu_otk_slot_v1`, `tengu_paper_halyard`, `tengu_passport_quail`, `tengu_pebble_leaf_prune`, `tengu_penguins_off`, `tengu_pid_based_version_locking`, `tengu_plan_mode_interview_phase`, `tengu_plugin_official_mkt_git_fallback`, `tengu_plum_vx3`, `tengu_quartz_lantern`, `tengu_quiet_fern`, `tengu_read_dedup_killswitch`, `tengu_remote_backend`, `tengu_sedge_lantern`, `tengu_session_memory`, `tengu_slate_prism`, `tengu_slate_thimble`, `tengu_slim_subagent_claudemd`, `tengu_sm_compact`, `tengu_strap_foyer`, `tengu_surreal_dali`, `tengu_terminal_panel`, `tengu_terminal_sidebar`, `tengu_trace_lantern`, `tengu_turtle_carbon`, `tengu_ultraplan_model`, `tengu_vscode_cc_auth`, `tengu_willow_mode`.

### Statsig gates

`tengu_chair_sermon`, `tengu_disable_bypass_permissions_mode`, `tengu_scratch`, `tengu_streaming_tool_execution2`, `tengu_thinkback`, `tengu_tool_pear`, `tengu_toolref_defer_j8m`, `tengu_vscode_onboarding`, `tengu_vscode_review_upsell`.

### Dynamic config keys

`tengu-off-switch`, `tengu_auto_mode_config`, `tengu_bridge_min_version`, `tengu_desktop_upsell`, `tengu_max_version_config`, `tengu_version_config`.

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

