# Claude Code & Codex Source Analysis

Deep-dive analysis of the [Claude Code](https://github.com/anthropics/claude-code) and [Codex](https://github.com/openai/codex) codebases — architecture, memory systems, compaction, security layers, telemetry, and errata.

Initial analysis was performed with GPT-5.4 xhigh in Codex for both codebases. Subsequent review passes with GPT-5.2 xhigh and Claude Opus 4.6 max are forthcoming.

## Files

```
ANALYSIS-claudecode.md   — Full Claude Code source analysis (architecture, memory,
                           compaction, telemetry, hidden features, etc.)
ANALYSIS-codex.md        — Full Codex source analysis (Rust core, sandboxing,
                           memory pipeline, plugins/skills/MCP, etc.)
ERRATA-claudecode.md     — Claude Code issues worth reporting (sourcemaps,
                           compaction bugs, security gaps)
ERRATA-codex.md          — Codex issues worth reporting (history persistence,
                           compaction cache key, sandbox gaps)
```
