# Codex Catalog (Features, Commands, Env Vars, Events)

Index for the public `codex/` checkout in this workspace.

Notes:
- Line numbers are snapshot-specific.
- This focuses on product knobs and externally relevant identifiers (flags, commands, env vars, event types), not a complete symbol index.
- Full config key schema is in `codex/codex-rs/core/config.schema.json`.

## Feature flags

Source of truth is the `FEATURES` registry (`codex/codex-rs/features/src/lib.rs:521-855`).

| Key | Stage | Default | Ref |
|---|---|---:|---|
| `undo` | Stable | `false` | codex/codex-rs/features/src/lib.rs:523 |
| `shell_tool` | Stable | `true` | codex/codex-rs/features/src/lib.rs:529 |
| `unified_exec` | Stable | `!cfg!(windows)` | codex/codex-rs/features/src/lib.rs:535 |
| `shell_zsh_fork` | UnderDevelopment | `false` | codex/codex-rs/features/src/lib.rs:541 |
| `shell_snapshot` | Stable | `true` | codex/codex-rs/features/src/lib.rs:547 |
| `js_repl` | Experimental (JavaScript REPL) | `false` | codex/codex-rs/features/src/lib.rs:553 |
| `code_mode` | UnderDevelopment | `false` | codex/codex-rs/features/src/lib.rs:563 |
| `code_mode_only` | UnderDevelopment | `false` | codex/codex-rs/features/src/lib.rs:569 |
| `js_repl_tools_only` | UnderDevelopment | `false` | codex/codex-rs/features/src/lib.rs:575 |
| `web_search_request` | Deprecated | `false` | codex/codex-rs/features/src/lib.rs:581 |
| `web_search_cached` | Deprecated | `false` | codex/codex-rs/features/src/lib.rs:587 |
| `search_tool` | Removed | `false` | codex/codex-rs/features/src/lib.rs:593 |
| `codex_git_commit` | UnderDevelopment | `false` | codex/codex-rs/features/src/lib.rs:600 |
| `runtime_metrics` | UnderDevelopment | `false` | codex/codex-rs/features/src/lib.rs:606 |
| `sqlite` | Removed | `true` | codex/codex-rs/features/src/lib.rs:612 |
| `memories` | UnderDevelopment | `false` | codex/codex-rs/features/src/lib.rs:618 |
| `child_agents_md` | UnderDevelopment | `false` | codex/codex-rs/features/src/lib.rs:624 |
| `image_detail_original` | UnderDevelopment | `false` | codex/codex-rs/features/src/lib.rs:630 |
| `apply_patch_freeform` | UnderDevelopment | `false` | codex/codex-rs/features/src/lib.rs:636 |
| `exec_permission_approvals` | UnderDevelopment | `false` | codex/codex-rs/features/src/lib.rs:642 |
| `codex_hooks` | UnderDevelopment | `false` | codex/codex-rs/features/src/lib.rs:648 |
| `request_permissions_tool` | UnderDevelopment | `false` | codex/codex-rs/features/src/lib.rs:654 |
| `use_linux_sandbox_bwrap` | Removed | `false` | codex/codex-rs/features/src/lib.rs:660 |
| `use_legacy_landlock` | Stable | `false` | codex/codex-rs/features/src/lib.rs:666 |
| `request_rule` | Removed | `false` | codex/codex-rs/features/src/lib.rs:672 |
| `experimental_windows_sandbox` | Removed | `false` | codex/codex-rs/features/src/lib.rs:678 |
| `elevated_windows_sandbox` | Removed | `false` | codex/codex-rs/features/src/lib.rs:684 |
| `remote_models` | Removed | `false` | codex/codex-rs/features/src/lib.rs:690 |
| `enable_request_compression` | Stable | `true` | codex/codex-rs/features/src/lib.rs:696 |
| `multi_agent` | Stable | `true` | codex/codex-rs/features/src/lib.rs:702 |
| `multi_agent_v2` | UnderDevelopment | `false` | codex/codex-rs/features/src/lib.rs:708 |
| `enable_fanout` | UnderDevelopment | `false` | codex/codex-rs/features/src/lib.rs:714 |
| `apps` | Stable | `true` | codex/codex-rs/features/src/lib.rs:720 |
| `tool_search` | UnderDevelopment | `false` | codex/codex-rs/features/src/lib.rs:726 |
| `tool_suggest` | Stable | `true` | codex/codex-rs/features/src/lib.rs:732 |
| `plugins` | Stable | `true` | codex/codex-rs/features/src/lib.rs:738 |
| `image_generation` | UnderDevelopment | `false` | codex/codex-rs/features/src/lib.rs:744 |
| `skill_mcp_dependency_install` | Stable | `true` | codex/codex-rs/features/src/lib.rs:750 |
| `skill_env_var_dependency_prompt` | UnderDevelopment | `false` | codex/codex-rs/features/src/lib.rs:756 |
| `steer` | Removed | `true` | codex/codex-rs/features/src/lib.rs:762 |
| `default_mode_request_user_input` | UnderDevelopment | `false` | codex/codex-rs/features/src/lib.rs:768 |
| `guardian_approval` | Experimental (Guardian Approvals) | `false` | codex/codex-rs/features/src/lib.rs:774 |
| `collaboration_modes` | Removed | `true` | codex/codex-rs/features/src/lib.rs:784 |
| `tool_call_mcp_elicitation` | Stable | `true` | codex/codex-rs/features/src/lib.rs:790 |
| `personality` | Stable | `true` | codex/codex-rs/features/src/lib.rs:796 |
| `artifact` | UnderDevelopment | `false` | codex/codex-rs/features/src/lib.rs:802 |
| `fast_mode` | Stable | `true` | codex/codex-rs/features/src/lib.rs:808 |
| `realtime_conversation` | UnderDevelopment | `false` | codex/codex-rs/features/src/lib.rs:814 |
| `tui_app_server` | Removed | `true` | codex/codex-rs/features/src/lib.rs:820 |
| `prevent_idle_sleep` | Experimental/UnderDevelopment (Prevent sleep while running) | `false` | codex/codex-rs/features/src/lib.rs:826 |
| `responses_websockets` | Removed | `false` | codex/codex-rs/features/src/lib.rs:844 |
| `responses_websockets_v2` | Removed | `false` | codex/codex-rs/features/src/lib.rs:850 |

