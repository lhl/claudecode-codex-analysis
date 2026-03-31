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

## 2. `/feedback` connectivity diagnostics include raw proxy/base-url env var values (no redaction)

`FeedbackDiagnostics` collects proxy environment variables and `OPENAI_BASE_URL`, then includes them verbatim in the consent UI and the uploaded `codex-connectivity-diagnostics.txt` attachment when logs are enabled.

- Env var capture and attachment text: `codex/codex-rs/feedback/src/feedback_diagnostics.rs:3-77`
- Tests demonstrate credential-shaped values are preserved verbatim (for example, `https://user:password@...`): `codex/codex-rs/feedback/src/feedback_diagnostics.rs:111-145`
- Consent UI renders the details and lists the attachment: `codex/codex-rs/tui/src/bottom_pane/feedback_view.rs:500-571`
- Upload path attaches the diagnostics file when `include_logs = true`: `codex/codex-rs/feedback/src/lib.rs:176-214`, `codex/codex-rs/feedback/src/lib.rs:245-340`

Why it matters:

- Real-world proxy env vars often embed credentials or tokens in the URL; this behavior can leak them into feedback uploads.
- Even with consent, "verbatim env var dump" is a sharp edge. At minimum it should redact URL userinfo and common query-token shapes.

## 3. Remote compaction requests appear to skip `prompt_cache_key`

Normal Responses requests include `prompt_cache_key = conversation_id`, but the compaction request type does not expose that field and the remote compaction payload omits it.

- Normal Responses request includes `prompt_cache_key`: `codex/codex-rs/core/src/client.rs:732-752`, `codex/codex-rs/codex-api/src/common.rs:153-171`
- Compaction payload type has no `prompt_cache_key`: `codex/codex-rs/codex-api/src/common.rs:22-34`
- Compaction request builder omits it: `codex/codex-rs/core/src/client.rs:391-407`

Why it matters:

- Likely cost/latency regression on a hot path.
- Especially relevant because OpenAI-provider compaction defaults to the remote endpoint (`codex/codex-rs/core/src/codex.rs:6215-6235`).

## 4. `apply_patch` safety for `UnlessTrusted` has a self-identified likely logic bug

`assess_patch_safety()` immediately asks the user under `AskForApproval::UnlessTrusted`, but the inline TODO says this is probably wrong and that writable-path checks should happen first.

- Logic and TODO: `codex/codex-rs/core/src/safety.rs:41-53`
- Writable-path auto-approval logic that is currently skipped in that branch: `codex/codex-rs/core/src/safety.rs:61-105`

Why it matters:

- This can cause unnecessary prompts and inconsistent approval behavior.
- The code itself already flags the issue.

## 5. Remote plugin sync currently conflates "disabled" with "uninstalled"

The plugin manager explicitly notes that remote `enabled = false` is currently treated as uninstall rather than a distinct disabled state.

- Comment and behavior: `codex/codex-rs/core/src/plugins/manager.rs:737-742`

Why it matters:

- This is a real state-model mismatch.
- Install state and enable state should not collapse into one bit.

## 6. Analytics queue drops events under backpressure and records no metric for the drop

The analytics queue is bounded. When it fills, events are dropped and only a warning is logged; the TODO says there is no metric yet.

- Queue overflow behavior: `codex/codex-rs/analytics/src/analytics_client.rs:141-164`

Why it matters:

- Silent-ish observability blind spot.
- If analytics reliability matters, this makes overload analysis harder.

## 7. Built-in `explorer` role is advertised as specialized, but its config file is empty

The built-in role registry exposes `explorer` as a special codebase-question role and points it at `explorer.toml`, but the referenced file is empty in this checkout.

- Role registration: `codex/codex-rs/core/src/agent/role.rs:345-368`
- Config-file resolution: `codex/codex-rs/core/src/agent/role.rs:409-417`
- `codex/codex-rs/core/src/agent/builtins/explorer.toml`: empty file

Why it matters:

- The surface presentation suggests stronger specialization than the runtime actually enforces.
- This is probably not a correctness bug, but it is product/implementation drift.

## 8. Unix dangerous-command detection is much narrower than the product surface implies

On Unix, the dangerous-command heuristic boils down to `rm -f`, `rm -rf`, and recursive `sudo`, plus those commands when embedded in plain `bash -lc` scripts.

- Main logic: `codex/codex-rs/shell-command/src/command_safety/is_dangerous_command.rs:7-29`
- Specific destructive cases: `codex/codex-rs/shell-command/src/command_safety/is_dangerous_command.rs:153-165`

Why it matters:

- Broader destructive protection must come from sandboxing, approval policy, exec policy, and guardian review, not from this heuristic.
- That may be intentional, but it is still a gap worth documenting clearly.

## 9. Memory phase-2 handler documents a race around agent-status subscription

The phase-2 consolidation agent handler includes a TODO noting "we might have a very small race here" around subscribing to status after spawn.

- TODO and surrounding logic: `codex/codex-rs/core/src/memories/phase2.rs:349-359`

Why it matters:

- Probably low severity.
- Still worth fixing in a subsystem that is supposed to serialize and safely maintain global memory state.

## 10. `request_permissions` still bypasses guardian auto-review

There is an explicit TODO noting that guardian auto-review does not yet cover `request_permissions` / `with_additional_permissions`; those still route through the manual event path.

- TODO: `codex/codex-rs/core/src/codex.rs:3125-3134`

Why it matters:

- This creates an inconsistent approval story between normal exec approvals and permission-upgrade requests.
- It is more of a missing feature than a bug, but it is exactly the sort of security-surface inconsistency that tends to matter later.

## 11. `codex app` DMG installer path does not do integrity verification client-side

On macOS, `codex app` will `curl` a DMG, mount it via `hdiutil`, and copy `Codex.app` via `ditto`. The download URL is configurable via a flag.

- Default DMG URL: `codex/codex-rs/cli/src/app_cmd.rs:4-15`
- Download and install logic: `codex/codex-rs/cli/src/desktop_app/mac.rs:1-237`

Why it matters:

- This is user-initiated, but it is still a supply-chain surface: there is no hash/signature verification in the client itself (beyond whatever macOS enforces when launching).

## Summary

The public Codex repo is much cleaner than the Claude Code leak, but the rough edges cluster in predictable places:

- persistence/privacy defaults,
- cache/perf gaps on specialized paths,
- state-model mismatches in plugins,
- thin or incomplete safety/role polish around otherwise strong core designs.

Most of these are reportable without overstating them as catastrophic vulnerabilities.
