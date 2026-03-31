# Codex Source Analysis

## Scope and confidence

This analysis is based on the public `codex/` GitHub checkout in this workspace, not on OpenAI's private backend services. That matters more here than it did for the Claude Code leak.

- High confidence: client/runtime architecture, tool routing, sandboxing and approvals, memory pipeline, feature flags, plugin/MCP/app plumbing, analytics/feedback surfaces, and most performance tricks implemented in the local Rust crates.
- Medium confidence: the behavior of server-backed endpoints such as `/responses/compact`, auth-gated connectors/apps, and any backend-side prompt/cache heuristics that are only partially visible from the client (`codex/codex-rs/core/src/client.rs:341-409`, `codex/codex-rs/codex-api/src/endpoint/compact.rs:32-58`).
- Low confidence: the quality characteristics of remote compaction itself, any non-public model routing logic, and any server-side security checks that are not mirrored in the open-source client.

The main structural caveat is that "Codex is open source" does not mean every important subsystem is fully open. The client-side compaction pipeline is visible, but the default OpenAI-provider compaction path delegates to a backend endpoint rather than a local prompt-only summarizer (`codex/codex-rs/core/src/compact.rs:50-67`, `codex/codex-rs/core/src/codex.rs:6215-6235`, `codex/codex-rs/core/src/compact_remote.rs:117-166`).

## Executive summary

- Codex is dramatically more deliberate than the Claude Code source leak. The center of gravity is a typed Rust orchestration engine with explicit subsystem boundaries, not a giant prompt plus ad hoc JS glue (`codex/codex-rs/core/src/lib.rs:7-200`).
- Its strongest engineering areas are transport/perf plumbing, sandboxing/approval layering, and the skills/plugins/apps/MCP stack. There are many small "operator-grade" details: websocket prewarm, incremental Responses requests, prompt-cache keys, approval caches, host-approval caches, typed feature metadata, and background memory consolidation (`codex/codex-rs/core/src/client.rs:708-752`, `codex/codex-rs/core/src/session_startup_prewarm.rs:158-240`, `codex/codex-rs/core/src/tools/sandboxing.rs:64-115`, `codex/codex-rs/core/src/tools/network_approval.rs:77-83`, `codex/codex-rs/core/src/tools/network_approval.rs:167-183`).
- The memory system is real, but not "SOTA agent memory" in the research sense. It is a startup pipeline that extracts memories from prior rollouts into a state DB, then consolidates them into on-disk artifacts via a locked-down sub-agent (`codex/codex-rs/core/src/memories/README.md:17-31`, `codex/codex-rs/core/src/memories/README.md:67-126`, `codex/codex-rs/core/src/memories/phase2.rs:262-320`).
- Codex's compaction plumbing is cleaner than Claude Code's, but the default OpenAI path is still partly opaque because it calls `/responses/compact`. The fallback local prompt is extremely generic, so the really important quality question lives on the server side rather than in this repo (`codex/codex-rs/core/src/compact_remote.rs:117-166`, `codex/codex-rs/core/templates/compact/prompt.md:1-9`).
- Security is layered rather than magical. There is no single "mass deletion blocker"; instead Codex combines sandboxing, approval policy, exec-policy rules, safe-command heuristics, network approval, optional guardian review, and user hooks. That is the right overall shape, but the destructive-command heuristic itself is much narrower than the rest of the security story (`codex/codex-rs/core/src/exec_policy.rs:41-97`, `codex/codex-rs/shell-command/src/command_safety/is_safe_command.rs:47-236`, `codex/codex-rs/shell-command/src/command_safety/is_dangerous_command.rs:153-165`).
- I did not find a Claude-style sentiment or "bad words list" analytics subsystem. The visible telemetry is extensive, but it is mostly skill/app/plugin usage analytics, OTEL, and feedback-log upload plumbing (`codex/codex-rs/analytics/src/analytics_client.rs:64-123`, `codex/codex-rs/analytics/src/analytics_client.rs:578-620`, `codex/codex-rs/core/src/otel_init.rs:13-93`, `codex/codex-rs/feedback/src/lib.rs:28-34`, `codex/codex-rs/feedback/src/lib.rs:245-340`).