## Slash commands (TUI)

Source of truth is the `SlashCommand` enum and `description()` (`codex/codex-rs/tui/src/slash_command.rs:1-207`).

| Command | Description | Notes | Ref |
|---|---|---|---|
| `/model` | choose what model and reasoning effort to use |  | codex/codex-rs/tui/src/slash_command.rs:15 |
| `/fast` | toggle Fast mode to enable fastest inference at 2X plan usage |  | codex/codex-rs/tui/src/slash_command.rs:16 |
| `/approvals` | choose what Codex is allowed to do |  | codex/codex-rs/tui/src/slash_command.rs:17 |
| `/permissions` | choose what Codex is allowed to do |  | codex/codex-rs/tui/src/slash_command.rs:18 |
| `/setup-default-sandbox` | set up elevated agent sandbox |  | codex/codex-rs/tui/src/slash_command.rs:20 |
| `/sandbox-add-read-dir` | let sandbox read a directory: /sandbox-add-read-dir <absolute_path> | windows-only | codex/codex-rs/tui/src/slash_command.rs:22 |
| `/experimental` | toggle experimental features |  | codex/codex-rs/tui/src/slash_command.rs:23 |
| `/skills` | use skills to improve how Codex performs specific tasks |  | codex/codex-rs/tui/src/slash_command.rs:24 |
| `/review` | review my current changes and find issues |  | codex/codex-rs/tui/src/slash_command.rs:25 |
| `/rename` | rename the current thread |  | codex/codex-rs/tui/src/slash_command.rs:26 |
| `/new` | start a new chat during a conversation |  | codex/codex-rs/tui/src/slash_command.rs:27 |
| `/resume` | resume a saved chat |  | codex/codex-rs/tui/src/slash_command.rs:28 |
| `/fork` | fork the current chat |  | codex/codex-rs/tui/src/slash_command.rs:29 |
| `/init` | create an AGENTS.md file with instructions for Codex |  | codex/codex-rs/tui/src/slash_command.rs:30 |
| `/compact` | summarize conversation to prevent hitting the context limit |  | codex/codex-rs/tui/src/slash_command.rs:31 |
| `/plan` | switch to Plan mode |  | codex/codex-rs/tui/src/slash_command.rs:32 |
| `/collab` | change collaboration mode (experimental) |  | codex/codex-rs/tui/src/slash_command.rs:33 |
| `/agent` | switch the active agent thread |  | codex/codex-rs/tui/src/slash_command.rs:34 |
| `/diff` | show git diff (including untracked files) |  | codex/codex-rs/tui/src/slash_command.rs:36 |
| `/copy` | copy the latest Codex output to your clipboard | not-android | codex/codex-rs/tui/src/slash_command.rs:37 |
| `/mention` | mention a file |  | codex/codex-rs/tui/src/slash_command.rs:38 |
| `/status` | show current session configuration and token usage |  | codex/codex-rs/tui/src/slash_command.rs:39 |
| `/debug-config` | show config layers and requirement sources for debugging |  | codex/codex-rs/tui/src/slash_command.rs:40 |
| `/title` | configure which items appear in the terminal title |  | codex/codex-rs/tui/src/slash_command.rs:41 |
| `/statusline` | configure which items appear in the status line |  | codex/codex-rs/tui/src/slash_command.rs:42 |
| `/theme` | choose a syntax highlighting theme |  | codex/codex-rs/tui/src/slash_command.rs:43 |
| `/mcp` | list configured MCP tools |  | codex/codex-rs/tui/src/slash_command.rs:44 |
| `/apps` | manage apps |  | codex/codex-rs/tui/src/slash_command.rs:45 |
| `/plugins` | browse plugins |  | codex/codex-rs/tui/src/slash_command.rs:46 |
| `/logout` | log out of Codex |  | codex/codex-rs/tui/src/slash_command.rs:47 |
| `/quit` | exit Codex |  | codex/codex-rs/tui/src/slash_command.rs:48 |
| `/exit` | exit Codex |  | codex/codex-rs/tui/src/slash_command.rs:49 |
| `/feedback` | send logs to maintainers |  | codex/codex-rs/tui/src/slash_command.rs:50 |
| `/rollout` | print the rollout file path | debug-build-only | codex/codex-rs/tui/src/slash_command.rs:51 |
| `/ps` | list background terminals |  | codex/codex-rs/tui/src/slash_command.rs:52 |
| `/stop` | stop all background terminals | aliases: /clean | codex/codex-rs/tui/src/slash_command.rs:54 |
| `/clear` | clear the terminal and start a new chat |  | codex/codex-rs/tui/src/slash_command.rs:55 |
| `/personality` | choose a communication style for Codex |  | codex/codex-rs/tui/src/slash_command.rs:56 |
| `/realtime` | toggle realtime voice mode (experimental) |  | codex/codex-rs/tui/src/slash_command.rs:57 |
| `/settings` | configure realtime microphone/speaker |  | codex/codex-rs/tui/src/slash_command.rs:58 |
| `/test-approval` | test approval request | debug-build-only | codex/codex-rs/tui/src/slash_command.rs:59 |
| `/subagents` | switch the active agent thread |  | codex/codex-rs/tui/src/slash_command.rs:61 |
| `/debug-m-drop` | DO NOT USE |  | codex/codex-rs/tui/src/slash_command.rs:64 |
| `/debug-m-update` | DO NOT USE |  | codex/codex-rs/tui/src/slash_command.rs:66 |

