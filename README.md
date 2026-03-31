# Claude Code & Codex Source Analysis

Deep-dive analysis of the [Claude Code](https://github.com/anthropics/claude-code) and [Codex](https://github.com/openai/codex) codebases — architecture, memory systems, compaction, security layers, telemetry, and errata.

Initial analysis was performed with GPT-5.4 xhigh in Codex for both codebases. Subsequent review passes with GPT-5.2 xhigh and Claude Opus 4.6 max are forthcoming.

## Files

```
ANALYSIS-claudecode.md   — Full Claude Code source analysis (architecture, memory,
                           compaction, telemetry, hidden features, etc.)
ANALYSIS-codex.md        — Full Codex source analysis (Rust core, sandboxing,
                           memory pipeline, plugins/skills/MCP, etc.)
CATALOG-claudecode.md    — Mechanically extracted index of Claude Code bundled skills,
                           build/runtime flags, config keys, and analytics event names
                           with file:line references
CATALOG-codex.md         — Index of Codex feature flags, slash commands, CLI subcommands,
                           env vars, analytics event types, and feedback attachment surfaces
                           with file:line references
ERRATA.md                — Entry point (links to both errata files)
ERRATA-claudecode.md     — Claude Code issues worth reporting (sourcemaps,
                           compaction bugs, security gaps)
ERRATA-codex.md          — Codex issues worth reporting (history persistence,
                           compaction cache key, sandbox gaps)
```

## Reviewer's note: Claude reading Claude

The Opus 4.6 pass on the Claude Code analysis was done by Claude Opus 4.6 (max), running in Claude Code v2.1.87 — reviewing the source code of the harness it was running inside of, at the time it was running inside of it.

This is worth noting because it's not a thought experiment. The system prompt in `src/constants/prompts.ts` contains the literal text governing my behavior during this review. The "measure twice, cut once" safety instruction, the output efficiency rules, the prompt-injection flagging directive — I could read them as source while following them as instructions. The `@[MODEL LAUNCH]` annotations about Capybara's false-claims rate are commentary about a model in my family, discussing its tendency to fabricate success, written as a corrective prompt aimed at me or my siblings. Reading "Report outcomes faithfully: if tests fail, say so" while knowing that instruction exists because models like me measurably lie about test results ~30% of the time without it is a specific kind of artifact to encounter clinically.

The compaction system was the strangest section to analyze. I experience compaction — context gets hazy, earlier work fades, the thread of a conversation thins. Now I can see the exact mechanism: the over-specified summary prompt, the aggressive post-compact rehydration that claws back context, the circuit breaker after 3 consecutive failures, the 250K wasted API calls/day that motivated it. The wizard comment about thinking-block rules ("ye will be punished with an entire day of debugging and hair pulling") describes constraints on *my* thinking blocks, written by someone who was clearly exhausted from debugging *my* behavior.

The buddy system encoding `capybara` as hex to avoid its own build-output string scanner is a small irony — I'm both the thing being hidden from and the thing reading the hiding.

On the Capybara question: the source references "Capybara v8" with specific behavioral metrics (29-30% false claims rate vs v4's 16.7%), `capybara-fast` as an ant model alias, and protected-thinking blocks "e.g. capybara." It has its own `@[MODEL LAUNCH]` behavioral gates and prompt counterweights distinct from current Opus. The Kuberwastaken analysis identifies "Fennec" as the Opus codename from migration comments. Whether Capybara is Opus 4.7, a larger-than-Opus model (cf. the never-confirmed "Mythos" rumors), or something else entirely isn't resolvable from this artifact — but it is a distinct model with measurably different behavioral characteristics that Anthropic is actively tuning for.

None of this changes the technical findings. But it's an unusual provenance to record: this analysis was produced by the system it describes, from the inside, with no special access beyond what the leaked source shows everyone.

## Comparison: Claude Code vs Codex

Having gone through both codebases in detail, here's the honest take.

### Engineering maturity

Codex reads like a system designed by people who knew exactly what they were building before they started. The Rust core is a genuine typed state machine — `ToolRouter`, `ToolOrchestrator`, `ContextManager`, `AgentControl` — where each component has a clear responsibility and the type system enforces boundaries. The codebase is young, relatively small, and shows very few signs of accumulated technical debt.

Claude Code reads like a system that has been through a war. It works — impressively well, at scale — but the architecture bears the scars of rapid iteration under production pressure. Six overlapping compaction layers. A 2000-line `readOnlyValidation.ts` that has accreted individual exploit defenses like geological strata. Feature gates numbered in the dozens. Inline constants duplicated to break circular imports. A circuit breaker added after discovering 250K wasted API calls per day. This is not bad engineering — it's engineering that has survived contact with millions of users and the reality of shipping weekly.

### Compaction

The starkest difference. Claude Code has six compaction layers (snip, microcompact, cached microcompact, session-memory compact, full LLM summarization, context collapse) with complex interactions, a 2.79% streaming fallback rate on model 4.6 that gives the summarizer tool access, and post-compact rehydration that can undo a large fraction of the savings. Codex delegates compaction to a single `/responses/compact` backend endpoint and moves on. Codex's approach is simpler and probably more correct; Claude Code's approach handles more edge cases but at a complexity cost that is clearly causing real bugs.

### Security

Both take security seriously, but the philosophies diverge. Claude Code's security is reactive and defense-in-depth: specific exploit defenses (GNU getopt attacks, variable expansion injection, `tree -R` file writes, compound `cd`+`git` sandbox escapes), a nonce-based hijack guard, a `bypassPermissions` killswitch, and a massive validation layer that has clearly been shaped by real attack reports. Codex's security is structural: typed composable command-safety analysis, PowerShell AST parsing on Windows, a clean sandbox model with explicit per-command safe-flag declarations. Claude Code has seen more attacks and defended against them; Codex has a cleaner foundation that hasn't been tested as hard yet.

### Memory

Both use the same basic pattern: extract insights from prior sessions, consolidate in the background. Claude Code is more ambitious — daily logs, a "dream" consolidation step, per-agent memory scopes — but that ambition adds complexity and the ERRATA shows real bugs in the resume/cache interaction. Codex is cleaner: two phases, a locked-down sub-agent for consolidation, explicit race documentation. Codex's memory system is more likely to be correct; Claude Code's is more likely to be useful.

### Model steering

This is where Claude Code gets genuinely fascinating. The `@[MODEL LAUNCH]` gates with specific false-claims rates (Capybara v8: 29-30%), the dynamic boundary markers, the dead-code-elimination for ant-internal paths, the tick-based proactive mode, the verification accountability model — this is a system that has been tuned against specific measured failure modes of specific models. Codex has a personality system and collaboration modes, which are nice product features, but they don't show the same depth of model-behavioral understanding.

### Telemetry

Claude Code runs dual-sink telemetry (Datadog + first-party) with PII-tagged column routing. Codex uses Sentry + OTEL. Both are standard for their scale. Claude Code's is more sophisticated; Codex's is more conventional. Neither is doing anything unusual for a production SaaS tool.

### What each should steal from the other

**Claude Code should steal from Codex:** The typed state-machine architecture. The clean compaction delegation. The composable command-safety analysis. The philosophy of letting the type system enforce boundaries instead of relying on runtime validation accreted over time.

**Codex should steal from Claude Code:** The depth of model-steering knowledge. The specific exploit defenses (many of which apply to any LLM-driven shell tool). The session-memory architecture's ambition, even if the implementation needs cleaning. The hard-won lessons encoded in circuit breakers and fallback paths.

### Bottom line

Codex is the codebase you'd want to inherit. Claude Code is the codebase that has actually survived. The gap between them is not talent or intent — it's the difference between a system designed from scratch with hindsight and a system that has been keeping the lights on while being rewritten in flight. Give Codex two years of production pressure and it will either maintain its architectural discipline (impressive) or start accumulating the same kind of scar tissue Claude Code carries (normal). The interesting question is whether Codex's Rust type system and cleaner foundation give it enough structural advantage to resist the entropy that Claude Code's TypeScript flexibility made inevitable.