## What this codebase is

The public repo is a real agent runtime, not just a thin CLI wrapper. `codex-core` exports a large, intentionally separated surface: agent control, compaction, guardian review, MCP, memories, plugins, sandboxing, skills, unified exec, OTEL, rollout persistence, thread management, and more (`codex/codex-rs/core/src/lib.rs:7-200`).

`codex-core` is also explicit about platform sandboxing constraints:

- macOS uses Seatbelt and keeps `.git`, resolved `gitdir`, and `.codex` read-only under workspace-write mode.
- Linux routes between the legacy Landlock model and bubblewrap depending on whether the split filesystem policy can round-trip through the legacy abstraction.
- Windows supports only the subset of split policies that can be enforced without semantic weakening; unsupported cases fail closed.

That platform split is documented unusually clearly in the crate README, which is a useful signal about engineering maturity (`codex/codex-rs/core/README.md:7-61`).

The repo is also strongly Rust-centric. Compared with Claude Code's leaked TypeScript build artifacts, Codex shows much less "prompt-first vibe coding" and much more typed control-plane machinery. The rough edges that do exist tend to be TODOs, semantics mismatches, or unimplemented polish rather than giant undocumented subsystems.

## Architecture and control flow

### The main architectural pattern

Codex is built around typed state machines and reusable services:

- `ToolRouter` maps model output into typed local shell, function, MCP, custom-tool, and tool-search invocations (`codex/codex-rs/core/src/tools/router.rs:115-210`).
- `ToolOrchestrator` owns the main execution pipeline: approval, sandbox selection, first attempt, optional escalation on denial, and managed-network follow-through (`codex/codex-rs/core/src/tools/orchestrator.rs:1-8`, `codex/codex-rs/core/src/tools/orchestrator.rs:119-225`, `codex/codex-rs/core/src/tools/orchestrator.rs:227-320`).
- `ContextManager` owns transcript normalization, token heuristics, rollback, and replacement history state (`codex/codex-rs/core/src/context_manager/history.rs:32-49`, `codex/codex-rs/core/src/context_manager/history.rs:113-166`, `codex/codex-rs/core/src/context_manager/history.rs:212-317`).
- `AgentControl` owns the multi-agent registry and thread lifecycle (`codex/codex-rs/core/src/agent/control.rs:96-109`, `codex/codex-rs/core/src/agent/control.rs:120-252`).

This is one of the biggest differences from Claude Code. The runtime has an actual control plane.

### Hidden gem: `reference_context_item`

The most interesting context-management trick is `reference_context_item`. It acts as the baseline for later context diffs and model-visible settings reinjection. If it is absent, the next turn falls back to full reinjection; rollback can deliberately clear it when mixed contextual bundles would make diffing stale (`codex/codex-rs/core/src/context_manager/history.rs:38-48`, `codex/codex-rs/core/src/context_manager/history.rs:223-250`).

This is a cleaner answer to "how do you preserve stable context across compaction/rollback without constantly replaying everything?" than the Claude Code leak exposed.

### Token counting is heuristic, not exact

Codex is explicit that its client-side token estimation is coarse and byte-based rather than tokenizer-accurate (`codex/codex-rs/core/src/context_manager/history.rs:129-154`). That honesty matters when interpreting any context-limit behavior elsewhere in the system.

## Memory system and long-lived state

### What the memory system actually is

The checked-in memory subsystem is a two-phase startup pipeline:

1. Phase 1 extracts structured memories from prior rollouts into the state DB.
2. Phase 2 consolidates those stage-1 outputs into on-disk memory artifacts and runs a dedicated consolidation agent.

That split is documented directly in both the module docs and the README (`codex/codex-rs/core/src/memories/mod.rs:1-6`, `codex/codex-rs/core/src/memories/README.md:17-31`, `codex/codex-rs/core/src/memories/README.md:67-126`).

### Phase 1: rollout extraction

Phase 1 is not "read `MEMORY.md` and vibesummarize." It has several concrete protections and scale controls:

