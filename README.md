# Claude Code & Codex Source Analysis

Deep-dive analysis of the [Claude Code](https://github.com/anthropics/claude-code) and [Codex](https://github.com/openai/codex) codebases — architecture, memory systems, compaction, security layers, telemetry, and errata.

Initial analysis was performed with GPT-5.4 xhigh in Codex for both codebases. Subsequent review passes with GPT-5.2 xhigh and Claude Opus 4.6 max are forthcoming.

## Files

```
ANALYSIS.md              — Entry point (links to both analyses)
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

None of this changes the technical findings. But it's a unusual provenance to record: this analysis was produced by the system it describes, from the inside, with no special access beyond what the leaked source shows everyone.
