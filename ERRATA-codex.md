# Codex Errata / Bug Notes

These are the issues and rough edges that looked most worth reporting upstream while reviewing the public repo. They are ordered roughly by practical importance, not by certainty that they are already user-visible bugs.

## 1. Raw message history is persisted by default, with an explicit TODO for secret filtering

`message_history.rs` writes raw message text into `~/.codex/history.jsonl`, and the default config is `HistoryPersistence::SaveAll`. The write path also contains a TODO to check the text for sensitive patterns before persisting it.

- Default-on persistence: `codex/codex-rs/core/src/config/types.rs:347-367`
- Raw history file schema and path: `codex/codex-rs/core/src/message_history.rs:1-18`
- Save-all branch and TODO: `codex/codex-rs/core/src/message_history.rs:84-103`

Why it matters:

- This is a privacy/security footgun for anyone who assumes prompts are ephemeral.
- The code is already aware of the risk, but the mitigation is not implemented.

## 2. Remote compaction requests appear to skip `prompt_cache_key`

Normal Responses requests include `prompt_cache_key = conversation_id`, but the compaction request type does not expose that field and the remote compaction payload omits it.

- Normal Responses request includes `prompt_cache_key`: `codex/codex-rs/core/src/client.rs:732-752`, `codex/codex-rs/codex-api/src/common.rs:153-171`
- Compaction payload type has no `prompt_cache_key`: `codex/codex-rs/codex-api/src/common.rs:22-34`
- Compaction request builder omits it: `codex/codex-rs/core/src/client.rs:391-407`

Why it matters:

- Likely cost/latency regression on a hot path.
- Especially relevant because OpenAI-provider compaction defaults to the remote endpoint (`codex/codex-rs/core/src/codex.rs:6215-6235`).

## 3. `apply_patch` safety for `UnlessTrusted` has a self-identified likely logic bug

`assess_patch_safety()` immediately asks the user under `AskForApproval::UnlessTrusted`, but the inline TODO says this is probably wrong and that writable-path checks should happen first.

- Logic and TODO: `codex/codex-rs/core/src/safety.rs:41-53`
- Writable-path auto-approval logic that is currently skipped in that branch: `codex/codex-rs/core/src/safety.rs:61-105`

Why it matters:

- This can cause unnecessary prompts and inconsistent approval behavior.
- The code itself already flags the issue.

## 4. Remote plugin sync currently conflates "disabled" with "uninstalled"

The plugin manager explicitly notes that remote `enabled = false` is currently treated as uninstall rather than a distinct disabled state.

- Comment and behavior: `codex/codex-rs/core/src/plugins/manager.rs:737-742`

Why it matters:

- This is a real state-model mismatch.
- Install state and enable state should not collapse into one bit.

## 5. Analytics queue drops events under backpressure and records no metric for the drop

The analytics queue is bounded. When it fills, events are dropped and only a warning is logged; the TODO says there is no metric yet.

- Queue overflow behavior: `codex/codex-rs/analytics/src/analytics_client.rs:141-164`

Why it matters:

- Silent-ish observability blind spot.
- If analytics reliability matters, this makes overload analysis harder.

## 6. Built-in `explorer` role is advertised as specialized, but its config file is empty

The built-in role registry exposes `explorer` as a special codebase-question role and points it at `explorer.toml`, but the referenced file is empty in this checkout.

- Role registration: `codex/codex-rs/core/src/agent/role.rs:345-368`
- Config-file resolution: `codex/codex-rs/core/src/agent/role.rs:409-417`
- `codex/codex-rs/core/src/agent/builtins/explorer.toml`: empty file

Why it matters:

- The surface presentation suggests stronger specialization than the runtime actually enforces.
- This is probably not a correctness bug, but it is product/implementation drift.

## 7. Unix dangerous-command detection is much narrower than the product surface implies

On Unix, the dangerous-command heuristic boils down to `rm -f`, `rm -rf`, and recursive `sudo`, plus those commands when embedded in plain `bash -lc` scripts.

- Main logic: `codex/codex-rs/shell-command/src/command_safety/is_dangerous_command.rs:7-29`
- Specific destructive cases: `codex/codex-rs/shell-command/src/command_safety/is_dangerous_command.rs:153-165`

Why it matters:

- Broader destructive protection must come from sandboxing, approval policy, exec policy, and guardian review, not from this heuristic.
- That may be intentional, but it is still a gap worth documenting clearly.

## 8. Memory phase-2 handler documents a race around agent-status subscription

The phase-2 consolidation agent handler includes a TODO noting "we might have a very small race here" around subscribing to status after spawn.

- TODO and surrounding logic: `codex/codex-rs/core/src/memories/phase2.rs:349-359`

Why it matters:

- Probably low severity.
- Still worth fixing in a subsystem that is supposed to serialize and safely maintain global memory state.

## 9. `request_permissions` still bypasses guardian auto-review

There is an explicit TODO noting that guardian auto-review does not yet cover `request_permissions` / `with_additional_permissions`; those still route through the manual event path.

- TODO: `codex/codex-rs/core/src/codex.rs:3125-3134`

Why it matters:

- This creates an inconsistent approval story between normal exec approvals and permission-upgrade requests.
- It is more of a missing feature than a bug, but it is exactly the sort of security-surface inconsistency that tends to matter later.

## Summary

The public Codex repo is much cleaner than the Claude Code leak, but the rough edges cluster in predictable places:

- persistence/privacy defaults,
- cache/perf gaps on specialized paths,
- state-model mismatches in plugins,
- thin or incomplete safety/role polish around otherwise strong core designs.

Most of these are reportable without overstating them as catastrophic vulnerabilities.