- bounded startup claim rules in the state DB, including source filters, age windows, idle-time requirements, scan limits, and leases (`codex/codex-rs/core/src/memories/README.md:30-58`, `codex/codex-rs/core/src/memories/phase1.rs:180-219`);
- a fixed concurrency cap of `8` and a default model of `gpt-5.1-codex-mini` at low reasoning effort (`codex/codex-rs/core/src/memories/mod.rs:32-61`, `codex/codex-rs/core/src/memories/phase1.rs:222-252`);
- a structured JSON-schema output contract requiring `rollout_summary`, `rollout_slug`, and `raw_memory` (`codex/codex-rs/core/src/memories/phase1.rs:149-160`);
- secret redaction on all generated memory fields before persistence (`codex/codex-rs/core/src/memories/README.md:43-58`, `codex/codex-rs/core/src/memories/phase1.rs:384-387`).

This is pragmatic engineering rather than frontier memory research, but it is substantially more robust than plain markdown notes.

### Phase 2: consolidation

Phase 2 is the more interesting long-lived-agent piece.

It:

- claims a single global job so consolidation is serialized (`codex/codex-rs/core/src/memories/README.md:71-99`, `codex/codex-rs/core/src/memories/phase2.rs:58-77`);
- computes selection diffs (`added`, `retained`, `removed`) against the last successful selection (`codex/codex-rs/core/src/memories/README.md:100-116`);
- syncs on-disk artifacts under `~/.codex/memories`, including `raw_memories.md` and `rollout_summaries/` (`codex/codex-rs/core/src/memories/mod.rs:27-30`, `codex/codex-rs/core/src/memories/phase2.rs:95-126`);
- spawns a consolidation sub-agent with approvals disabled, network disabled, collab disabled, memory tool disabled, and write access constrained to `codex_home` (`codex/codex-rs/core/src/memories/README.md:89-99`, `codex/codex-rs/core/src/memories/phase2.rs:279-309`);
- heartbeats the global lease while the agent runs (`codex/codex-rs/core/src/memories/phase2.rs:405-454`).

That locked-down consolidation agent is exactly the kind of pattern worth stealing for long-lived frameworks: the agent is powerful enough to rewrite memory artifacts, but its sandbox and feature surface are intentionally tiny.

### Memory trace ingestion

There is also a separate trace-to-memory path. `memory_trace.rs` normalizes raw trace files, tolerates JSON arrays and JSONL-ish inputs, filters for allowed trace item shapes, and sends them to `/v1/memories/trace_summarize` (`codex/codex-rs/core/src/memory_trace.rs:29-74`, `codex/codex-rs/core/src/memory_trace.rs:112-217`).

That is not a grand memory architecture, but it is a useful ingestion primitive for any future "always on" system.

### Assessment for long-lived/general-agent work

The public Codex repo does not expose a Kairos-style always-on assistant or daily-log subsystem. The closest durable-state primitives are:

- rollout persistence and thread/session history,
- the state DB,
- startup memory extraction and consolidation,
- background sub-agents for maintenance tasks.

So the design is relevant to long-lived agents, but it is not itself an always-running general-agent framework. It is closer to "durable interactive coding sessions plus background maintenance workers" than to "resident autonomous assistant."

## Security, steerability, and dangerous behavior mitigation

### The security model is layered

Codex does not rely on a single prompt instruction. Its client-side defenses span:

- static command allowlists and dangerous-command heuristics (`codex/codex-rs/shell-command/src/command_safety/is_safe_command.rs:47-236`, `codex/codex-rs/shell-command/src/command_safety/is_dangerous_command.rs:7-29`, `codex/codex-rs/shell-command/src/command_safety/is_dangerous_command.rs:153-165`);
- exec-policy rules and bounded prefix amendments, with explicit bans on absurdly broad prefixes like `python`, `bash -lc`, `git`, `node -e`, and `sudo` (`codex/codex-rs/core/src/exec_policy.rs:50-97`);
- sandbox selection and retry semantics in the tool orchestrator (`codex/codex-rs/core/src/tools/orchestrator.rs:1-8`, `codex/codex-rs/core/src/tools/orchestrator.rs:174-320`);
- approval caching keyed by request targets (`codex/codex-rs/core/src/tools/sandboxing.rs:39-115`);
- specialized patch safety checks that only auto-approve `apply_patch` when writes stay inside writable roots and an actual sandbox exists (`codex/codex-rs/core/src/safety.rs:27-105`);
- managed network approval with per-host caching (`codex/codex-rs/core/src/tools/network_approval.rs:77-83`, `codex/codex-rs/core/src/tools/network_approval.rs:167-183`);
- optional guardian review for on-request approvals (`codex/codex-rs/core/src/guardian/mod.rs:1-13`).

