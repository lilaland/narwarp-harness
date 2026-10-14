# Claude Code control surfaces, terms/billing, and Ollama + local models for the Narwarp harness (state as of 2026-10-13)

Method note: Primary sources were read directly where possible. These were the raw Claude Code docs (`code.claude.com/docs/en/*.md`), the raw `anthropics/claude-code` and `claude-agent-sdk-python` CHANGELOG.md files, npm and PyPI registry metadata (for release dates), the raw Ollama docs (`ollama/ollama/docs/*.mdx`), raw ollama.com library and tag HTML, the Ollama blog, and Hugging Face model cards. The local `claude --help` output (v2.1.296) was also checked in this environment. Practitioner and secondary sources are labeled as such. "(>6 mo)" marks information older than about 6 months that may be stale.

---

## 1. Claude Agent SDK (Python): version, query() vs ClaudeSDKClient, sessions, hooks, custom tools, subagents, permissions, streaming, settings sources, cost reporting

### Takeaway
The Python Agent SDK is at **v0.2.165 (2026-10-08)**. It bundles Claude Code CLI 2.1.294 and runs the real `claude` binary as a subprocess. `ClaudeSDKClient` is the right primitive for a long-lived, interruptible ticket worker. It provides interrupts, permission-mode and model switching, MCP toggling, background-task control and context-usage queries. The SDK also offers resume/fork/session-store, in-process MCP tools, programmatic subagents, a `can_use_tool` callback, typed rate-limit events, and per-model cost estimates. These cover nearly everything a harness needs. The main caveat is that only a subset of hook events can be registered as Python callbacks.

