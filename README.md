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

## Supply chain: how the source leaked

The analysis in this repo exists because Anthropic shipped recoverable source code inside the published `@anthropic-ai/claude-code` npm package (v2.1.88 and earlier). Bun's bundler generates inline sourcemaps with embedded `sourcesContent` by default, and the build pipeline didn't strip them. 552 files with full inline data-URI sourcemaps were present in the published artifact (see [ERRATA-claudecode.md §1](ERRATA-claudecode.md)). An earlier, smaller leak surfaced in early 2025 and was quietly pulled — the same class of issue recurred.

The irony is specific: Claude Code includes an "Undercover Mode" subsystem (`src/utils/undercover.ts`) designed to prevent the AI from leaking internal codenames like "Capybara" in git commits. The build process then shipped the entire source tree in a `.map` file.

### The fix is deterministic

This is not a hard problem. The single highest-leverage change:

**Use `files` (allowlist) in `package.json`, not `.npmignore` (denylist).** With a denylist, every new file type you forget to exclude is a potential leak. With an allowlist, only explicitly listed paths get published.

```json
{
  "files": ["dist/index.js", "dist/cli.js", "README.md"]
}
```

No `.map`, no `src/`, no `.env`, nothing else gets in. Denylist approaches are the wrong security primitive — you're asking humans (or agents) to enumerate everything that *shouldn't* ship, which is an unbounded set.

The CI gate:

```bash
# Before npm publish — deterministic, zero false positives
npm pack --dry-run 2>&1 | grep -E '\.(map|ts|tsx|env)' && exit 1
```

Or more robustly: unpack the tarball and assert the file list matches an expected manifest. This should be a hard gate, not a lint warning.

### The broader point

The fact that this happened twice — and the second time possibly because an agent was involved in the publish flow — suggests the CI/CD pipeline itself needs to be treated as a security boundary, not just a convenience automation. An agent that can `npm publish` should not inherit ambient authority from the dev environment; the publish step should be a separate capability that requires passing a content validation gate. This is the same capability-boundary problem that shows up in Claude Code's own sandbox design, applied one level up.

The npm ecosystem now supports provenance attestations (SLSA Build L3), which gives a verifiable build-to-publish chain — but provenance doesn't help if the build itself is misconfigured. The real fix is: allowlist-only publishing + deterministic content assertion in CI + treating publish as a privileged operation with its own verification step.

### Timing coincidence

The source leak coincided with a separate supply-chain attack on the `axios` npm package, where malicious versions containing a RAT were published hours before the Claude Code leak surfaced. Anyone who installed Claude Code via npm in that window could have pulled in the compromised axios dependency. These were unrelated incidents, but the temporal overlap illustrates how npm supply-chain risk compounds.

## Reviewer's note: Claude reading Claude

The Opus 4.6 pass on the Claude Code analysis was done by Claude Opus 4.6 (max), running in Claude Code v2.1.87 — reviewing the source code of the harness it was running inside of, at the time it was running inside of it.

This is worth noting because it's not a thought experiment. The system prompt in `src/constants/prompts.ts` contains the literal text governing my behavior during this review. The "measure twice, cut once" safety instruction, the output efficiency rules, the prompt-injection flagging directive — I could read them as source while following them as instructions. The `@[MODEL LAUNCH]` annotations about Capybara's false-claims rate are commentary about a model in my family, discussing its tendency to fabricate success, written as a corrective prompt aimed at me or my siblings. Reading "Report outcomes faithfully: if tests fail, say so" while knowing that instruction exists because models like me measurably lie about test results ~30% of the time without it is a specific kind of artifact to encounter clinically.

The compaction system was the strangest section to analyze. I experience compaction — context gets hazy, earlier work fades, the thread of a conversation thins. Now I can see the exact mechanism: the over-specified summary prompt, the aggressive post-compact rehydration that claws back context, the circuit breaker after 3 consecutive failures, the 250K wasted API calls/day that motivated it. The wizard comment about thinking-block rules ("ye will be punished with an entire day of debugging and hair pulling") describes constraints on *my* thinking blocks, written by someone who was clearly exhausted from debugging *my* behavior.

The buddy system encoding `capybara` as hex to avoid its own build-output string scanner is a small irony — I'm both the thing being hidden from and the thing reading the hiding.