### Safe-command heuristics are conservative and explicit

The "safe command" side is more sophisticated than the "dangerous command" side. Codex whitelists read-only-ish commands and checks flags for unsafe variants of `find`, `rg`, `git`, `base64`, and `sed -n`, including git global-option abuse cases (`codex/codex-rs/shell-command/src/command_safety/is_safe_command.rs:52-236`).

This is a solid steerability/performance trick: it lets the harness auto-approve lots of obvious read-only work without asking the user every turn.

### Dangerous-command detection is narrower than the rest of the story

The Unix dangerous-command heuristic is intentionally small: it effectively flags `rm -f`, `rm -rf`, and recursive `sudo` cases, plus any dangerous commands inside plain `bash -lc` scripts (`codex/codex-rs/shell-command/src/command_safety/is_dangerous_command.rs:15-29`, `codex/codex-rs/shell-command/src/command_safety/is_dangerous_command.rs:153-165`).

So the answer to "does Codex explicitly prevent mass deletion?" is:

- not with a dedicated broad-spectrum deletion detector;
- mostly through sandboxing, approval prompts, exec policy, and optional guardian review;
- only partly through destructive-command heuristics.

That is a reasonable architecture, but it means the narrow heuristic should not be mistaken for the whole defense model.

### Guardian reviewer: the most interesting safety layer

The guardian subsystem is one of the strongest and most deliberate pieces in the repo.

It:

- reconstructs a compact transcript for review and fails closed on timeout, malformed output, or execution failure (`codex/codex-rs/core/src/guardian/mod.rs:4-13`, `codex/codex-rs/core/src/guardian/review.rs:73-89`, `codex/codex-rs/core/src/guardian/review.rs:137-170`);
- treats transcript, tool arguments, tool results, retry reason, and planned action as untrusted evidence rather than instructions (`codex/codex-rs/core/src/guardian/prompt.rs:56-107`);
- always retains all user turns and then spends bounded budget on recent assistant/tool context (`codex/codex-rs/core/src/guardian/prompt.rs:110-198`);
- allows read-only tool checks to gather local evidence (`codex/codex-rs/core/src/guardian/prompt.rs:99-106`);
- approves only when `risk_score < 80` (`codex/codex-rs/core/src/guardian/mod.rs:11-13`, `codex/codex-rs/core/src/guardian/review.rs:172-209`);
- pins the review session to read-only sandboxing, `approval_policy = never`, and disables collab plus web search (`codex/codex-rs/core/src/guardian/review.rs:246-259`, `codex/codex-rs/core/src/guardian/review_session.rs:636-689`).

That is materially better than just telling the main model "be careful."

### Hooks and project docs are part of steerability

Codex's steerability is layered, not only system-prompt based:

- `AGENTS.md` files are discovered hierarchically from project root to current working directory and concatenated into user instructions (`codex/codex-rs/core/src/project_doc.rs:1-17`, `codex/codex-rs/core/src/project_doc.rs:77-120`, `codex/codex-rs/core/src/project_doc.rs:181-270`);
- feature-specific instruction bundles can be injected, including detailed JS REPL usage guidance and optional child-agent guidance (`codex/codex-rs/core/src/project_doc.rs:43-75`, `codex/codex-rs/core/src/project_doc.rs:101-113`);
- post-tool-use hooks can add extra model context, emit feedback to the model, block, or stop execution (`codex/codex-rs/hooks/src/events/post_tool_use.rs:35-43`, `codex/codex-rs/hooks/src/events/post_tool_use.rs:121-143`, `codex/codex-rs/hooks/src/events/post_tool_use.rs:188-247`).

That hook surface is especially relevant for anyone working on injection resistance or domain-specific guardrails.

## Compaction, context management, caching, and performance tricks

### Context management: better than it looks at first glance