## CLI subcommands

Source of truth is the `Subcommand` enum (`codex/codex-rs/cli/src/main.rs:89-155`).

| Subcommand | Description | Notes | Ref |
|---|---|---|---|
| `codex exec` | Run Codex non-interactively. | alias: e; visible_alias: e | codex/codex-rs/cli/src/main.rs:92 |
| `codex review` | Run a code review non-interactively. |  | codex/codex-rs/cli/src/main.rs:95 |
| `codex login` | Manage login. |  | codex/codex-rs/cli/src/main.rs:98 |
| `codex logout` | Remove stored authentication credentials. |  | codex/codex-rs/cli/src/main.rs:101 |
| `codex mcp` | Manage external MCP servers for Codex. |  | codex/codex-rs/cli/src/main.rs:104 |
| `codex mcp-server` | Start Codex as an MCP server (stdio). |  | codex/codex-rs/cli/src/main.rs:107 |
| `codex app-server` | [experimental] Run the app server or related tooling. |  | codex/codex-rs/cli/src/main.rs:110 |
| `codex app` | Launch the Codex desktop app (downloads the macOS installer if missing). | platform-gated | codex/codex-rs/cli/src/main.rs:114 |
| `codex completion` | Generate shell completion scripts. |  | codex/codex-rs/cli/src/main.rs:117 |
| `codex sandbox` | Run commands within a Codex-provided sandbox. |  | codex/codex-rs/cli/src/main.rs:120 |
| `codex debug` | Debugging tools. |  | codex/codex-rs/cli/src/main.rs:123 |
| `codex execpolicy` | Execpolicy tooling. | hidden | codex/codex-rs/cli/src/main.rs:127 |
| `codex apply` | Apply the latest diff produced by Codex agent as a `git apply` to your local working tree. | alias: a; visible_alias: a | codex/codex-rs/cli/src/main.rs:131 |
| `codex resume` | Resume a previous interactive session (picker by default; use --last to continue the most recent). |  | codex/codex-rs/cli/src/main.rs:134 |
| `codex fork` | Fork a previous interactive session (picker by default; use --last to fork the most recent). |  | codex/codex-rs/cli/src/main.rs:137 |
| `codex cloud` | [EXPERIMENTAL] Browse tasks from Codex Cloud and apply changes locally. | alias: cloud-tasks | codex/codex-rs/cli/src/main.rs:141 |
| `codex responses-api-proxy` | Internal: run the responses API proxy. | hidden | codex/codex-rs/cli/src/main.rs:145 |
| `codex stdio-to-uds` | Internal: relay stdio to a Unix domain socket. | hidden | codex/codex-rs/cli/src/main.rs:149 |
| `codex features` | Inspect feature flags. |  | codex/codex-rs/cli/src/main.rs:152 |