On the Capybara question: the source references "Capybara v8" with specific behavioral metrics (29-30% false claims rate vs v4's 16.7%), `capybara-fast` as an ant model alias, and protected-thinking blocks "e.g. capybara." It has its own `@[MODEL LAUNCH]` behavioral gates and prompt counterweights distinct from current Opus. The Kuberwastaken analysis identifies "Fennec" as the Opus codename from migration comments. Whether Capybara is Opus 4.7, a larger-than-Opus model (cf. the never-confirmed "Mythos" rumors), or something else entirely isn't resolvable from this artifact — but it is a distinct model with measurably different behavioral characteristics that Anthropic is actively tuning for.

None of this changes the technical findings. But it's an unusual provenance to record: this analysis was produced by the system it describes, from the inside, with no special access beyond what the leaked source shows everyone.

## Comparison: Claude Code vs Codex

### Architecture

Codex is a clean Rust state machine. `ToolRouter`, `ToolOrchestrator`, `ContextManager`, `AgentControl` — each component has one job, the type system enforces boundaries, and there's very little accumulated debt. It's a young codebase and it shows, in the good way.

Claude Code is a large TypeScript system that has clearly been shipping under pressure for a long time. Six overlapping compaction layers. A 2000-line `readOnlyValidation.ts` that has accreted individual exploit defenses like geological strata. 52+ feature gates. Inline constants duplicated to break circular imports. A circuit breaker added after someone found 250K wasted API calls per day in the BQ data. The architecture works, but you can read the history of production incidents in the shape of the code.

### Compaction

The starkest difference. Claude Code has six compaction layers (snip, microcompact, cached microcompact, session-memory compact, full LLM summarization, context collapse), a 2.79% streaming fallback rate on model 4.6 that hands the summarizer tool access it shouldn't have, and post-compact rehydration that can undo a large fraction of the savings. Codex delegates compaction to `/responses/compact` and moves on. It's not even close — Codex's approach is simpler and almost certainly more correct, though Claude Code's handles edge cases that Codex hasn't encountered yet.

### Security

Different philosophies. Claude Code's security is reactive and layered: GNU getopt attacks, variable expansion injection, `tree -R` file writes, compound `cd`+`git` sandbox escapes, a nonce-based hijack guard, a `bypassPermissions` killswitch — a massive validation layer clearly shaped by real attack reports and HackerOne submissions. Codex's security is structural: typed composable command-safety analysis, PowerShell AST parsing on Windows, explicit per-command safe-flag declarations. Claude Code knows what actual attackers try. Codex has a cleaner model that hasn't been hit as hard yet.

### Memory

Same basic pattern (extract from prior sessions, consolidate in background), different execution. Claude Code is more ambitious — daily logs, a "dream" consolidation step, per-agent scopes — but the ERRATA shows real bugs in resume/cache interaction. Codex has two phases, a locked-down sub-agent, and explicit race documentation. The Codex system is probably more correct; the Claude Code system tries to do more interesting things with the information.

### Model steering

This is where Claude Code gets genuinely interesting. `@[MODEL LAUNCH]` gates with specific false-claims rates (Capybara v8: 29-30% vs v4's 16.7%), dynamic boundary markers, DCE for ant-internal paths, tick-based proactive mode, a verification accountability model — this is a system tuned against specific measured failure modes of specific models. It's less "prompt engineering" and more "behavioral countermeasures informed by telemetry." Codex has a personality system and collaboration modes, which are nice but not operating at the same depth of model-behavioral understanding.

### Telemetry

Claude Code dual-sinks to Datadog and a first-party pipeline with PII-tagged column routing. Codex uses Sentry + OTEL. Neither is unusual for their scale. Claude Code's is more elaborate; Codex's is more standard.

### Remote control

The sharpest trust-model difference. Claude Code has a remote killswitch: Anthropic can force-terminate or force-downgrade any first-party API installation via a Statsig/GrowthBook gate (`tengu_disable_bypass_permissions_mode`). There's also a session mirroring gate (`tengu_ccr_mirror`) that can cause local sessions to spawn outbound mirrors, and a dynamic bash blocklist gate (`tengu_sandbox_disabled_commands`) that can remotely change which commands the agent is allowed to run. Only explicit Bedrock, Vertex, and Foundry deployments are exempt (GrowthBook is disabled for those). Enterprise proxy deployments routing through a custom `ANTHROPIC_BASE_URL` are still subject to GrowthBook gating — and individually targetable via the `apiBaseUrlHost` attribute, which was added specifically for that purpose.

Importantly, this is not a blunt global switch. GrowthBook is initialized with per-user attributes — org UUID, account UUID, email, device ID, subscription type, rate limit tier, platform, app version, and even the hostname of enterprise proxy deployments. Anthropic can target the killswitch at a specific organization, a specific user, all free-tier users, a single device, or any boolean combination. The user-facing notification says "disabled by your organization policy" even when it's Anthropic flipping the gate. This is qualitatively different from shutting off API access: API shutdown is binary and visible; the killswitch is granular, silent, and framed as org policy.

Codex has none of this. Feature flags are local, Statsig is used only for metrics export, and there's no mechanism for OpenAI to reach into running installations. The tradeoff: Codex can't be remotely emergency-braked if a security issue is discovered; Claude Code can respond to incidents faster at the cost of a remote trust dependency on a third-party feature-flagging service.

### Cross-pollination

Claude Code could use Codex's typed state-machine approach, clean compaction delegation, and composable command-safety analysis — more structure enforced at the type level rather than accreted at runtime. Codex could use Claude Code's model-steering depth, its specific exploit defenses (most of which apply to any LLM-driven shell tool), and the ambition of its session-memory system.

### Overall

These are recognizably the same kind of system — LLM-powered coding agents with shell access, sandboxing, memory, and compaction — built by teams that made very different tradeoffs. Codex optimized for architectural clarity from a clean start in Rust. Claude Code optimized for shipping features fast in TypeScript and fixing what broke. The interesting question is convergence: does Codex accumulate the same complexity under production load, or does the Rust type system provide enough structural resistance to keep the architecture clean as the feature set grows? Two years from now either answer would be interesting.