`ContextManager` is doing more than "store a list of messages." It tracks raw items, token info, and the `reference_context_item` baseline for later diffed reinjection (`codex/codex-rs/core/src/context_manager/history.rs:32-49`). That baseline is a hidden gem because it lets Codex rebuild context efficiently after compaction/rollback instead of blindly replaying everything.

### Remote compaction is the default OpenAI-provider path

This is the key compaction nuance.

For OpenAI providers, Codex prefers the remote compaction endpoint rather than the inline prompt summarizer (`codex/codex-rs/core/src/compact.rs:50-52`, `codex/codex-rs/core/src/codex.rs:6215-6235`).

The remote path does the following client-side:

- trims trailing Codex-generated items until the compaction request fits the current context window estimate (`codex/codex-rs/core/src/compact_remote.rs:76-89`, `codex/codex-rs/core/src/compact_remote.rs:273-300`);
- includes the model-visible tool specs in the compaction prompt (`codex/codex-rs/core/src/compact_remote.rs:98-115`);
- calls `compact_conversation_history()` against `/responses/compact` (`codex/codex-rs/core/src/compact_remote.rs:117-139`, `codex/codex-rs/core/src/client.rs:341-409`, `codex/codex-rs/codex-api/src/endpoint/compact.rs:32-58`);
- filters the backend output to drop developer messages and non-user synthetic wrappers while keeping real user messages, assistant messages, compaction items, and hook prompts (`codex/codex-rs/core/src/compact_remote.rs:168-229`);
- re-injects canonical initial context at the correct boundary for mid-turn compaction and preserves ghost snapshots so `/undo` still works after compaction (`codex/codex-rs/core/src/compact_remote.rs:90-96`, `codex/codex-rs/core/src/compact_remote.rs:148-165`, `codex/codex-rs/core/src/compact_remote.rs:168-188`).

This is significantly cleaner than the Claude Code compaction stack. The main limitation is that the actual compaction quality is mostly in the backend endpoint, not in the client repo.

### Inline compaction is the fallback path, and it is very generic

The non-remote/local path is still valuable to inspect because it reveals Codex's fallback philosophy.

Important details:

- mid-turn compaction keeps the compaction summary as the last item and injects initial context just above the last real user message because that ordering matches model expectations (`codex/codex-rs/core/src/compact.rs:35-48`, `codex/codex-rs/core/src/compact.rs:199-216`);
- on compaction overflow it removes the oldest history item and retries, explicitly trying to preserve recent context and prefix-based cache behavior (`codex/codex-rs/core/src/compact.rs:154-163`);
- the replacement history keeps a synthetic summary plus selected user messages, then warns that long threads and repeated compactions reduce accuracy (`codex/codex-rs/core/src/compact.rs:191-231`);
- the actual summarization prompt is only "create a handoff summary" with progress, constraints, next steps, and references (`codex/codex-rs/core/templates/compact/prompt.md:1-9`);
- the summary prefix is similarly generic (`codex/codex-rs/core/templates/compact/summary_prefix.md:1`).

So if you are specifically interested in "why is compaction good or bad?", the client-side answer is:

- Codex's transcript surgery is thoughtful.
- The local fallback prompt is not very sophisticated.
- The default OpenAI-provider path is backend-driven, so the most important part is not actually open here.

### Prompt cache and transport engineering

This repo is full of small performance tricks:

- Normal Responses requests set `prompt_cache_key = conversation_id` (`codex/codex-rs/core/src/client.rs:732-752`, `codex/codex-rs/codex-api/src/common.rs:153-171`).
- Websocket sessions are cached at the model-client level and reused across turn-scoped sessions when possible (`codex/codex-rs/core/src/client.rs:255-317`).
- Websocket requests can become incremental by attaching `previous_response_id` plus only the delta input instead of replaying the whole prompt (`codex/codex-rs/core/src/client.rs:830-856`).
- Request compression defaults to `zstd` when using OpenAI providers with ChatGPT auth (`codex/codex-rs/core/src/client.rs:975-984`).
- Startup prewarm opens the websocket before the first real turn so the first request can consume an already-warmed transport (`codex/codex-rs/core/src/session_startup_prewarm.rs:158-240`).
- Guardian review tries to preserve a stable prompt-cache key by reusing a cached "trunk" review session when possible (`codex/codex-rs/core/src/guardian/review.rs:246-259`).
- Codex apps tools and accessible connectors are cached on disk and in memory to avoid expensive startup blocking (`codex/codex-rs/core/src/mcp_connection_manager.rs:102-107`, `codex/codex-rs/core/src/mcp_connection_manager.rs:115-140`, `codex/codex-rs/core/src/mcp_connection_manager.rs:361-385`, `codex/codex-rs/core/src/connectors.rs:69-91`, `codex/codex-rs/core/src/connectors.rs:141-166`).