## Environment variables

Extracted from `CODEX_*` string literals under `codex/codex-rs`, plus a few extra high-signal UX/runtime vars.

| Env var | First ref |
|---|---|
| `ALL_PROXY` | codex/codex-rs/windows-sandbox-rs/src/setup_orchestrator.rs:474 |
| `CODEX_API_KEY` | codex/codex-rs/login/src/auth/manager.rs:393 |
| `CODEX_APPLY_GIT_CFG` | codex/codex-rs/git-utils/src/apply.rs:62 |
| `CODEX_APP_SERVER_LOGIN_ISSUER` | codex/codex-rs/app-server/tests/suite/v2/account.rs:49 |
| `CODEX_APP_SERVER_MANAGED_CONFIG_PATH` | codex/codex-rs/app-server/tests/suite/v2/config_rpc.rs:426 |
| `CODEX_APP_SERVER_URL` | codex/codex-rs/app-server-test-client/src/lib.rs:119 |
| `CODEX_ARC_MONITOR_ENDPOINT_OVERRIDE` | codex/codex-rs/core/src/arc_monitor.rs:17 |
| `CODEX_ARC_MONITOR_TOKEN` | codex/codex-rs/core/src/arc_monitor.rs:18 |
| `CODEX_BIN` | codex/codex-rs/app-server-test-client/src/lib.rs:112 |
| `CODEX_BWRAP_SOURCE_DIR` | codex/codex-rs/linux-sandbox/build.rs:89 |
| `CODEX_CA_CERTIFICATE` | codex/codex-rs/codex-client/tests/ca_env.rs:17 |
| `CODEX_CI` | codex/codex-rs/core/src/unified_exec/process_manager_tests.rs:19 |
| `CODEX_CLOUD_TASKS_BASE_URL` | codex/codex-rs/cloud-tasks/src/lib.rs:47 |
| `CODEX_CLOUD_TASKS_FORCE_INTERNAL` | codex/codex-rs/cloud-tasks/src/lib.rs:797 |
| `CODEX_CLOUD_TASKS_MODE` | codex/codex-rs/cloud-tasks/src/lib.rs:44 |
| `CODEX_CONNECTORS_TOKEN` | codex/codex-rs/core/src/mcp/mod.rs:34 |
| `CODEX_E2E_MODEL` | codex/codex-rs/app-server-test-client/src/lib.rs:262 |
| `CODEX_ESCALATE_SOCKET` | codex/codex-rs/shell-escalation/src/unix/escalate_protocol.rs:11 |
| `CODEX_EXEC_SERVER_URL` | codex/codex-rs/exec-server/src/environment.rs:15 |
| `CODEX_HOME` | codex/codex-rs/windows-sandbox-rs/src/setup_main_win.rs:351 |
| `CODEX_INTERNAL_ORIGINATOR_OVERRIDE` | codex/codex-rs/login/src/auth/default_client.rs:35 |
| `CODEX_JS_REPL_NODE_MODULE_DIRS` | codex/codex-rs/core/src/tools/js_repl/mod_tests.rs:2112 |
| `CODEX_JS_REPL_NODE_PATH` | codex/codex-rs/core/tests/suite/js_repl.rs:185 |
| `CODEX_JS_TMP_DIR` | codex/codex-rs/core/src/tools/js_repl/mod.rs:1020 |
| `CODEX_MANAGED_BY_BUN` | codex/codex-rs/tui/src/update_action.rs:34 |
| `CODEX_MANAGED_BY_NPM` | codex/codex-rs/tui/src/update_action.rs:33 |
| `CODEX_NETWORK_ALLOW_LOCAL_BINDING` | codex/codex-rs/windows-sandbox-rs/src/setup_orchestrator.rs:483 |
| `CODEX_NETWORK_POLICY_VIOLATION` | codex/codex-rs/network-proxy/src/runtime.rs:40 |
| `CODEX_OSS_BASE_URL` | codex/codex-rs/core/src/model_provider_info.rs:366 |
| `CODEX_OSS_PORT` | codex/codex-rs/core/src/model_provider_info.rs:359 |
| `CODEX_PROTOCOL_TEST_SYMLINKED_TMPDIR` | codex/codex-rs/protocol/src/permissions.rs:1260 |
| `CODEX_REALTIME_CONVERSATION_TEST_SUBPROCESS` | codex/codex-rs/core/tests/suite/realtime_conversation.rs:43 |
| `CODEX_REFRESH_TOKEN_URL_OVERRIDE` | codex/codex-rs/login/src/auth/manager.rs:85 |
| `CODEX_REMOTE_AUTH_TOKEN` | codex/codex-rs/cli/src/main.rs:1754 |
| `CODEX_REPO_ROOT_MARKER` | codex/codex-rs/utils/cargo-bin/src/lib.rs:172 |
| `CODEX_RMCP_CLIENT_AUTH_STATUS_TEST_TOKEN` | codex/codex-rs/rmcp-client/src/auth_status.rs:296 |
| `CODEX_RS_SSE_FIXTURE` | codex/codex-rs/tui/tests/suite/model_availability_nux.rs:94 |
| `CODEX_SANDBOX` | codex/codex-rs/login/src/auth/default_client.rs:242 |
| `CODEX_SANDBOX_ENV_VAR` | codex/codex-rs/shell-command/src/parse_command.rs:572 |
| `CODEX_SANDBOX_NETWORK_DISABLED` | codex/codex-rs/core/src/spawn.rs:19 |
| `CODEX_SNAPSHOT_POLICY_MARKER` | codex/codex-rs/core/tests/suite/shell_snapshot.rs:41 |
| `CODEX_SQLITE_HOME` | codex/codex-rs/state/src/lib.rs:55 |
| `CODEX_STARTING_DIFF` | codex/codex-rs/cloud-tasks-client/src/http.rs:333 |
| `CODEX_TEST_REMOTE_ENV` | codex/codex-rs/core/tests/common/lib.rs:323 |
| `CODEX_TEST_UNSET_OVERRIDE` | codex/codex-rs/core/src/tools/runtimes/mod_tests.rs:381 |
| `CODEX_THREAD_ID` | codex/codex-rs/core/src/exec_env.rs:8 |
| `CODEX_TUI_RECORD_SESSION` | codex/codex-rs/tui/src/session_log.rs:81 |
| `CODEX_TUI_ROUNDED` | codex/codex-rs/cloud-tasks/src/ui.rs:64 |
| `CODEX_TUI_SESSION_LOG_PATH` | codex/codex-rs/tui/src/session_log.rs:88 |
| `HTTPS_PROXY` | codex/codex-rs/windows-sandbox-rs/src/setup_orchestrator.rs:473 |
| `HTTP_PROXY` | codex/codex-rs/windows-sandbox-rs/src/setup_orchestrator.rs:472 |
| `OPENAI_BASE_URL` | codex/codex-rs/feedback/src/feedback_diagnostics.rs:3 |
| `ZELLIJ` | codex/codex-rs/terminal-detection/src/terminal_tests.rs:285 |
| `ZELLIJ_SESSION_NAME` | codex/codex-rs/terminal-detection/src/lib.rs:380 |
| `ZELLIJ_VERSION` | codex/codex-rs/terminal-detection/src/lib.rs:381 |
| `all_proxy` | codex/codex-rs/windows-sandbox-rs/src/setup_orchestrator.rs:479 |
| `http_proxy` | codex/codex-rs/windows-sandbox-rs/src/setup_orchestrator.rs:477 |
| `https_proxy` | codex/codex-rs/windows-sandbox-rs/src/setup_orchestrator.rs:478 |