### Cited Findings
**Versions and packaging**
- PyPI `claude-agent-sdk`: latest is 0.2.165, uploaded 2026-10-08. 0.1.0 was uploaded 2025-09-28 and 0.2.82 on 2026-05-15. Requires Python >=3.10 — [PyPI](https://pypi.org/project/claude-agent-sdk/)
- 0.2.165 bundles CLI 2.1.294. 0.2.164 (Oct 6) bundles 2.1.292, and 0.2.160 (Sep 25) bundles 2.1.283. Most recent releases are only CLI version bumps — [SDK releases](https://github.com/anthropics/claude-agent-sdk-python/releases); [SDK CHANGELOG](https://github.com/anthropics/claude-agent-sdk-python/blob/main/CHANGELOG.md)
- The TypeScript SDK `@anthropic-ai/claude-agent-sdk` is at 0.3.296 (modified 2026-10-09). Its version tracks the CLI version number (2.1.296) — [npm](https://www.npmjs.com/package/@anthropic-ai/claude-agent-sdk)
- The SDK is described as "a library that runs the Claude Code binary, with Claude Code's capabilities, such as built-in tools, permissions, sessions, and hooks." Callers in other languages are told to run `claude -p --output-format json` as a subprocess — [Agent SDK overview](https://code.claude.com/docs/en/agent-sdk/overview)

**query() vs ClaudeSDKClient**
- The official comparison table:

  | | `query()` | `ClaudeSDKClient` |
  |---|---|---|
  | Session | New session by default | Reuses the same session |
  | Exchanges | Single | Multiple, same context |
  | Interrupts | No | Yes |
  | Continue chat | Manual (`continue_conversation`/`resume`) | Automatic |

  Both support streaming input, hooks and custom tools — [Python SDK reference](https://code.claude.com/docs/en/agent-sdk/python)
- `ClaudeSDKClient` methods: `connect`, `query(prompt, session_id)`, `receive_messages`, `receive_response`, `interrupt`, `set_permission_mode`, `set_model`, `rewind_files` (needs `enable_file_checkpointing`), `get_mcp_status`, `get_context_usage`, `reconnect_mcp_server`, `toggle_mcp_server`, `stop_task(task_id)`, `get_server_info`, `disconnect` — [Python SDK reference](https://code.claude.com/docs/en/agent-sdk/python)

**Sessions (resume/fork/store)**
- How the SDK finds a session:
  - *Continue* resumes the most recent session in the cwd.
  - *Resume* takes a specific session ID.
  - *Fork* creates a new session with a copy of the history and leaves the original unchanged.
  - `ClaudeSDKClient` tracks IDs automatically.

  — [SDK sessions](https://code.claude.com/docs/en/agent-sdk/sessions)
- `ClaudeAgentOptions` session fields:
  - `resume`
  - `session_id` (a custom UUID; it can't be combined with resume/continue unless `fork_session` is set)
  - `fork_session`
  - `resume_session_at` (branch from a message UUID; SDK ≥0.2.137)
  - `resume_drops_turn`
  - `continue_conversation`

  — [Python SDK reference](https://code.claude.com/docs/en/agent-sdk/python)
- Top-level session functions are `list_sessions()`, `get_session_messages()`, `get_session_info()`, `rename_session()`, `tag_session()` and `fork_session()` — [Python SDK reference](https://code.claude.com/docs/en/agent-sdk/python)
- The `SessionStore` protocol has `append`, `load`, `list_sessions`, `delete` and `list_subkeys`. It mirrors transcripts to external storage so another host can resume them. Reference adapters exist for S3, Redis and Postgres (since 0.1.64). The `session_store_flush` option is `"batched"` or `"eager"` — [SDK CHANGELOG](https://github.com/anthropics/claude-agent-sdk-python/blob/main/CHANGELOG.md); [Python SDK reference](https://code.claude.com/docs/en/agent-sdk/python)

**Hooks in the Python SDK**
- **Python supports** these hook callbacks: `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `UserPromptSubmit`, `Stop`, `SubagentStart`, `SubagentStop`, `PreCompact`, `PermissionRequest`, `Notification`.
- **TypeScript-only** callbacks: `SessionStart`, `SessionEnd`, `Setup`, `PostToolBatch`, `MessageDisplay`, `StopFailure`, `PostCompact`, `PermissionDenied`, `TeammateIdle`, `TaskCreated`, `TaskCompleted`, `Elicitation`/`ElicitationResult`, `ConfigChange`, `InstructionsLoaded`, `WorktreeCreate`/`WorktreeRemove`, `CwdChanged`, `FileChanged`, `DirectoryAdded`, `UserPromptExpansion`, `Pre`/`PostModelSwitch`.

  — [SDK hooks](https://code.claude.com/docs/en/agent-sdk/hooks)
- Hooks are passed as `hooks={"PreToolUse": [HookMatcher(matcher="Bash", hooks=[cb])]}` — [SDK hooks](https://code.claude.com/docs/en/agent-sdk/hooks)

**Custom in-process MCP tools and subagents**
- Custom tools are defined with `@tool`, wrapped in `create_sdk_mcp_server()`, and passed via `mcp_servers`. The docs call this the SDK's "in-process MCP server" — [SDK custom tools](https://code.claude.com/docs/en/agent-sdk/custom-tools)
- Subagents are defined programmatically through `agents: dict[str, AgentDefinition]`. `AgentDefinition` fields are `description`, `prompt`, `tools`, `model`, `skills`, `memory` (user/project/local), `background` and `effort` — [Python SDK reference](https://code.claude.com/docs/en/agent-sdk/python)
- 0.2.160 fix: stdin was closed early when a background subagent finished just before the result, which broke follow-up turns. The SDK now waits for the CLI's `session_state_changed` idle. The wait ceiling is `CLAUDE_CODE_PRINT_BG_WAIT_CEILING_MS`, default 10 min — [SDK CHANGELOG](https://github.com/anthropics/claude-agent-sdk-python/blob/main/CHANGELOG.md)
- Typed `TaskStarted`, `TaskProgress` and `TaskNotification` messages were added in 0.1.46. An `origin` field on `UserMessage`/`ResultMessage` (0.2.137) distinguishes human-submitted turns from background-task notifications — [SDK CHANGELOG](https://github.com/anthropics/claude-agent-sdk-python/blob/main/CHANGELOG.md)

**Permissions**
- `PermissionMode` values: `default`, `acceptEdits`, `plan`, `dontAsk` (deny anything not pre-approved), `bypassPermissions`, and `auto` (a model classifier reviews actions) — [Python SDK reference](https://code.claude.com/docs/en/agent-sdk/python)
- `can_use_tool` fires only when the permission flow falls through to an "ask". It does not fire on allow or deny decisions (doc clarification in 0.1.74). It works with `query()` and string prompts since 0.2.140. `ToolPermissionContext` gains `decision_reason`, `blocked_path`, `title` and `description` — [SDK CHANGELOG](https://github.com/anthropics/claude-agent-sdk-python/blob/main/CHANGELOG.md)
- `verbatim_prompts=True` (0.2.158, requires CLI ≥2.1.248) delivers user messages "exactly as written, with no `@path` file expansion and no slash-command dispatch". It is meant for untrusted text — [SDK releases](https://github.com/anthropics/claude-agent-sdk-python/releases)
- Other relevant options:
  - `allowed_tools` and `disallowed_tools` (scoped rules like `Bash(rm *)` deny in every mode, including bypass)
  - `sandbox`
  - `user` (run the CLI as another OS user)

  — [Python SDK reference](https://code.claude.com/docs/en/agent-sdk/python)

**Streaming, settings sources, other options**
- `include_partial_messages` yields `StreamEvent` objects.
- `include_hook_events` yields `HookEventMessage` objects.
- `forward_subagent_text` forwards subagent text and thinking.
- `output_format` accepts `{"type":"json_schema","schema":…}` for structured output.
- `effort` takes `low`/`medium`/`high`/`xhigh`/`max`. `thinking` and `task_budget` are also available.

  — [Python SDK reference](https://code.claude.com/docs/en/agent-sdk/python)
- **Settings sources (current behavior):** when `setting_sources` is omitted, `query()` "loads the same filesystem settings as the Claude Code CLI: user, project, and local". `[]` disables all filesystem settings. Precedence is local > project > user, and programmatic options override them, but managed policy overrides programmatic options — [Python SDK reference](https://code.claude.com/docs/en/agent-sdk/python)
- This contradicts the 0.1.0 release note (2025-09-28, >6 mo): "No filesystem settings by default". Older blog posts that repeat the 0.1.0 behavior are stale — [SDK CHANGELOG](https://github.com/anthropics/claude-agent-sdk-python/blob/main/CHANGELOG.md)

**Cost and usage reporting**
- `ResultMessage` fields: `total_cost_usd`, `usage`, `model_usage` (per model, including subagents), `num_turns`, `duration_ms`, `session_id`, `permission_denials`, `structured_output`, `stop_reason`, `terminal_reason` and `origin`. `subtype` is one of `success`, `error_during_execution`, `error_max_turns`, `error_max_budget_usd` or `error_max_structured_output_retries` — [Python SDK reference](https://code.claude.com/docs/en/agent-sdk/python)
- "The `total_cost_usd` and `costUSD` fields are client-side estimates, not authoritative billing data". They are computed from a bundled price table. Use `model_usage` for whole-tree accounting, because `usage` undercounts once subagents nest — [SDK cost tracking](https://code.claude.com/docs/en/agent-sdk/cost-tracking)
- `RateLimitEvent`/`RateLimitInfo` fields:
  - `status`: `allowed`, `allowed_warning` or `rejected`
  - `rate_limit_type`: `five_hour`, `seven_day`, `seven_day_opus`, `seven_day_sonnet` or `overage`
  - `utilization` (0–1)
  - `resets_at`
  - `overage_status`

  — [Python SDK reference](https://code.claude.com/docs/en/agent-sdk/python)
- `max_budget_usd` stops a query when the client-side estimate is reached (added in 0.1.6). `ClaudeSDKClient.get_context_usage()` returns the same breakdown as `/context` (0.1.52) — [SDK CHANGELOG](https://github.com/anthropics/claude-agent-sdk-python/blob/main/CHANGELOG.md)

### Inferences
- For Narwarp, a "ticket worker" maps naturally to one `ClaudeSDKClient` per active ticket. The harness would:
  - persist `session_id` on the ticket and `resume` it later;
  - `fork_session` for "try another approach" or handoffs;
  - use `interrupt()` and `set_permission_mode()` from the kanban UI;
  - route `can_use_tool` asks to the message board or a notification;
  - expose board and ticket operations as in-process MCP tools (`post_update`, `handoff`, `link_linear_issue`).
- Because Python cannot register `SessionStart` or `SessionEnd` callbacks, a Python harness should use `UserPromptSubmit` and `PreToolUse` callbacks, or settings-file hooks, for session start and end context and status. It can also just observe `system/init` and `ResultMessage` in the message stream.
- Use `verbatim_prompts=True` whenever ticket text comes from Discord, email or Linear. Otherwise injected `@path` or `/command` strings could trigger file reads or skills.
- Set `setting_sources` explicitly. Otherwise the user's personal hooks, plugins and CLAUDE.md leak into every harness worker, which may be desirable for coding tickets but not for life-management agents.

### Gaps
- No primary documentation was found that states exactly when the SDK's default `setting_sources` behavior flipped back to "load all".
- I did not test the SDK end-to-end in this environment; the behavior described comes from the docs and changelog.

---

## 2. Headless CLI (`claude -p`, stream-json in/out, resume/continue/session-id) and hooks for status reporting and context injection

### Takeaway
`claude -p` with `--output-format stream-json` and `--input-format stream-json` is the language-agnostic wire protocol underneath the SDKs. It has stable session flags (`--resume <id|path.jsonl>`, `--continue`, `--session-id <uuid>`, `--fork-session`) and structured output (`--json-schema`). Claude Code's hook system now has 33 events and five handler types, including **HTTP hooks** that POST JSON to a URL. That makes it the cleanest way for a harness daemon to receive status events and inject context. There is one important risk: docs say `--bare` (which never reads OAuth credentials) "will become the default for `-p` in a future release."

### Cited Findings
**Current CLI version and flags**
- Claude Code CLI 2.1.296 was published to npm on 2026-10-09. 2.1.0 was published 2026-01-07 and 2.0.0 on 2025-09-29. There were 542 published versions in total since 0.2.6 on 2025-02-24, which is a very fast release cadence — [npm @anthropic-ai/claude-code](https://www.npmjs.com/package/@anthropic-ai/claude-code?activeTab=versions)
- Key flags in v2.1.296, verified in the local `claude --help` output:
  - `-p/--print`
  - `--output-format text|json|stream-json` and `--input-format text|stream-json`
  - `--include-partial-messages`, `--include-hook-events`, `--replay-user-messages`, `--forward-subagent-text`
  - `--json-schema`, `--max-budget-usd`
  - `--permission-mode` (acceptEdits, auto, bypassPermissions, manual, dontAsk, plan) and `--permission-prompts host|none`
  - `--allowedTools`, `--disallowedTools`, `--tools`
  - `--mcp-config`, `--strict-mcp-config`, `--setting-sources`, `--settings`
  - `--agents <json>`, `--plugin-dir`, `--plugin-url`
  - `-r/--resume [id]`, `-c/--continue`, `--session-id <uuid>`, `--fork-session`, `--no-session-persistence`
  - `--fallback-model`, `--effort`, `--bare`, `--restricted`, `--safe-mode`
  - `--bg`, `-n/--name`, `-w/--worktree`, `--tmux`, `--remote-control`, `--prompt-suggestions`

  — [CLI reference](https://code.claude.com/docs/en/cli-reference)
- `--resume` also accepts the absolute path to a session's `.jsonl` transcript. Since v2.1.257, `--continue` "opens a background session that has finished, but not one that is still running" — [Headless docs](https://code.claude.com/docs/en/headless)

**stream-json event model**
- `system/init` is the first event. It reports the model, tools, MCP servers and plugins. It is preceded by `hook_started`/`hook_progress`/`hook_response` events when SessionStart or Setup hooks run — [Headless docs](https://code.claude.com/docs/en/headless)
- Other events:
  - `system/api_retry` is emitted before retries.
  - `system/plugin_install` is emitted when `CLAUDE_CODE_SYNC_PLUGIN_INSTALL` is set.
  - Denials appear as `permission_denied` system messages and in the result's `permission_denials`.
  - Token streaming needs `--output-format stream-json --verbose --include-partial-messages`.

  — [Headless docs](https://code.claude.com/docs/en/headless)
- `--output-format json` includes `total_cost_usd` and a per-model cost breakdown. `--json-schema` puts the validated object in `structured_output` — [Headless docs](https://code.claude.com/docs/en/headless)
- The CLI emits a `rate_limit_event` message type. SDK 0.1.40 fixed crashes when it encountered this then-unknown type — [SDK CHANGELOG](https://github.com/anthropics/claude-agent-sdk-python/blob/main/CHANGELOG.md)
- "Background tasks at exit": `-p` waits for background subagents, workflows and pending `/loop` wakeups before emitting the result. It prints `Waiting for background work to finish` to stderr — [Headless docs](https://code.claude.com/docs/en/headless)
- History:
  - stream-json output existed in 0.2.66 (2025-04-09).
  - 2.1.251 (2026-08-28) fixed client-injected assistant tool calls in `--input-format stream-json`.

  — [Claude Code CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md); release dates from [npm](https://www.npmjs.com/package/@anthropic-ai/claude-code?activeTab=versions)

**Bare mode and trust**
- `--bare` skips hooks, skills, plugins, MCP servers, auto memory and CLAUDE.md discovery. "In bare mode, Claude Code never reads OAuth credentials or the system keychain". It needs `ANTHROPIC_API_KEY` or `apiKeyHelper`. The docs say: "`--bare` is the recommended mode for scripted and SDK calls, and will become the default for `-p` in a future release." — [Headless docs](https://code.claude.com/docs/en/headless)
- Without `--bare`, a `-p` session runs project `.claude/settings.json` hooks and connects `.mcp.json` servers "even in a folder you've never trusted". It shows no trust dialog — [Headless docs](https://code.claude.com/docs/en/headless)

**Hooks reference (33 events, 5 handler types)**
- Events:
  - Session lifecycle: `SessionStart`, `Setup`, `SessionEnd`
  - Prompts: `UserPromptSubmit`, `UserPromptExpansion`
  - Tools and permissions: `PreToolUse`, `PermissionRequest`, `PermissionDenied`, `PostToolUse`, `PostToolUseFailure`, `PostToolBatch`
  - Output and stops: `Notification`, `MessageDisplay`, `Stop`, `StopFailure`
  - Subagents, tasks and teams: `SubagentStart`, `SubagentStop`, `TaskCreated`, `TaskCompleted`, `TeammateIdle`
  - Config and filesystem: `InstructionsLoaded`, `ConfigChange`, `CwdChanged`, `DirectoryAdded`, `FileChanged`, `WorktreeCreate`, `WorktreeRemove`
  - Compaction and models: `PreCompact`, `PostCompact`, `PreModelSwitch`, `PostModelSwitch`
  - MCP: `Elicitation`, `ElicitationResult`

  — [Hooks reference](https://code.claude.com/docs/en/hooks)
- Handler types are `command`, `http` (POSTs the JSON input to a URL and reads the decision from the response), `mcp_tool`, `prompt` and `agent` (experimental) — [Hooks reference](https://code.claude.com/docs/en/hooks)
- HTTP hooks were added in v2.1.63 (2026-02-28). `onFailure: "block"` for command and HTTP hooks was added in v2.1.295 (2026-10-08) — [Claude Code CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)
- Common input fields: `session_id`, `transcript_path`, `cwd`, `permission_mode`, `hook_event_name`, `prompt_id`, `scratchpad_dir`, `effort`, `agent_id` and `agent_type` — [Hooks reference](https://code.claude.com/docs/en/hooks)
- Output fields:
  - `continue`/`stopReason`, `systemMessage`
  - `terminalSequence` (an escape sequence Claude Code emits, for notifications, titles or the bell)
  - `decision:"block"`
  - `hookSpecificOutput.additionalContext` (injected as a system reminder, capped at 10,000 chars)
  - `permissionDecision` (allow/deny/ask/defer), `updatedInput`, `updatedToolOutput`
  - PermissionRequest `decision.behavior`

  — [Hooks reference](https://code.claude.com/docs/en/hooks)
- Which events can do what:
  - `SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `SubagentStart`, `Stop` and `PostModelSwitch` can inject `additionalContext`.
  - `Stop` and `SubagentStop` can block, which keeps Claude working.
  - `async: true` runs a hook in the background. `asyncRewake: true` wakes Claude on exit code 2.

  — [Hooks reference](https://code.claude.com/docs/en/hooks)
- Hook config locations: `~/.claude/settings.json`, `.claude/settings.json`, `.claude/settings.local.json`, managed policy, plugin `hooks/hooks.json`, skill frontmatter and subagent frontmatter. Hooks from all sources merge — [Hooks reference](https://code.claude.com/docs/en/hooks)

**Status line and terminal-level monitoring**
- The status-line JSON includes `cost.total_cost_usd`, `context_window.used_percentage`, and `rate_limits.five_hour` / `rate_limits.seven_day` `.used_percentage` and `.resets_at` — [Status line docs](https://code.claude.com/docs/en/statusline)
- `rate_limits` was added to statusline input in v2.1.80 (2026-03-19) — [Claude Code CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)
- **OSC 7501 Program Status Protocol**: "terminals that implement it can show whether Claude Code is working, waiting on you, or done". It was added in v2.1.295 (2026-10-08) — [Claude Code CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)
- Background sessions and sessions inside tmux don't report OSC 7501 to the outer terminal — [Terminal config](https://code.claude.com/docs/en/terminal-config)
- OSC 7501 is a terminal-native protocol proposed by Mitchell Hashimoto around 2026-10-08. Per secondary coverage, its states include idle, working, blocked, done and error, and it has hierarchical task IDs. Claimed adopters include libghostty, Claude Code and Codex — [zeli.app write-up (secondary)](https://zeli.app/story/49984159)
- Claude Code also emits an OSC 9;4 progress bar (since 2.0.56, 2025-12-01), OSC 8 hyperlinks, and desktop notifications in Ghostty, Kitty and iTerm2 — [Claude Code CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md); [Terminal config](https://code.claude.com/docs/en/terminal-config)

### Inferences
- Narwarp is itself a terminal, which gives it a unique advantage. It can **implement OSC 7501** (plus OSC 9;4 and notifications) to get zero-config working/blocked/done status for every interactive Claude Code pane, and for other agents that adopt it, without screen-scraping. HTTP hooks can complement this with structured events (tool calls, permission requests, stops) POSTed to a local Narwarp daemon endpoint.
- Context injection pattern:
  - A `SessionStart` (settings) hook or the `UserPromptSubmit` `additionalContext` adds the ticket brief, project rules and recent board messages, within the 10k-char cap.
  - A `Stop` hook decides whether the ticket's acceptance criteria are met before letting the session stop.
  - `PermissionRequest` or `Notification` hooks forward "needs you" states to the board.
- Treat the `--bare`-becomes-default warning as a real risk for any harness that relies on subscription auth in `-p`. Pin CLI versions in Narwarp, and keep an API-key path ready.

### Gaps
- The exact OSC 7501 wire format (field names) was not verified against the spec text; only secondary coverage was found.
- No date has been announced for `--bare` becoming the `-p` default, and the docs don't say whether an opt-out will preserve OAuth.

---

## 3. Other Claude Code surfaces: Channels, Remote Control, background sessions/agent view, cross-session messaging, Agent Teams, scheduled tasks/routines, plugins/skills/mods/marketplaces, MCP client config, ACP adapter

### Takeaway
Claude Code now has first-party answers to most harness needs:
- **Channels** push external events into a running session. They are in research preview.
- **Background sessions + `claude agents --json`** provide a supervised local fleet with a supported JSON status interface.
- **Cross-session messaging** (`ListAgents`/`SendMessage`) lets sessions message each other.
- **Remote Control** lets you steer sessions from a phone. It requires a subscription.
- **Routines** run cloud cron, API-fired and GitHub-triggered jobs.
- **Plugins/skills/mods** handle packaging.
- An **ACP adapter** wraps the TypeScript Agent SDK.

Agent Teams remains experimental and is not available in `-p`/SDK sessions.

### Cited Findings
**Channels (verified: pushing external messages into a running session)**
- "A channel is an MCP server that pushes events into your running Claude Code session". It can be two-way, like a chat bridge. "Events only arrive while the session is open". Telegram, Discord and iMessage plugins ship in the research preview, plus a `fakechat` demo, all as Bun scripts. You start it with `claude --channels plugin:telegram@claude-plugins-official` — [Channels](https://code.claude.com/docs/en/channels)
- Channels require claude.ai or Console API-key auth and are not available on Bedrock, Vertex or Foundry. Pro and Max users without an org skip the enterprise checks. Team and Enterprise need `channelsEnabled` — [Channels](https://code.claude.com/docs/en/channels)
- Timeline:
  - `--channels` was added in v2.1.80 (2026-03-19).
  - Permission relay to phone was added in 2.1.81.
  - Console API-key auth was added in 2.1.128 (2026-05-04).

  (Introduced >6 mo ago, and still labeled research preview today.) — [Claude Code CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)
- During the preview, `--channels` accepts only allowlisted plugins. Custom channels require `--dangerously-load-development-channels`. Neither flag appears in `--help`. "Being in `.mcp.json` isn't enough… a server also has to be named in `--channels`". Each plugin keeps a sender allowlist with pairing. In `-p` mode, tools that need terminal input are disabled — [Channels](https://code.claude.com/docs/en/channels)
- Building a channel takes three steps: declare `capabilities: { experimental: { 'claude/channel': {} } }`, emit `notifications/claude/channel` events, and connect over stdio. A reply tool makes it two-way. The docs include an example webhook receiver on port 8788 — [Channels reference](https://code.claude.com/docs/en/channels-reference)
- "On the v2 MCP client runtime… Claude Code doesn't register a channel server that negotiates protocol revision 2026-07-28" — [Channels](https://code.claude.com/docs/en/channels)

**Remote Control**
- Remote Control drives a local session from claude.ai/code or the Claude mobile app. It is available on Pro, Max, Team and Enterprise. "API keys are not supported" — [Remote Control](https://code.claude.com/docs/en/remote-control)
- Modes:
  - `claude remote-control` is server mode. It takes `--spawn same-dir|worktree|session` and `--capacity <N>` (default 32 concurrent sessions).
  - `claude --remote-control` / `--rc` starts an interactive session with Remote Control on.
  - `/remote-control` turns it on inside a session.
  - A config toggle can enable it for all sessions.

  — [Remote Control](https://code.claude.com/docs/en/remote-control)
- Transport: it makes "outbound HTTPS requests only and never opens inbound ports". It registers with the Anthropic API and polls for work. Server-mode sessions can be brought back for about 4 hours after stopping (`--continue`/`--session-id`, v2.1.200+) — [Remote Control](https://code.claude.com/docs/en/remote-control)
- Remote Control appeared around February 2026 (fixes in 2.1.53, 2026-02-24; expansion in 2.1.58; >6 mo) — [Claude Code CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)

**Background sessions / agent view**
- Agent view (`claude agents`) is labeled research preview. It was added in 2.1.139 (2026-05-11). It is "one screen for all your background sessions". A **supervisor process** runs sessions, so they survive closing the terminal and machine sleep — [Agent view](https://code.claude.com/docs/en/agent-view); [Claude Code CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)
- Commands: `claude --bg`, `claude attach <id>`, `claude logs`, `claude stop`, `claude rm`, `claude respawn --all` and `claude daemon stop [--keep-workers]` — [Agent view](https://code.claude.com/docs/en/agent-view); [CLI reference](https://code.claude.com/docs/en/cli-reference)
- `claude agents --json [--all] [--cwd]` "is the supported way to read session state from outside Claude Code". Its fields are `id`, `state` (working/blocked/done/failed/stopped), `pid`, `status` (busy/waiting/idle), `waitingFor` (permission prompt, input needed, …), `sessionId` and `name`. The JSON listing was added in 2.1.145 (2026-05-19). `~/.claude/jobs/<id>/state.json` "is not a stable interface" — [Agent view](https://code.claude.com/docs/en/agent-view); [Claude Code CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)
- "Background sessions consume your subscription usage the same as interactive sessions, so running ten agents in parallel uses quota roughly ten times as fast" — [Agent view](https://code.claude.com/docs/en/agent-view)

**Cross-session messaging**
- Sessions discover each other with `ListAgents` and send with `SendMessage`. Messaging reaches sessions on this machine, other machines (via Remote Control) and the cloud. It requires v2.1.224+ (2026-08-07) on macOS and Linux and is on by default — [Cross-session messaging](https://code.claude.com/docs/en/cross-session-messaging); [Claude Code CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)
- Relayed messages "no longer carry user authority — receivers refuse relayed permission requests" (2.1.166) — [Claude Code CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)

**Agent Teams, parallel-work options**
- "Agent teams are experimental and disabled by default". They are enabled with `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`. "In non-interactive mode with the `-p` flag, including Agent SDK sessions, Claude doesn't spawn teammates". Display is in-process or split panes (tmux/iTerm2) — [Agent teams](https://code.claude.com/docs/en/agent-teams)
- Agent Teams was first released in 2.1.32 (2026-02-05; >6 mo, still experimental) — [Claude Code CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)
- Official parallelism options are subagents, agent view, agent teams, dynamic workflows and projects. `/batch` splits a change into 5–30 worktree-isolated subagents — [Run agents in parallel](https://code.claude.com/docs/en/agents)

**Scheduling: /loop, Desktop tasks, Routines**
- Comparison of the three options:

  | | Cloud routines | Desktop tasks | `/loop` |
  |---|---|---|---|
  | Persistence | Persistent | Persistent | Restored on `--resume` with exceptions |
  | Minimum interval | 1 hour | 1 minute | 1 minute |

  — [Scheduled tasks](https://code.claude.com/docs/en/scheduled-tasks)
- `/loop` and `CronCreate` are session-scoped. A session can hold at most 50 tasks. Recurring tasks expire after 7 days, and jitter shifts runs by up to 30 min. `/loop` was added in 2.1.71 (2026-03-06) — [Scheduled tasks](https://code.claude.com/docs/en/scheduled-tasks); [Claude Code CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)
- Routines are in research preview.
  - Triggers are Scheduled, API and GitHub. They are available on Pro, Max, Team and Enterprise, and run on Anthropic cloud or self-hosted environments.
  - There is no permission picker; they run autonomously.
  - Runs draw down subscription usage.
  - Hourly caps: 100 scheduled runs per hour per account, and 100 API fires per hour per account (30 per routine).
  - The API trigger is `POST https://api.anthropic.com/v1/claude_code/routines/<trig_id>/fire` with a bearer token and `anthropic-beta: experimental-cc-routine-2026-04-01`. It is "available to claude.ai users only".

  — [Routines](https://code.claude.com/docs/en/routines)

**Plugins, skills, marketplaces, mods; MCP client config**
- Plugins package skills, agents, hooks and MCP servers. They install from marketplaces such as `claude-plugins-official`. `--plugin-dir` and `--plugin-url` load plugins per session, and the SDK takes a `plugins` option — [Plugins overview](https://code.claude.com/docs/en/plugins/overview); [CLI reference](https://code.claude.com/docs/en/cli-reference); [Python SDK reference](https://code.claude.com/docs/en/agent-sdk/python)
- **Mods** are a newer plugin type made of JS/TS handlers that run *inside* Claude Code. They can draw panes or bands, intercept or answer tool calls, "send one request to a different model", and add instant `/commands`. They work in the CLI and Desktop Code tab — [Mods overview](https://code.claude.com/docs/en/plugins/mods/overview)
- MCP configuration:
  - `claude mcp add --transport http <name> <url>`; HTTP is the recommended transport for remote servers.
  - JSON config lives in `.mcp.json`, `~/.claude.json` or `claude mcp add-json`.
  - `--mcp-config` and `--strict-mcp-config` control per-run servers.
  - claude.ai connectors are also available.

  — [MCP docs](https://code.claude.com/docs/en/mcp); [CLI reference](https://code.claude.com/docs/en/cli-reference)

**ACP adapter**
- `@agentclientprotocol/claude-agent-acp` is "An ACP-compatible coding agent powered by the Claude Agent SDK (TypeScript)". The latest stable is 0.89.1 and the newest preview is 0.89.2-preview.18 (2026-10-13). It depends on `@anthropic-ai/claude-agent-sdk` 0.3.293 and `@agentclientprotocol/sdk` 1.8.0. 193 versions have been published since 0.24.0 (2026-03-26) — [npm](https://www.npmjs.com/package/@agentclientprotocol/claude-agent-acp); [GitHub](https://github.com/agentclientprotocol/claude-agent-acp)
- The predecessor `@zed-industries/claude-code-acp` was last modified 2026-03-26 at 0.16.2 (>6 mo, superseded) — [npm](https://www.npmjs.com/package/@zed-industries/claude-code-acp)

### Inferences
- For Narwarp's integrations (Discord, Linear, GitHub, email → "post into this ticket's live session"), a **custom Narwarp channel MCP server** is the native ingress. It would be one server, keyed by ticket ID, that pushes `notifications/claude/channel` into the matching session. During the preview it needs `--dangerously-load-development-channels`, and the contract may change, so wrap it behind an adapter.
- `claude --bg` plus polling `claude agents --json --all` gives a supervisor-managed fleet "for free". This is an alternative to Narwarp owning processes; process-architecture researchers should weigh it. Its `waitingFor` field maps directly to a kanban "Blocked on you" swimlane.
- The ACP adapter matters if Narwarp wants to speak ACP (Zed-style editor/agent protocol) to *many* agents (Claude, Codex, Gemini CLI) uniformly. Otherwise, the Python SDK is more direct.
- Cross-session `SendMessage` is useful between Claude sessions, but local Ollama agents aren't part of it. A Narwarp-owned board (as an MCP server used by all agents) remains necessary for heterogeneous collaboration.

### Gaps
- Channel semantics for a session in `-p` or SDK mode, such as whether an SDK-hosted session can receive channel events, were not explicitly documented beyond "tools that need terminal input are disabled" in `-p`.
- No public ACP adapter README was read. Its auth behavior (whether it uses the user's own Claude Code login) is not verified.

---

## 4. Terms & billing: subscription OAuth vs API key for a personal harness; 2025–2026 changes; usage limits and how harnesses track them

### Takeaway
**Clearly allowed:** a user signing in to the **unmodified official `claude` binary** with their own subscription, including when another product hosts or runs it. Subscription limits "assume ordinary, individual usage of Claude Code and the Agent SDK."

**Clearly not allowed:**
- third-party apps offering claude.ai login;
- collecting, storing or intermediating OAuth tokens;
- routing other users through your plan;
- spoofing the Claude Code client.

Enforcement escalated in 2026. In January, third-party OAuth was blocked server-side. In April, subscription limits stopped covering third-party harnesses like OpenClaw. In May–June, a planned separate "Agent SDK credit" for `claude -p` and SDK usage was announced and then reportedly paused.

A personal Narwarp that shells out to `claude` or uses the official SDK under the user's own login is within the documented "ordinary individual use" zone. Anthropic's SDK docs still steer *developers building products* to API keys.

### Cited Findings
**Current official policy text**
- "OAuth authentication is intended exclusively for purchasers of Claude Free, Pro, Max, Team, and Enterprise subscription plans and is designed to support ordinary use of Claude Code and other native Anthropic applications." — [Claude Code legal & compliance](https://code.claude.com/docs/en/legal-and-compliance)
- "Developers building products or services that interact with Claude's capabilities, including those using the Agent SDK, should use API key authentication… Anthropic does not permit third-party developers to offer Claude.ai login into their own applications, or to route requests through Free, Pro, or Max plan credentials on behalf of their users. Moreover, developers may not collect, store, or intermediate Claude.ai credentials or session tokens." — [Claude Code legal & compliance](https://code.claude.com/docs/en/legal-and-compliance)
- "Nor does it prevent an end user from signing in to the unmodified Claude Code binary with their own Claude subscription". Also: "Advertised usage limits for Pro and Max plans assume ordinary, individual usage of Claude Code and the Agent SDK." — [Claude Code legal & compliance](https://code.claude.com/docs/en/legal-and-compliance)
- Products embedding Claude Code must agree to the Commercial Terms. In addition, "The Claude Code binary must not be modified", and they may not "pay for, resell, or intermediate Claude usage on their end users' behalf". Names and logos can't be used in a product name — [Claude Code legal & compliance](https://code.claude.com/docs/en/legal-and-compliance)
- "Unless previously approved, Anthropic does not allow third party developers to offer claude.ai login or rate limits for their products, including agents built on the Claude Agent SDK." Use of the Agent SDK "is governed by Anthropic's Commercial Terms of Service". Branding rules forbid calling your product "Claude Code" or mimicking it — [Agent SDK overview](https://code.claude.com/docs/en/agent-sdk/overview)
- The Consumer Terms (effective 2025-10-08, >6 mo) prohibit accessing the Services "through automated or non-human means, whether through a bot, script, or otherwise" except "via an Anthropic API Key or where we otherwise explicitly permit it". They also prohibit sharing credentials or making your account available to others. They don't mention OAuth in third-party tools specifically — [Consumer Terms](https://www.anthropic.com/legal/consumer-terms)

**Documented headless auth mechanics**
- Auth precedence, highest first:
  1. Cloud provider credentials
  2. `ANTHROPIC_AUTH_TOKEN` (bearer, for gateways)
  3. `ANTHROPIC_API_KEY`
  4. `apiKeyHelper`
  5. `CLAUDE_CODE_OAUTH_TOKEN` (from `claude setup-token`, "a one-year OAuth token" for "CI pipelines and scripts")
  6. Anthropic profile/federation credentials
  7. Subscription OAuth from `/login`

  — [Authentication](https://code.claude.com/docs/en/authentication)
- "Setting only `ANTHROPIC_BASE_URL`, with no gateway credential… a saved claude.ai login stays the active credential, so the subscription's usage limits and billing apply" — [Gateways](https://code.claude.com/docs/en/gateways)

**2025–2026 timeline of changes (dates; mostly secondary reporting, labeled)**
- **2026-01-09, about 02:20 UTC:** server-side safeguards blocked subscription OAuth outside the official CLI. OpenCode was the main tool hit, because it spoofed the Claude Code client. Users saw the error "This credential is only authorized for use with Claude Code." Thariq Shihipar (Anthropic) said: "Yesterday we tightened our safeguards against spoofing the Claude Code harness." API keys kept working — [paddo.dev (practitioner blog, 2026-01-10)](https://paddo.dev/blog/anthropic-walled-garden-crackdown/)
- Secondary sources say:
  - Cline and RooCode were also affected.
  - On 2026-02-19 Anthropic codified an "Authentication and credential use" section.
  - On 2026-03-19 OpenCode merged PR #18186, which removed its Anthropic OAuth plugin.

  These claims are unverified against primary sources — [Rida Kaddir (secondary)](https://ridakaddir.com/blog/post/did-anthropic-kill-opencode-claude-subscription-ban); [DEV Community (secondary)](https://dev.to/stevengonsalvez/anthropic-just-killed-third-party-harnesses-and-im-properly-gutted-dfk)
- **2026-04-04, 12pm PT:** subscribers would "no longer be able to use your Claude subscription limits for third-party harnesses including OpenClaw". The policy "applies to all third-party harnesses and will be rolled out to more shortly". Third-party use moved to pay-as-you-go extra usage, with a one-time credit. Claude Code and Cowork were unaffected. Boris Cherny said "subscriptions weren't built for the usage patterns of these third-party tools" — [TechCrunch](https://techcrunch.com/2026/04/04/anthropic-says-claude-code-subscribers-will-need-to-pay-extra-for-openclaw-support/); [HN thread](https://news.ycombinator.com/item?id=47633396)
- **2026-05-13 announcement, June 15 effective date (reportedly paused):** programmatic usage (Agent SDK, `claude -p`, GitHub Actions) on subscriptions was to move to a separate monthly credit at API rates: $20 for Pro, $100 for Max 5x and $200 for Max 20x. Later reports say Anthropic paused the change, so these usages "still draw from the subscription's usage limits" and a revised plan would come with advance notice. This is secondary and conflicting, and I could not read the Help Center notice directly — [MagnaCapax gist (secondary)](https://gist.github.com/MagnaCapax/d9177e35b355853f03c730dfcaa693ef); [Digital Applied (secondary)](https://www.digitalapplied.com/blog/anthropic-claude-credit-overhaul-june-15-2026); [GIGAZINE (secondary)](https://www.gigazine.net/gsc_news/en/20260514-anthropic-claude-agent-sdk-credits); [Agentic Wire (secondary)](https://www.agenticwire.news/article/claude-agent-sdk-200-credit-dev-guide)

**Usage limits and tracking**
- Help Center statements:
  - "Both Pro and Max plans have a five-hour session limit and a weekly limit."
  - Max has "a separate weekly limit for Fable".
  - "These limits are shared across Claude and Claude Code."
  - Usage credits let you continue past limits.

  — [Help Center: Using Claude Code with your Pro or Max plan](https://support.claude.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan)
- Machine-readable signals:
  - statusline `rate_limits.five_hour/seven_day.used_percentage` and `resets_at` — [Status line](https://code.claude.com/docs/en/statusline)
  - SDK `RateLimitEvent` with `utilization`, `resets_at` and types `five_hour`/`seven_day`/`seven_day_opus`/`seven_day_sonnet`/`overage` — [Python SDK reference](https://code.claude.com/docs/en/agent-sdk/python)
  - `/usage` plan bars, which fall back to "last-known usage" when the usage endpoint is rate limited — [Costs](https://code.claude.com/docs/en/costs)
- For subscribers, the session-cost figure in `/usage` "isn't relevant for billing purposes" — [Costs](https://code.claude.com/docs/en/costs)
- Practitioner and secondary claims (unverified):
  - Weekly-reset mechanics differ by source.
  - Since 2026-03-26, 5-hour windows reportedly drain faster on weekdays 5–11am PT.
  - Exact message counts aren't published.

  — [Leanware (secondary)](https://leanware.co/insights/claude-max-plan); [InventiveHQ (secondary)](https://inventivehq.com/blog/claude-code-rate-limits-explained)
- Claude model aliases today: `fable` resolves to Fable 5.1 ("for your hardest and longest-running tasks"), `best` to Fable where available, plus `opus`, `sonnet` and `haiku`. Pinned full model IDs are also accepted — [Model config](https://code.claude.com/docs/en/model-config)

### Inferences
- **Per-surface billing implications for a personal (single-user) Narwarp:**

  | Surface | Status |
  |---|---|
  | Interactive `claude` in a Narwarp PTY, background sessions, Remote Control, routines | Clearly fine on subscription |
  | `claude -p` / Agent SDK on the user's own login | Within "ordinary, individual usage" per the legal page. Exposed to the (paused) SDK-credit plan and the coming `--bare` default |
  | `--bare` or API key | Metered API billing |
  | Claude Code pointed at Ollama | No Anthropic billing at all |

- **If Narwarp is ever distributed:** each user must sign in through Anthropic's own flow in the unmodified binary. Narwarp must never read, store or proxy their tokens; for example, it must not run a LiteLLM-style proxy in front of a subscription session that sees OAuth bearer tokens. It must not use Claude Code branding as its product name.
- Make auth a pluggable profile in Narwarp (subscription / `CLAUDE_CODE_OAUTH_TOKEN` / API key / Ollama). Budget-aware scheduling should read `RateLimitEvent` utilization and statusline `rate_limits`, and shift low-value work to local models when `allowed_warning` appears.

### Gaps
- I found no primary Anthropic page confirming the SDK-credit pause or its current status. The pause wording comes only from secondary sources.
- Anthropic publishes no numeric 5-hour or weekly caps.
- I could not confirm whether the Feb 19, 2026 "ToS" change refers to the Consumer Terms (which still show effective 2025-10-08) or to the Claude Code legal page.

---

## 5. Ollama: current version and features (tool calling, structured outputs, thinking, streaming, embeddings, OpenAI/Anthropic-compatible APIs, Claude Code on Ollama, `ollama launch`, cloud, model management, keep-alive/concurrency, MLX)

### Takeaway
Ollama is at **v0.40.3 (latest stable, 2026-10-11)**, with v0.40.4 in pre-release (Oct 12). Key milestones:
- **MLX** became the default engine for supported architectures on Apple Silicon in v0.40.0.
- **Anthropic Messages API** compatibility arrived in v0.14.0 (Jan 2026).
- **`ollama launch claude`** arrived in v0.15 (Jan 2026). It runs the real Claude Code against local or cloud open models.
- **Typed "decision models"** shipped in v0.35 via `/v1/systemone`, aimed at triage and routing.

Claude Code on Ollama works for chat, edits, tools and subagents. Prompt caching, `tool_choice` and token counting are missing at the API level, and quality depends heavily on the model.

### Cited Findings
**Versions and recent releases**
- **v0.40.3** (Latest, Oct 11): `ollama launch claude` no longer warns about Claude connectors. Background model upgrades were paused.
- **v0.40.2** (Oct 8): `ollama launch claude` uses the model's full context length. Old models are upgraded in the background.
- **v0.40.1** (Oct 7): the server proxies cloud usage and balance APIs.
- **v0.40.4** (pre-release, Oct 12): updated llama.cpp and MLX.

  — [Ollama releases](https://github.com/ollama/ollama/releases)
- **v0.40.0:** "Models run on MLX on Apple Silicon by default… model architectures supported by the MLX runtime will automatically run on MLX". The release covers qwen3.8, gemma4, qwen3.6 and qwen3.5, the decision models (Nimble, tev1, clef, clef-flash), and embeddinggemma-2. The page shows the date "25 Sep", which is oddly earlier than v0.35.x (Sep 28–29); it may be the tag-creation date — [v0.40.0 release](https://github.com/ollama/ollama/releases/tag/v0.40.0); corroborated by [byteiota (secondary)](https://byteiota.com/ollama-040-mlx-default-apple-silicon/)
- **v0.35.0/0.35.1** (Sep 28–29): decision models via `/v1/systemone`; web search raised to 10 searches per response; Modelfile `CAPABILITY`. **v0.34.4** (Sep 23): "Structured outputs on thinking models apply in a single pass" and XGrammar updated. **v0.34.3:** `/api/show` advertises thinking levels — [Ollama releases](https://github.com/ollama/ollama/releases)

**MLX on Apple Silicon (verified)**
- The MLX preview shipped in 0.19 on 2026-03-30 (>6 mo) and required ">32GB of unified memory". Measurements on an M5 Max with Qwen3.5-35B-A3B:
  - 0.19 NVFP4: prefill 1810 tok/s, decode 112 tok/s
  - 0.18 Q4_K_M: 1154 and 58 tok/s
  - int4: 1851 and 134 tok/s

  The preview also added cache reuse across conversations "when using a shared system prompt with tools like Claude Code" — [Ollama blog: MLX preview](https://ollama.com/blog/mlx)
- 2026-06-11: the MLX engine became "up to 20% faster". NVFP4 "roughly halves the quality loss" of 4-bit quantization versus q4_K_M on Gemma 4 12B. A new **snapshot system** saves model state so subagents, parallel sessions, thinking turns and branches avoid re-prefill — [Ollama blog: MLX performance](https://ollama.com/blog/mlx-performance)
- 2026-06-29 (0.31): Gemma 4 multi-token prediction is "nearly 90% faster" on Apple Silicon on Aider polyglot (Gemma 4 12B nvfp4, M5 Max). It is on by default and auto-tuned — [Ollama blog: Gemma 4 MTP](https://ollama.com/blog/faster-gemma-4-mlx-mtp)

**APIs**
- Anthropic `/v1/messages` (local and `https://ollama.com/v1/messages`):
  - Supported: messages, streaming and function calling. Base64 images are supported; URL images are not.
  - Thinking is basic: `budget_tokens` is "accepted but not enforced".
  - **Not supported:** `count_tokens`, `tool_choice`, `metadata`, prompt caching (`cache_control`), batches, citations, PDF, and SSE error events.
  - Token counts are approximations.

  — [Ollama Anthropic compatibility](https://docs.ollama.com/api/anthropic-compatibility)
- OpenAI compatibility covers `/v1/chat/completions`, `/v1/completions`, `/v1/models`, `/v1/embeddings` and `/v1/responses`. The cloud does not support stateful Responses — [Ollama OpenAI compatibility](https://docs.ollama.com/api/openai-compatibility)
- Structured outputs take `format: "json"` or a JSON schema. "Ollama's Cloud currently does not support structured outputs" — [Ollama structured outputs](https://docs.ollama.com/capabilities/structured-outputs)
- Thinking:
  - `think` accepts true, false, null, or a named level from `/api/show` `thinking.values`; gpt-oss offers low, medium and high.
  - Output is split into `message.thinking` and `message.content`.

  — [Ollama thinking](https://docs.ollama.com/capabilities/thinking)
- Tool calling supports single, parallel and multi-turn agent loops, and tool calling with streaming. Streaming tool calls date from 2025-05-28 (>6 mo) — [Ollama tool calling](https://docs.ollama.com/capabilities/tool-calling); [Ollama blog: streaming tools](https://ollama.com/blog/streaming-tool)
- Embeddings: `/api/embed` returns L2-normalized vectors. Recommended models are embeddinggemma, qwen3-embedding and all-minilm. Use the same model for indexing and querying — [Ollama embeddings](https://docs.ollama.com/capabilities/embeddings)
- **Decision models (Sept 29, 2026):** `POST /v1/systemone` takes a text `state` plus named `questions` of three types: `choice` (with criteria), `noul` (yes/no probability) and `score` (ordinal). It returns `probabilities` and `confidence` per answer. It is "based on TypeSafe's Jev API" and pitched for "ticket triage, model routing, and content or safety moderation".
  - At launch the models were nimble (9B, Bespoke Labs), tev1 (4B) and tev1:0.8b (Together AI).
  - "Nimble 9B averaged 91ms per decision… on an M5 Max."
  - The example response used 841 input tokens and only 4 output tokens.

  — [Ollama blog: decision models](https://ollama.com/blog/ollama-now-supports-jev-style-decision-models)

**Claude Code on Ollama, `ollama launch`**
- Ollama v0.14.0+ is "compatible with the Anthropic Messages API", which makes Claude Code usable with open models. Setup: `ANTHROPIC_AUTH_TOKEN=ollama`, `ANTHROPIC_BASE_URL=http://localhost:11434`, then `claude --model gpt-oss:20b`. A context of at least 32K is recommended (2026-01-16, >6 mo) — [Ollama blog: Claude Code](https://ollama.com/blog/claude)
- `ollama launch` (v0.15+, 2026-01-23, >6 mo) sets up and runs Claude Code, OpenCode, Codex and Droid "with local or cloud models. No environment variables or config files needed". Example: "~23 GB VRAM required with 64000 tokens context length" for glm-4.7-flash — [Ollama blog: launch](https://ollama.com/blog/launch)
- Current docs:
  - `ollama launch claude --model gemma4:cloud --yes -- -p "…"` is the non-interactive form; args after `--` go to Claude Code.
  - Local models should be set to "a 64k+ context window".
  - `/loop` and Telegram channels work through `ollama launch claude -- --channels …`.
  - Direct cloud use: `ANTHROPIC_BASE_URL=https://ollama.com ANTHROPIC_AUTH_TOKEN=$OLLAMA_API_KEY claude --model glm-5.3-flash`.
  - "Hosted WebSearch and advanced tool controls are not fully supported."

  — [Ollama Claude Code integration](https://docs.ollama.com/integrations/claude-code)
- Claude Desktop can use Ollama as a "third-party gateway provider" (2026-08-25) — [Ollama blog: Claude Desktop](https://ollama.com/blog/claude-desktop)
- Claude Code 2.1.285 changed sessions behind a custom `ANTHROPIC_BASE_URL` to assume 1M context where models have it ("run `/autocompact 200k` if your gateway stops at 200K"). The CLI's `--autocompact` accepts "auto, or 100k–1M tokens" — [Claude Code CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md); [CLI reference](https://code.claude.com/docs/en/cli-reference)

**Cloud, pricing, privacy**
- Cloud models run without downloads (`gemma4:cloud` via a signed-in server, or `https://ollama.com/api/*` with `OLLAMA_API_KEY`). The docs say "We do not use them to train models" — [Ollama cloud](https://docs.ollama.com/cloud)
- New plans (2026-08-31):
  - Pro: $20/mo including $60 of usage
  - Max: $100 including $300
  - Team: $500 including $1,000

  There is "no 5-hour or weekly limits", per-token rates, "zero data retention" and US/EU hosting — [Ollama blog: transparent pricing](https://ollama.com/blog/transparent-pricing)
- Example cloud prices (per 1M input/output tokens) from the search page:

  | Model | Price |
  |---|---|
  | glm-5.3-flash | $0.15 / $0.50 |
  | deepseek-v4.1-flash | $0.30 / $1.20 |
  | mistral-large-4 | $0.68 / $2.09 |
  | glm-5.3 | $1.40 / $4.40 |
  | kimi-k3 | $3.00 / $15.00 |

  — [ollama.com search](https://ollama.com/search)
- Local-only mode: set `"disable_ollama_cloud": true` in `~/.ollama/server.json`, or `OLLAMA_NO_CLOUD=1` — [Ollama FAQ](https://docs.ollama.com/faq)

**Keep-alive, concurrency, context**
- Models stay loaded 5 min by default. `keep_alive` takes -1 (forever) or 0 (unload), and `OLLAMA_KEEP_ALIVE` sets it server-wide — [Ollama FAQ](https://docs.ollama.com/faq)
- Concurrency settings:
  - `OLLAMA_MAX_LOADED_MODELS` defaults to 3× the number of GPUs (or 3 on CPU).
  - `OLLAMA_NUM_PARALLEL` "default 1"; required RAM scales with NUM_PARALLEL × CONTEXT_LENGTH.
  - `OLLAMA_MAX_QUEUE` defaults to 512, and the server returns 503 when the queue is full.
  - Flash attention is automatic, and `OLLAMA_KV_CACHE_TYPE` defaults to f16.

  — [Ollama FAQ](https://docs.ollama.com/faq)
- Default context depends on memory: <24 GiB gives 4k, 24–48 GiB gives 32k, and ≥48 GiB gives 256k. `ollama ps` shows context and the GPU/CPU split. This conflicts with the FAQ's "4096 default", which is probably stale — [Ollama context length](https://docs.ollama.com/context-length); [Ollama FAQ](https://docs.ollama.com/faq)
- The memory scheduler measures exact memory before loading. This reduced OOMs (2025-09-23, >6 mo) — [Ollama blog: model scheduling](https://ollama.com/blog/new-model-scheduling)

### Inferences
- Because `ollama launch claude --yes -- -p …` runs the *same* Claude Code binary, Narwarp can use **one worker protocol** (stream-json, hooks, MCP tools, sessions) for both Claude and local agents. Only `ANTHROPIC_BASE_URL` and the model change, and the local runs incur no Anthropic billing or ToS exposure.
- Claude Code's large system prompt and tool definitions make prefill the bottleneck locally. MLX snapshot caching and MoE models (A3B/A4B) on M5-class chips are what make this tolerable.
- Claude Code's `--autocompact` floor of 100k suggests serving local models at ≥128K context for Claude Code use. That increases memory needs beyond the raw weight sizes.

### Gaps
- No independent benchmark of Claude Code itself running on local Ollama models was found, comparing task success against Claude models. Only vendor guides and setup tutorials turned up.
- The exact v0.40.0 publication date is ambiguous on the releases page.
- The `OLLAMA_NUM_PARALLEL` default may differ by engine (MLX vs GGML); the docs don't say.

---

## 6. Local model selection on Apple Silicon (as of Oct 2026): classification/routing, summarization, embeddings, tool calling, light coding — sizes, speeds, reliability techniques

### Takeaway
The October 2026 local landscape is dominated by a few families:
- **Qwen 3.5 / 3.6 / 3.8** (0.8B–122B; 27B dense and 35B-A3B MoE are the sweet spots)
- **Gemma 4** (E2B/E4B/12B/26B-A4B/31B)
- **gpt-oss-20b**
- MoE "always-on agent" models: Nemotron 3.5 Lightning, Meta Muse Glimmer, Laguna XS
- **purpose-built decision models** (nimble/tev1) for typed routing

For Narwarp's auto-sorting, decision models or a 4–9B model with schema-constrained output are reliable. Tool-calling quality drops sharply below about 9B: Qwen3.5 BFCL-V4 scores 66.1 at 9B versus 50.3 at 4B. Coding-agent quality approaches late-2025 frontier only at 27B+, and those numbers are vendor-reported.

### Cited Findings
**Ollama library state and download sizes (default tags, mostly q4)**

Captured 2026-10-13 from the [Ollama library](https://ollama.com/library?sort=newest) and tag pages (for example [gemma4 tags](https://ollama.com/library/gemma4/tags), [qwen3.6 tags](https://ollama.com/library/qwen3.6/tags), [qwen3.5 tags](https://ollama.com/library/qwen3.5/tags)):

- **General families**
  - **gemma4** (26.9M pulls; vision/tools/thinking/audio; updated 18 h ago):

    | Tag | Size | Context |
    |---|---|---|
    | e2b | 4.6 GB | 128K |
    | e4b | 6.6 GB | 128K |
    | 12b | 7.7 GB | 256K |
    | 26b (A4B MoE) | 16 GB | 256K |
    | 31b | 19 GB | 256K |

  - **qwen3.5** (22.2M pulls; all 256K context):

    | Tag | Size |
    |---|---|
    | 0.8b | 1.2 GB |
    | 2b | 2.7 GB |
    | 4b | 3.3 GB |
    | 9b | 6.6 GB |
    | 27b | 17 GB |
    | 35b (A3B) | 22 GB |
    | 122b | 81 GB |

  - **qwen3.6** (7.1M pulls): 27b is 18 GB (q4_K_M 17 GB) and 35b-a3b is 23 GB. It also has `-coding`, `-mtp`, `-mlx`, `nvfp4` and `mxfp8` variants.
  - **qwen3.8** (27b, 18 GB, 256K; 3.7M pulls; used as the v0.40.0 example).
  - **gpt-oss**: 20b is 14 GB and 120b is 65 GB, both 128K (13.8M pulls). Released Aug 2025 (>6 mo) but still current.
- **Agent and coding models**
  - **nemotron-3.5-lightning** (30B-A3B): 25 GB q4_K_M (1M ctx) or 23 GB MLX.
  - **muse-glimmer** (Meta, 30B, Apache 2.0, "built for always-on local agents"): 18 GB.
  - **granite4.2** (IBM): 3b 2.2 GB, 8b 5.3 GB, 30b 18 GB; covers "tool use, and JSON output".
  - **lfm2.5** (8B-A1B, "fast, reliable tool calling on consumer hardware"): 5.2 GB.
  - **ministral-3**: 3b 3.0 GB, 8b 6.0 GB, 14b 9.1 GB.
  - **devstral-small-2** (24b, 15 GB, 384K; 10 months old).
  - **qwen3-coder** 30b-a3b (19 GB; about 1 year old, possibly stale).
  - **qwen3-coder-next** (52 GB).
  - **glm-4.7-flash** (19 GB).
  - **laguna-xs-2.1** (33B-A3B, 20 GB).
  - **north-mini-code-1.0** (Cohere 30B-A3B, 19 GB, 488K).
- **Decision models**
  - nimble 9b: q4_K_M 5.6 GB, default 9.3 GB
  - nimble-v3 (new)
  - tev1: 4b q4_K_M 2.7 GB, 0.8b ~0.8 GB
  - strands-decider 2b (2.0 GB, 4K ctx)
  - clef-flash 9b and clef 27b (Cloudflare)
  - laya 421M (ModernBERT)
- **Embeddings**
  - **embeddinggemma-2** (Gemma 4-based, multimodal): 270m 378 MB up to 740m 1.3 GB.
  - **qwen3-embedding**: 0.6b 639 MB (32K), 4b 2.5 GB, 8b 4.7 GB (40K).
  - Also nomic-embed-text-v2-moe, embeddinggemma (300m), bge-m3 and mxbai-embed-large.

**Benchmarks (vendor-reported unless noted)**
- Qwen3.5 tool-calling table: BFCL-V4 is **66.1 for Qwen3.5-9B** and **50.3 for 4B**, versus Qwen3-30B-A3B-Thinking-2507 at 42.4 and Qwen3-Next-80B-A3B at 49.7. TAU2-Bench is 79.1 (9B) and 79.9 (4B). IFEval is 91.5 (9B) and 89.8 (4B). The context is 262,144 native, and Qwen advises "at least 128K tokens to preserve thinking capabilities" — [Qwen3.5-9B model card](https://huggingface.co/Qwen/Qwen3.5-9B)
- Secondary sources report Qwen3.5 BFCL-V4 values of 0.8B 25.3, 2B 43.6 and 397B-A17B 72.9, with a "sharp drop from 9B to 4B" — [vdf.ai (secondary)](https://vdf.ai/blog/best-local-llm-for-agents-tool-calling/); [llm-stats BFCL-V4](https://llm-stats.com/benchmarks/bfcl-v4)
- Qwen3.6-27B scores **77.2 on SWE-bench Verified** and **59.3 on Terminal-Bench 2.0**. Qwen3.6-35B-A3B scores 73.4 and 51.5, Gemma4-31B 52.0 and 42.9, and Claude 4.5 Opus 80.9 and 59.3. These use Qwen's internal bash+edit scaffold, not Claude Code — [Qwen3.6-27B model card](https://huggingface.co/Qwen/Qwen3.6-27B)
- On MCPMark, Qwen3.6-35B-A3B scores 37.0 versus Gemma4-26B-A4B 14.2. On Tool Decathlon it scores 26.9 versus 12.0. On SWE-bench Verified, Gemma4-26B-A4B scores only 17.4 — [Qwen3.6-35B-A3B model card](https://huggingface.co/Qwen/Qwen3.6-35B-A3B)
- Gemma 4 Tau2 (average of 3):

  | Model | Tau2 |
  |---|---|
  | 31B | 76.9% |
  | 26B A4B | 68.2% |
  | 12B | 69.0% |
  | E4B | 42.2% |
  | E2B | 24.5% |
  | Gemma 3 27B | 16.2% |

  Gemma 4 has "Native support for structured tool use". Context is 128K (E2B/E4B) or 256K (12B/26B/31B). The 26B A4B has 25.2B total and 3.8B active parameters — [Gemma 4 12B model card](https://huggingface.co/google/gemma-4-12b-it)
- Decision models: Ollama reports mean accuracy across 13 human-labeled public datasets (3,880 decisions) for Nimble and Tev1 versus Bespoke's Jev 1.13. Nimble 9B averages 91 ms per decision on an M5 Max — [Ollama blog: decision models](https://ollama.com/blog/ollama-now-supports-jev-style-decision-models)

**Apple Silicon speed and RAM**
- Ollama-measured on an M5 Max: Qwen3.5-35B-A3B at 112–134 tok/s decode and about 1.8k tok/s prefill on MLX — [Ollama blog: MLX preview](https://ollama.com/blog/mlx)
- Practitioner aggregator (internally inconsistent):

  | Machine | Qwen3.6-35B-A3B | Gemma 4 26B-A4B | Gemma 4 31B dense |
  |---|---|---|---|
  | M5 Max 128 GB | 55 tok/s | 50 tok/s | 26 tok/s |
  | M4 Pro 24 GB | 32 tok/s | 28 tok/s | — |

  An update note on the same page claims about 126 tok/s for Qwen3.6-35B-A3B on M5 Max — [LLMCheck (practitioner)](https://llmcheck.net/blog/qwen-36-vs-gemma-4-deep-technical-comparison/)
- Other practitioner reports: Gemma-4-26B-A4B at 59.9 tok/s on an M5 MacBook Pro via llama.cpp, the 31B dense at 11.9 tok/s, and the 26B-A4B at 16.2 tok/s on a MacBook Air M5 32 GB. Attribution among these roundup sites is unclear — [InsiderLLM (practitioner)](https://insiderllm.com/guides/best-local-llms-mac-2026/); [LLMCheck benchmarks (practitioner)](https://llmcheck.net/benchmarks)
- Memory bandwidth, which bounds decode speed: M4 120 GB/s, M4 Pro 273 GB/s, M4 Max 546 GB/s — [Wikipedia: Apple M4](https://en.wikipedia.org/wiki/Apple_M4)
- "The main M5 improvement is prompt processing, not generation". Projected 32B prompt processing is about 210 tok/s on M4 Pro versus about 700 on M5 Pro, while generation only goes from 11.4 to 13.0. These figures are labeled projections — [Nordic Silicon (secondary, projections)](https://nordicsilicon.io/blog/mac-mini-m6-m5-pro-llm-benchmarks)

**Reliability techniques**
- Grammar-constrained decoding: Ollama applies JSON-schema `format`, and its structured-outputs stack uses XGrammar (updated in v0.34.4). Structured outputs on thinking models now apply "in a single pass" — [Ollama releases](https://github.com/ollama/ollama/releases); [Ollama structured outputs](https://docs.ollama.com/capabilities/structured-outputs)
- Typed questions with calibrated `probabilities`/`confidence` come from decision models (`choice`/`noul`/`score`) — [Ollama blog: decision models](https://ollama.com/blog/ollama-now-supports-jev-style-decision-models)
- Small dedicated function-calling model: functiongemma (270M) is "fine-tuned explicitly for function calling" — [Ollama library](https://ollama.com/library)

### Inferences
- **Auto-sorting prompts into project, Work/Life mode and column/priority.** Use a decision model via `/v1/systemone`:
  - a `choice` over the project list, with descriptions as criteria;
  - a `choice` for Work/Life;
  - a `score` for urgency;
  - a `noul` for "needs Linear issue?".

  Calibrated confidence gives a natural escalation threshold: below it, ask the user or ask Claude. If you avoid the newer API, the fallback is qwen3.5:4b/9b with a JSON schema whose fields are `enum`s of project IDs, plus 3–5 few-shot examples drawn from past tickets via embedding kNN, at temperature 0.
- **Summarization (boards, handoffs, digests):** gemma4:12b or qwen3.5:9b on 16–32 GB. Use qwen3.6:35b-a3b or gemma4:26b (MoE, fast) on 64 GB+.
- **Embeddings:** qwen3-embedding:0.6b, or embeddinggemma-2:270m/440m, is enough for ticket dedupe and similarity at low RAM. qwen3-embedding:4b/8b serves higher-quality retrieval on bigger machines.
- **Tool-calling life agent (local-only data):** qwen3.5:9b is the minimum credible size per BFCL-V4. At 64 GB+, qwen3.6:35b-a3b (MCPMark 37.0) is preferable to Gemma 4 26B-A4B, whose vendor-reported tool and coding numbers are much lower.
- **Light coding:** qwen3.6:27b or qwen3.8:27b (about 18 GB weights, plus KV cache at ≥64–128K context), via `ollama launch claude`, on 64 GB+ machines. On 32 GB, only small fixes, scripts and explanations are realistic.
- **Hardware tiers.** These are inferences from the cited tag sizes. They assume macOS reserves part of unified memory for the system and other apps, and that agents need long contexts.

  | Unified memory | Router | Summarizer / tool agent | Embeddings | Coding |
  |---|---|---|---|---|
  | **16 GB** | tev1:4b (2.7 GB) or qwen3.5:4b | gemma4:e4b / qwen3.5:4b | embeddinggemma-2:270m | Claude only |
  | **32 GB** | nimble:9b q4_K_M (5.6 GB) | qwen3.5:9b or gemma4:12b | qwen3-embedding:0.6b | No resident coder; at most one ~17–18 GB model with short context, swapped in on demand |
  | **64 GB** | nimble:9b | qwen3.6:35b-a3b (23 GB) | qwen3-embedding:4b | qwen3.6:27b swapped in for coding tickets; `OLLAMA_MAX_LOADED_MODELS=3`, `NUM_PARALLEL=2` |
  | **128 GB** | nimble:9b | qwen3.6:35b-a3b | qwen3-embedding:8b | qwen3.6:27b or qwen3.8:27b resident; optionally qwen3.5:122b (81 GB), gpt-oss:120b (65 GB) or qwen3-coder-next (52 GB); `NUM_PARALLEL` 2–4 |

  The 16 GB tier keeps the resident total under about 10 GB.

### Gaps
- The official Berkeley BFCL leaderboard (JS-rendered) was not read. No independent BFCL numbers were found for Gemma 4, gpt-oss-20b, nimble or tev1.
- No consistent, methodology-controlled tok/s table exists for current models across M1–M5 base, Pro and Max. The figures above are mixed vendor and practitioner data.
- Decision-model accuracy is reported only by Ollama and Bespoke Labs; there is no independent replication.
- The macOS GPU wired-memory default fraction was not verified in this research.

---

## 7. Local-plus-cloud collaboration: local model as triager/dispatcher/summarizer/watchdog with Claude as worker; routers (LiteLLM etc.); privacy routing

### Takeaway
The robust pattern is for Narwarp to own the routing decision itself. Local decision or small models classify, summarize and watch, and Claude Code sessions do the heavy coding work.

A transparent proxy in front of Claude Code needs care. With a subscription, a proxy would sit in the OAuth path, which conflicts with the "don't intermediate tokens" rule. With LiteLLM, there is a recent supply-chain history to weigh. Privacy routing is straightforward with Ollama's local-only mode, and Ollama Cloud is a ZDR fallback for open models.

### Cited Findings
- Claude Code can route through gateways. `ANTHROPIC_BASE_URL` plus `ANTHROPIC_AUTH_TOKEN` (bearer) replaces the subscription credential. Setting only the base URL keeps the claude.ai login active, so "subscription's usage limits and billing apply" — [Gateways](https://code.claude.com/docs/en/gateways); [LLM gateway](https://code.claude.com/docs/en/llm-gateway); [Authentication](https://code.claude.com/docs/en/authentication)
- The Claude Code docs say: "Several large enterprises reported using LiteLLM… to track spend by key". They add that "This project is unaffiliated" — [Costs](https://code.claude.com/docs/en/costs)
- LiteLLM PyPI versions 1.82.7 and 1.82.8 were compromised on 2026-03-24 (>6 mo) via a hijacked maintainer account. The malware stole SSH, cloud and Kubernetes credentials, and 1.82.8 used a `.pth` launcher that ran at Python startup. The bad versions were yanked the same day — [NetSPI](https://www.netspi.com/blog/executive-blog/ai-ml-pentesting/litellm-supply-chain-compromise/); [Comet](https://www.comet.com/site/blog/litellm-supply-chain-attack/)
- Developers "may not collect, store, or intermediate Claude.ai credentials or session tokens" — [Claude Code legal & compliance](https://code.claude.com/docs/en/legal-and-compliance)
- Mods can "send one request to a different model" from inside Claude Code — [Mods overview](https://code.claude.com/docs/en/plugins/mods/overview)
- Ollama local-only mode: `disable_ollama_cloud: true` or `OLLAMA_NO_CLOUD=1` — [Ollama FAQ](https://docs.ollama.com/faq)
- Ollama Cloud claims "zero data retention, hosted in the US and Europe" and that "we never train on your data" — [Ollama blog: transparent pricing](https://ollama.com/blog/transparent-pricing); [Ollama cloud](https://docs.ollama.com/cloud)
- Practitioner guidance: local models are "not a drop-in replacement" for complex reasoning, multi-file changes and debugging. Use them for private, simple or high-volume work — [KDnuggets (practitioner)](https://kdnuggets.com/pairing-claude-code-with-local-models); [Morph (practitioner)](https://www.morphllm.com/ollama-claude-code)
- NVIDIA's playbook runs Claude Code against local `qwen3.6:27b` via Ollama with a single command — [NVIDIA build playbook](https://build.nvidia.com/playbooks/local-coding-agent/claude-code)

### Inferences
Narwarp roles:
1. **Triager.** A decision model labels each new prompt with project, mode, urgency and whether it needs Claude. Low-confidence results go to the user.
2. **Dispatcher.** Narwarp code, not an LLM, picks the worker:

   | Ticket type | Worker |
   |---|---|
   | Work coding | Claude Code SDK worker on subscription |
   | Life-mode ticket | Local Ollama agent, with cloud disabled |
   | Trivial or bulk work | `ollama launch claude` with a local coder |
   | Overflow when Claude `RateLimitEvent` reports `allowed_warning` | Local or Ollama Cloud models |

3. **Summarizer.** A local model condenses board threads and session transcripts (`get_session_messages`) into handoff notes.
4. **Watchdog.** Rules run over `claude agents --json`, hook events and OSC 7501 states, with a small model to phrase alerts. It flags tickets stuck in "blocked", repeated permission denials, budget overruns, or a session idle with an unmet Stop condition.

Two further choices:
- **Prefer explicit per-request routing in Narwarp** (two clients: Claude SDK and Ollama) over a LiteLLM proxy for a single-user tool. If LiteLLM is used, pin versions, verify hashes, and never place it in the subscription OAuth path.
- **Privacy routing.** Tag data sources (email, calendar, personal Discord DMs) as `life/private`. Policy enforces local-only models for those tickets. Claude sees only redacted summaries, and only if the user approves.

### Gaps
- No primary documentation was found on a supported way to mix Claude (subscription) and local models inside *one* Claude Code process. Subagents use Claude models, and the model aliases point to one provider at a time. This would need testing (for example a mod-based reroute, or `ANTHROPIC_DEFAULT_*_MODEL` with a gateway).
- No measured data was found on local-triage accuracy for personal "project sorting" tasks specifically.

---

## 8. Recommended agent-runtime layer for Narwarp (which Claude Code surface for which job; which local models for which roles; hardware tiers)

### Takeaway
Build Narwarp's agent runtime on **official, unmodified Claude Code surfaces**:
- Python Agent SDK `ClaudeSDKClient` workers for tickets
- interactive `claude` panes for hands-on work
- HTTP hooks + OSC 7501 + `claude agents --json` for monitoring
- a custom channel for pushing integration events into live sessions
- Remote Control for phone steering

Add **Ollama (MLX)** for routing (decision models), summarization, embeddings and private life agents. Use `ollama launch claude` when you want local coding with the *same* harness. Keep auth pluggable because Anthropic's subscription policy for programmatic use is still moving.

### Cited Findings
- Each building block's evidence is in sections 1–7:
  - `ClaudeSDKClient` capabilities — [Python SDK reference](https://code.claude.com/docs/en/agent-sdk/python)
  - HTTP hooks and 33 events — [Hooks reference](https://code.claude.com/docs/en/hooks)
  - OSC 7501 (v2.1.295) — [Claude Code CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)
  - `claude agents --json` — [Agent view](https://code.claude.com/docs/en/agent-view)
  - Channels — [Channels](https://code.claude.com/docs/en/channels)
  - Remote Control — [Remote Control](https://code.claude.com/docs/en/remote-control)
  - Routines — [Routines](https://code.claude.com/docs/en/routines)
  - Decision models — [Ollama blog](https://ollama.com/blog/ollama-now-supports-jev-style-decision-models)
  - MLX default — [v0.40.0 release](https://github.com/ollama/ollama/releases/tag/v0.40.0)
  - `ollama launch claude` — [Ollama Claude Code integration](https://docs.ollama.com/integrations/claude-code)
  - Policy — [Claude Code legal & compliance](https://code.claude.com/docs/en/legal-and-compliance)

### Inferences
**A. Claude Code surface per Narwarp job**

| Narwarp job | Surface | Why / notes |
|---|---|---|
| Hands-on coding in a Narwarp pane | Interactive `claude` (subscription) in a PTY; optionally `--remote-control`, `-n <ticket>`, `-w` worktree | Clearly permitted "ordinary use". Narwarp implements OSC 7501 for per-pane status and reads statusline JSON for 5h/7d usage |
| Autonomous ticket worker (start, follow-ups, handoffs) | Python Agent SDK `ClaudeSDKClient` per ticket | Store `session_id` on the ticket; `resume`/`fork_session` for handoffs; `can_use_tool` routes to the board or notifications; in-process MCP tools for board, ticket and Linear ops; `verbatim_prompts=True` for external text; `setting_sources` set explicitly; `max_budget_usd`/`task_budget`; read `RateLimitEvent` |
| Fire-and-forget parallel jobs without Narwarp owning processes | `claude --bg` + poll `claude agents --json --all` | Supervisor survives terminal close and sleep; `waitingFor` maps to a "Blocked on you" lane; quota scales with session count |
| Status, telemetry and context injection | Settings-level HTTP hooks POSTing to the Narwarp daemon (SessionStart/UserPromptSubmit `additionalContext`, Notification, PermissionRequest, Stop, SubagentStop) | Works for interactive, `-p`, SDK and background sessions alike, which avoids the Python SDK's missing SessionStart/SessionEnd callbacks |
| Push Discord/Linear/GitHub/email events into a live session | Custom "narwarp" channel MCP server (`claude/channel` capability) | Research preview: needs `--dangerously-load-development-channels` and its contract may change, so wrap it. Official Discord/Telegram plugins work for phone chat |
| Phone steering and approvals | Remote Control (`claude remote-control --spawn worktree`) and/or channel permission relay | Subscription-only; outbound HTTPS only |
| Agent ↔ agent messages | Narwarp board exposed as an MCP server to all agents; Claude-native `SendMessage` as a bonus between Claude sessions | Relayed messages carry no user authority, which is a good safety default |
| Scheduled and calendar-driven work | Narwarp's own scheduler launching SDK workers; cloud **Routines** only for GitHub-event or always-on jobs; `/loop` for in-session polling | `/loop` tasks expire after 7 days. Routines need repos in the cloud and draw subscription usage, with hourly caps |
| Avoid or defer | Agent Teams (experimental; no `-p`/SDK); third-party harnesses using subscription OAuth (blocked and against policy); proxies in the OAuth path | — |
| Optional | ACP adapter (`@agentclientprotocol/claude-agent-acp`) if Narwarp standardizes on ACP across Claude, Codex and Gemini; Claude Code mods if you want UI inside Claude Code panes | — |

**B. Local model roles (Ollama ≥0.40, MLX default on Apple Silicon)**

| Role | Primary model | Fallback | Interface |
|---|---|---|---|
| Router/classifier (project, Work/Life, urgency, "needs Claude?") | `nimble` 9B (or `tev1:4b` on 16 GB) | `qwen3.5:4b`/`9b` with JSON-schema `enum` + few-shot | `/v1/systemone` decision API; structured outputs fallback |
| Summarizer (boards, transcripts, digests) | `gemma4:12b` / `qwen3.5:9b` (≤32 GB); `qwen3.6:35b-a3b` (64 GB+) | `gemma4:e4b` | `/api/chat`; `think:false` for speed |
| Embeddings (dedupe, similar tickets, kNN few-shot) | `qwen3-embedding:0.6b` or `embeddinggemma-2` | `qwen3-embedding:4b/8b` on 64 GB+ | `/api/embed` (L2-normalized) |
| Private life-management tool agent | `qwen3.5:9b` (32 GB) → `qwen3.6:35b-a3b` (64 GB+) | `granite4.2:8b`, `lfm2.5` | `/api/chat` tools or Anthropic `/v1/messages`; `OLLAMA_NO_CLOUD=1` profile |
| Watchdog / alert phrasing | `tev1` or `qwen3.5:4b` + rules | — | Hook events + `claude agents --json` |
| Local coding agent | `qwen3.6:27b` / `qwen3.8:27b` (64 GB+); `qwen3-coder-next` or `qwen3.5:122b` (128 GB) | Ollama Cloud (`glm-5.3`, `kimi-k2.7-code`) for Work, never Life | `ollama launch claude --model … --yes -- -p …` |

**C. Hardware tiers.** See the table in section 6. The short version:

| Unified memory | What fits |
|---|---|
| 16 GB | Router + small summarizer + tiny embedder; all coding goes to Claude |
| 32 GB | 9B router/summarizer/tool agent + embedder resident; no resident local coder |
| 64 GB | Router + 35B-A3B generalist + embedder resident; 27B coder swapped in |
| 128 GB | Everything resident, plus a 27B coder or a 65–81 GB large model; `OLLAMA_NUM_PARALLEL` 2–4 |

Set `OLLAMA_KEEP_ALIVE=-1` for the router and embedder so they never cold-start. Keep the context ≥64K, and ≥128K for Claude Code-on-Ollama.

**D. Guardrails**
- Pin Claude Code and SDK versions; the CLI shipped 542 versions in about 20 months.
- Abstract auth (subscription / setup-token / API key / Ollama).
- Expect `--bare` to become the `-p` default, and test SDK workers with an API key path.
- Never store or proxy OAuth tokens.
- If Narwarp is ever shared, each user signs in through Anthropic's own flow.

### Gaps
- None of this was load-tested. Actual concurrent-session limits on a Pro or Max plan, the decision-model accuracy on Narwarp-style project taxonomies, and Claude-Code-on-Ollama task success all need empirical trials.
- Process supervision (Narwarp-owned versus Claude's `--bg` supervisor), board schema and calendar integration are covered by other researchers. The recommendations above assume those designs can consume HTTP-hook events and `claude agents --json`.