These are the kinds of harness-level optimizations Claude Code's leaked codebase handled much less cleanly.

### Important compaction caveat

One likely performance gap remains: normal Responses requests carry a `prompt_cache_key`, but `CompactionInput` does not expose one, and the compact request payload omits it (`codex/codex-rs/codex-api/src/common.rs:22-34`, `codex/codex-rs/codex-api/src/common.rs:153-171`, `codex/codex-rs/core/src/client.rs:391-407`). I list that in `ERRATA-codex.md`.

## Multi-agent design and roles

### The multi-agent system is real

Codex has a proper thread/agent substrate:

- `spawn_agent` enforces depth checks, role application, optional forked context, runtime overrides, and telemetry (`codex/codex-rs/core/src/tools/handlers/multi_agents/spawn.rs:25-169`);
- `AgentControl` manages the live registry, spawn slots, forked rollouts, inherited shell snapshots, and inherited exec policy (`codex/codex-rs/core/src/agent/control.rs:96-109`, `codex/codex-rs/core/src/agent/control.rs:150-252`, `codex/codex-rs/core/src/agent/control.rs:254-338`);
- delegated approvals from sub-agents are routed back through the parent session rather than surfaced directly to the sub-agent consumer (`codex/codex-rs/core/src/codex_delegate.rs:57-62`, `codex/codex-rs/core/src/codex_delegate.rs:105-139`, `codex/codex-rs/core/src/codex_delegate.rs:222-280`).

This is a much more serious collaboration substrate than the typical "sub-agent" marketing wrapper.

### Role specialization is partly prompt/config, not deep runtime enforcement

The interesting caveat is that role specialization is not uniformly deep:

- roles are loaded as config layers at spawn time (`codex/codex-rs/core/src/agent/role.rs:1-7`, `codex/codex-rs/core/src/agent/role.rs:30-75`);
- the built-in `explorer` role is heavily described in the tool-facing metadata (`codex/codex-rs/core/src/agent/role.rs:345-368`);
- but its referenced config file, `builtins/explorer.toml`, is empty in this checkout.

That means the "explorer" specialization is mostly description-level and policy-level rather than some deep hard-coded capability profile. This is not fatal, but it is much thinner than the surface presentation suggests.

### Relevance to always-on agents

The multi-agent substrate is useful for long-lived frameworks because it already has:

- persistent threads,
- fork and resume behavior,
- inherited execution context,
- approval routing,
- rollout persistence.

What it does not expose is an obvious "always-running orchestrator daemon" equivalent. Public Codex is better understood as a persistent multi-threaded interactive runtime than as a resident autonomous agent system.

## Observability, analytics, and feedback

### What Codex explicitly tracks

The visible analytics layer tracks:

- skill invocations,
- app mentions,
- app use,
- plugin use,
- plugin installed/uninstalled/enabled/disabled state changes.

That is all explicit in `analytics_client.rs` (`codex/codex-rs/analytics/src/analytics_client.rs:64-123`, `codex/codex-rs/analytics/src/analytics_client.rs:208-285`).

The event payloads can include:

- thread ID,
- turn ID,
- model slug,
- product client ID,
- repo URL for local skills,
- connector IDs,
- plugin capability metadata.

See `ingest_skill_invoked`, app/plugin metadata builders, and `send_track_events()` (`codex/codex-rs/analytics/src/analytics_client.rs:430-575`, `codex/codex-rs/analytics/src/analytics_client.rs:578-620`).