## Analytics events

Codex's public analytics surface is a small set of event types, emitted as a list of `TrackEventRequest` items (`codex/codex-rs/analytics/src/analytics_client.rs:300-360`):

- `SkillInvocation`
- `AppMentioned`
- `AppUsed`
- `PluginUsed`
- `PluginInstalled`
- `PluginUninstalled`
- `PluginEnabled`
- `PluginDisabled`

The endpoint is `{base_url}/codex/analytics-events/events` (`codex/codex-rs/analytics/src/analytics_client.rs:600-609`).

## Feedback uploads and attachments

The `/feedback` upload path can include multiple attachments (`codex/codex-rs/feedback/src/lib.rs:245-340`, `codex/codex-rs/tui/src/bottom_pane/feedback_view.rs:500-571`):

- `codex-logs.log` (4 MiB ring buffer)
- rollout file (optional; filename depends on rollout path)
- `codex-connectivity-diagnostics.txt` (optional; includes raw proxy/base-url env vars when present)

## Prompt/model metadata (steerability)

If you want to understand "what the model is being told," don't miss these:

- Model registry + per-model base instructions: `codex/codex-rs/core/models.json:1-120`
- Full prompt text blobs for specific models: `codex/codex-rs/core/gpt_5_2_prompt.md:1-120`
- Built-in "orchestrator" agent template: `codex/codex-rs/core/templates/agents/orchestrator.md:1-114`