The POST target is `{base_url}/codex/analytics-events/events` with bearer auth plus `chatgpt-account-id` (`codex/codex-rs/analytics/src/analytics_client.rs:600-609`).

### OTEL and runtime metrics

`otel_init.rs` wires OTEL exporters for Statsig, OTLP HTTP, or OTLP gRPC, and only enables metrics export when analytics are enabled. Runtime metrics themselves are feature-gated (`codex/codex-rs/core/src/otel_init.rs:13-93`).

### Feedback capture is substantial

The feedback system is not just "send a thumbs down." It:

- captures full trace logs into a 4 MiB ring buffer (`codex/codex-rs/feedback/src/lib.rs:28-34`, `codex/codex-rs/feedback/src/lib.rs:63-94`, `codex/codex-rs/feedback/src/lib.rs:161-205`);
- collects structured feedback tags (`codex/codex-rs/feedback/src/lib.rs:82-94`, `codex/codex-rs/feedback/src/lib.rs:96-113`);
- can upload `bug`, `bad_result`, `good_result`, and `safety_check` reports plus attachments to Sentry (`codex/codex-rs/feedback/src/lib.rs:245-340`).

### What I did not find

I did not find a visible Claude-style "sentiment via bad-words list" subsystem in the public Codex repo. The telemetry here is still extensive, but it looks like product instrumentation and bug-report infrastructure rather than local sentiment mining.

## Skills, plugins, apps, MCP, and feature flags

### Skills

The skills subsystem is more polished than the surface docs suggest:

- it resolves env-var dependencies and can prompt the user for missing secret values, storing them in memory for the session only (`codex/codex-rs/core/src/skills.rs:56-169`);
- it tracks implicit skill invocation telemetry (`codex/codex-rs/core/src/skills.rs:171-229`);
- it can prompt to install missing MCP servers required by selected skills, and will auto-install in full-access mode (`codex/codex-rs/core/src/mcp/skill_dependencies.rs:35-41`, `codex/codex-rs/core/src/mcp/skill_dependencies.rs:65-134`, `codex/codex-rs/core/src/mcp/skill_dependencies.rs:136-221`).

Checked-in skills in this repo:

- `babysit-pr`: PR babysitting/watcher workflow (`codex/.codex/skills/babysit-pr/SKILL.md`)
- `remote-tests`: remote test execution guidance (`codex/.codex/skills/remote-tests/SKILL.md`)
- `test-tui`: TUI testing guidance (`codex/.codex/skills/test-tui/SKILL.md`)
- `imagegen`: raster-image generation/editing (`codex/codex-rs/skills/src/assets/samples/imagegen/SKILL.md`)
- `openai-docs`: MCP-first OpenAI docs lookup (`codex/codex-rs/skills/src/assets/samples/openai-docs/SKILL.md`)
- `plugin-creator`: local plugin scaffolding (`codex/codex-rs/skills/src/assets/samples/plugin-creator/SKILL.md`)
- `skill-creator`: skill authoring guidance (`codex/codex-rs/skills/src/assets/samples/skill-creator/SKILL.md`)
- `skill-installer`: skill installation workflow (`codex/codex-rs/skills/src/assets/samples/skill-installer/SKILL.md`)

### Plugins and apps

Plugin mentions are not passive. When a plugin is explicitly named, Codex injects developer hints that point the model at that plugin's visible MCP servers, enabled apps, and skill prefix (`codex/codex-rs/core/src/plugins/injection.rs:13-57`).

Apps and connectors are deeply integrated:

- connectors are cached based on auth/account identity (`codex/codex-rs/core/src/connectors.rs:69-91`, `codex/codex-rs/core/src/connectors.rs:141-166`);
- Codex apps are surfaced through an MCP server, `codex_apps`, with auth-derived bearer/account headers (`codex/codex-rs/core/src/mcp/mod.rs:33-35`, `codex/codex-rs/core/src/mcp/mod.rs:139-151`, `codex/codex-rs/core/src/mcp/mod.rs:165-223`);
- tool names are sanitized to satisfy Responses API naming constraints (`codex/codex-rs/core/src/mcp/mod.rs:36-54`);
- codex-apps tools are cached to disk keyed by account/user identity (`codex/codex-rs/core/src/mcp_connection_manager.rs:102-107`, `codex/codex-rs/core/src/mcp_connection_manager.rs:115-140`, `codex/codex-rs/core/src/mcp_connection_manager.rs:361-385`).

### Feature-flag inventory

The feature registry is centralized and unusually legible. It records:

- the feature key,
- lifecycle stage (`Stable`, `Experimental`, `UnderDevelopment`, `Deprecated`, `Removed`),
- default enablement.

See `Feature`, `Stage`, and the `FEATURES` registry (`codex/codex-rs/features/src/lib.rs:23-40`, `codex/codex-rs/features/src/lib.rs:70-185`, `codex/codex-rs/features/src/lib.rs:521-855`).

Notable default-enabled features in this checkout:

- `shell_tool`
- `unified_exec` on non-Windows
- `shell_snapshot`
- `enable_request_compression`
- `multi_agent`
- `apps`
- `tool_suggest`
- `plugins`
- `skill_mcp_dependency_install`
- `tool_call_mcp_elicitation`
- `personality`
- `fast_mode`

See `codex/codex-rs/features/src/lib.rs:529-551`, `codex/codex-rs/features/src/lib.rs:696-755`, `codex/codex-rs/features/src/lib.rs:790-855`.

Notable default-disabled but interesting features:

- `js_repl`
- `memories`
- `tool_search`
- `image_generation`
- `guardian_approval`
- `artifact`
- `realtime_conversation`
- `default_mode_request_user_input`
- `multi_agent_v2`
- `prevent_idle_sleep`

See `codex/codex-rs/features/src/lib.rs:553-562`, `codex/codex-rs/features/src/lib.rs:618-623`, `codex/codex-rs/features/src/lib.rs:726-783`, `codex/codex-rs/features/src/lib.rs:802-843`.

The main thing to notice is not any single flag; it is how much of the product is feature-driven. The repo is set up to ship a stable core while continuously incubating new collaboration, memory, artifact, and interaction layers.

## Notable hidden gems

These are the implementation ideas I would most want to steal or compare against:

- `reference_context_item` as a baseline for diff-based context reinjection rather than brute-force replay (`codex/codex-rs/core/src/context_manager/history.rs:38-48`);
- guardian trunk-session reuse to keep a stable prompt-cache key for repeated approval reviews (`codex/codex-rs/core/src/guardian/review.rs:246-259`);
- startup websocket prewarm for first-turn latency reduction (`codex/codex-rs/core/src/session_startup_prewarm.rs:158-240`);
- approval caching keyed at the right granularity, including multi-file `apply_patch` approval semantics (`codex/codex-rs/core/src/tools/sandboxing.rs:64-115`, `codex/codex-rs/core/src/tools/sandboxing.rs:239-246`);
- per-host managed-network approvals with session reuse (`codex/codex-rs/core/src/tools/network_approval.rs:77-83`, `codex/codex-rs/core/src/tools/network_approval.rs:167-183`);
- MCP/app tool-name sanitization to meet downstream API constraints (`codex/codex-rs/core/src/mcp/mod.rs:36-54`, `codex/codex-rs/core/src/mcp_connection_manager.rs:136-180`);
- phase-2 memory consolidation using selection diffs and a locked-down internal sub-agent (`codex/codex-rs/core/src/memories/README.md:89-116`, `codex/codex-rs/core/src/memories/phase2.rs:262-320`).

## Bottom line

Compared with Claude Code, Codex looks like the work of an engineering team that cared a lot more about typed boundaries, control-plane rigor, and transport/tooling hygiene.

The big takeaways for agent-harness builders are:

- the security story is strongest when approvals, sandboxing, hooks, and reviewer agents are all first-class;
- memory does not need to be "state of the art" to be useful if extraction, consolidation, and write permissions are carefully structured;
- compaction quality is only partly about the summarization prompt and mostly about transcript surgery, cache stability, and how much opaque backend behavior you are willing to rely on;
- skills/plugins/apps/MCP are not side features here, they are a core extension architecture.

The main limitation is that the most important server-backed parts, especially remote compaction quality, are not fully exposed in the open repo. So Codex is more open than Claude Code, but not completely open in the places that matter most for direct compaction-quality comparison.
