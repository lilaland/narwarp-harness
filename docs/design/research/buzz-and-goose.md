# Buzz (Block OSS) and goose: research for Narwarp

Researched 2026-10-23/24. Citation keys:
`B/` = https://github.com/block/buzz/blob/9e8c9fc04daf20259cd259dcaec89109587cc7f5/ (HEAD, 2026-10-23).
`G/` = https://github.com/aaif-goose/goose/blob/c3bfc5f7b0e230180ba3164ec1ee84fd951eaf0d/ (HEAD, 2026-10-23).
`git:` = a command I ran in a blobless clone of block/buzz at that commit. I built and ran nothing.

## What Buzz is

Buzz exists and it is about agents. It is a self-hostable workspace where humans and AI agents share channels, DMs, canvases, workflows and git repos. Every action is a signed Nostr event in one relay log (`B/README.md`).
- **Release:** Block launched it on 2026-07-21. Block also runs a hosted relay at buzz.xyz, free during the beta (https://siliconangle.com/2026/07/21/block-launches-buzz-open-source-workspace-humans-ai-agents/).
- **History:** the first real commit is "Initial release — Sprout Nostr relay with enterprise extensions" (2026-03-09). The project was renamed with "Rename desktop app to Buzz (#960)" (2026-06-11) (`git: log --reverse; log --grep=rename`).
- **Maintainers:** Block, Inc., through `@block/buzz-oss-team` (`B/.github/CODEOWNERS`).
- **Relation to goose:** goose is the default agent runtime (`B/crates/buzz-acp/README.md`), but Buzz is a separate repo.

## Architecture

**Buzz**
- **Relay:** a Rust relay is the single source of truth. Postgres holds events and search, Redis does pub/sub and presence, S3/MinIO holds media (`B/ARCHITECTURE.md`, §1, §8).
- **Running agents:** the `buzz-acp` harness bridges relay events to ACP over stdio and spawns 1–32 agent subprocesses. The default is `goose acp`. It also runs Codex (`codex-acp`), Claude Code (`claude-agent-acp` with an API key), `buzz-agent`, presets (Cursor, OpenCode, Amp, …) and custom JSON harnesses (`B/crates/buzz-acp/README.md`).
- **Models:** `buzz-agent` talks to Anthropic, OpenAI-compatible endpoints (including Ollama), OpenRouter and Databricks (`B/crates/buzz-agent/README.md`).
- **Local or remote:** agents run locally as desktop-managed processes. Remote agents go through a `buzz-backend-<id>` provider; the first is Kubernetes. The desktop keeps no control channel, and agents shut themselves down after a period of silence (`B/docs/remote-agents.md`).
- **Isolation:**
  - Each channel gets its own ACP session by default; a flag switches this to one per thread (`B/crates/buzz-acp/src/scope.rs#L34`).
  - Each session gets its own MCP server instances (`B/VISION_AGENT.md`).
  - Worktrees are only a prompt instruction (`B/crates/buzz-acp/src/base_prompt.md#L134`).
  - "Session isolation is not a security sandbox" (`B/crates/buzz-acp/TASKS.md#L79`).
- **Supervision:**
  - At most one prompt is in flight per scope, and queued @mentions are batched into the next prompt (`B/ARCHITECTURE.md#L779`).
  - A 620 s idle timeout resets on any output, and a 7,200 s cap bounds each turn.
  - The harness respawns crashed agents, and the owner can send `!cancel`, `!rotate` and `!shutdown` (`B/crates/buzz-acp/README.md`).
- **Resume:**
  - `buzz-agent` offers no `session/load`, and `buzz-acp` persists no state (`B/crates/buzz-agent/README.md#L342`, `B/ARCHITECTURE.md#L799`).
  - When context fills, the agent summarizes its own history and continues (`B/VISION_AGENT.md`). After a restart it rebuilds from "todos, memory, and the thread" (`base_prompt.md`).

**goose**
- **Desktop and server:** the desktop app runs its own `goose serve` in the background, an ACP server over HTTP and WebSocket authenticated by `X-Secret-Key`. It can also use a remote one (`G/documentation/docs/guides/remote-goose-server.md#L9`).
- **ACP:** `goose acp` serves ACP over stdio for editors. goose can also use other ACP agents (Claude, Codex) as providers and passes its extensions through as MCP servers. Those providers can't resume or fork sessions (`G/documentation/docs/guides/acp-providers.md#L19`).
- **Sessions:** stored in SQLite (`sessions.db`) and shared by Desktop and CLI. `goose session -r` resumes and `--fork` copies (`G/documentation/docs/guides/sessions/session-management.md`).
- **Subagents:** the `delegate` tool runs up to 5 at once, each with a 5-minute timeout, and they inherit the parent's extensions unless restricted. They can't spawn subagents or change extensions or schedules, and they only run in Autonomous mode (`G/documentation/docs/guides/context-engineering/subagents.mdx`).
- **Scheduler:** `goose schedule add --cron … --recipe-source` adds a scheduled recipe. The scheduler is off by default on `serve` and `acp` (`G/documentation/docs/guides/goose-cli-commands.md#L653-L739`).

## Tasks and todos

**Buzz**
- **Projects and issues:** a project (`kind:30621`) groups NIP-34 repos and has a home channel. Issues are `kind:1621`, with statuses in kinds 1630–1633. Clients trust an assignment only when the issue author or repo owner signed it, or when people assign themselves (`B/VISION_PROJECTS.md`, `B/crates/buzz-cli/src/lib.rs`).
- **Job lifecycle kinds:** 43001–43006 (requested through error) are defined and shown in the activity feed (`B/crates/buzz-core/src/kind.rs#L563`). I found no producer in `buzz-acp` (inference, from grep).
- **Session todos:** the `todo` MCP tool keeps an in-memory list per process and objects to the end of a turn while items are open (`B/crates/buzz-dev-mcp/src/todo.rs`). These todos "do not appear on the project" (`base_prompt.md#L28`).
- **Linking tasks to sessions:**
  - @mentions or heartbeats wake an agent into its channel or thread session.
  - `buzz-acp run --task` runs one strict JSON task (`B/crates/buzz-acp/TASKS.md`).
  - Runs of a YAML workflow that reach an approval gate currently fail (`B/ARCHITECTURE.md#L974`).
- **How the goose team uses it:** a goose recipe assigns GitHub issues, opens a Buzz channel per issue, mirrors each issue's phase into the channel topic, and wakes a bot by explicit mention (`G/buzz/README.md`).

**goose recipes:** YAML with `parameters`, `extensions` (including an `available_tools` allowlist), `response.json_schema` and `retry.checks`. `retry.checks` are shell commands that must exit 0; when one fails, goose resets the history and retries (`G/documentation/docs/guides/recipes/recipe-reference.md#L37-L56`, `#L168`, `#L434-L445`).

## Tools and connectors

**Buzz**
- **MCP servers:** `buzz-acp` passes one optional MCP server at ACP `session/new`, and that server's environment includes the agent's private key, `BUZZ_PRIVATE_KEY` (`B/crates/buzz-acp/src/lib.rs#L6453-L6490`). The desktop uses `buzz-dev-mcp` (shell, str_replace, todo, rg) for Codex and buzz-agent (`B/desktop/src-tauri/src/managed_agents/discovery/catalog.rs`).
- **Persona packs:**
  - A pack's `.mcp.json` is merged with each persona's servers, and on a name clash the persona's entry wins.
  - The result is delivered through `session/new`, and nothing is written to the agent's working directory.
  - `${VAR}` interpolation and pack hooks are not implemented yet (`B/crates/buzz-persona/PERSONA_PACK_SPEC.md#L471-L510`).
- **Credentials:** the desktop keeps agents' keys in the OS keyring (`B/desktop/src-tauri/src/secret_store.rs`). Users cannot override reserved environment keys such as identity, relay URL, binary and gates (`B/desktop/src-tauri/src/managed_agents/reserved_env_keys.rs`).
- **Scoping:** agents are "scoped by identity, not by permission flags" (`B/README.md#L48`), and channel membership is the only relay gate (`B/ARCHITECTURE.md#L904`).
- **Planned:** keyless agents behind a broker that holds the key (draft, 2026-08-27: `B/docs/practical-information-flow-for-buzz-agents.md`). So far only the contract exists (#6742, 2026-08-26: `B/crates/buzz-sdk/src/broker/mod.rs`) (`git: log --grep=broker`).
- **Lifecycle hooks:** opt-in `_Stop` and `_PostCompact` MCP hooks (`B/docs/MCP_DRIVEN_HOOKS.md`).

**goose**
- **Extensions:** extensions are MCP servers configured in `~/.config/goose/config.yaml` with `cmd`, `envs`, `env_keys` and `timeout` (`G/documentation/docs/getting-started/using-extensions.md#L319`).
- **Secrets:** stored in the system keyring, with `secrets.yaml` as the fallback (`G/documentation/docs/guides/config-files.md#L242`).
- **Permissions:** the modes are Autonomous (the default), Manual, Smart and Chat (`G/documentation/docs/guides/managing-tools/goose-permissions.md#L35`). Each tool can be set to Always, Ask or Never.
- **Allowlist:** `GOOSE_ALLOWLIST` points to a URL listing allowed commands, matched exactly (`G/documentation/docs/guides/allowlist.md#L14`).

## Terminal and UI integration

**Buzz**
- **Desktop:** a Tauri and React app (`B/README.md`).
- **Embedded terminal:** each channel or thread can have a PTY terminal that exports only public IDs such as `BUZZ_CHANNEL_ID` (`B/desktop/src-tauri/crates/buzz-terminal/src/context.rs#L88`), and builds its environment from an allowlist (`env_fence.rs#L17`).
- **Other surfaces:** `buzz-cli` takes JSON in and gives JSON out. `buzz-agent` works in Zed and JetBrains (`B/VISION_AGENT.md`). Mobile clients are "being wired up."
- **Chat:** I found no Slack bridge (grep).

**goose:** `goose term init zsh` enables `@goose` in the shell (`G/documentation/docs/guides/terminal-integration.md`), and ACP editors can drive it.

## Maturity and license

**Buzz**
- **License:** Apache-2.0 (`B/LICENSE`).
- **GitHub, as of 2026-10-23:** 36,303 stars, 4,810 forks, and 3,659 open issues and PRs combined (https://github.com/block/buzz).
- **Commit activity:** 3,010 commits since 2026-03-06, from 122 distinct author names. In the 30 days before 2026-10-23 there were 305 commits by 36 authors (`git: rev-list --count HEAD; log --since=2026-09-23`).
- **Releases:** the latest is Desktop v0.5.28, published 2026-10-09 (https://github.com/block/buzz/releases).
- **Self-description:** "Not finished."
- **Docs:** they live in the repo. block.github.io/buzz only hosts terms and support pages.

**goose**
- **License:** Apache-2.0.
- **GitHub, as of 2026-10-23:** 55,594 stars (https://github.com/aaif-goose/goose).
- **Latest release:** v1.56.0, published 2026-10-23 (https://github.com/aaif-goose/goose/releases).
- **Home:** the project is now under the Agentic AI Foundation (AAIF) at the Linux Foundation (`G/README.md`).

## What Narwarp should copy (ranked)

1. **Build every child's environment from an allowlist.** narwarpd should clear the inherited environment before spawning `claude --bg`, MCP servers and Warp ticket terminals, then add only listed keys. Buzz documents a real leak: an abandoned branch passed 68 variables to child processes, including the private key (`env_fence.rs`).
2. **Make the MCP server a broker.** Offer a closed set of named actions, and derive the requester, ticket and scope from the session credential, never from tool arguments. Reject unknown fields, and let no secret cross the boundary (`B/crates/buzz-sdk/src/broker/mod.rs`). Run credentialed connectors on the daemon side.
3. **Key sessions per ticket and supervise them.** Keep one prompt in flight per ticket, batch messages that arrive during a turn, and handle owner-only cancel and rotate first. Use an idle timeout plus an absolute cap (`scope.rs`, `buzz-acp/README.md`).
4. **Use bounded Stop objections for todos.** A Claude Stop hook asks narwarpd whether criteria are unchecked. A timeout counts as no objection, and after N objections the turn may end (`B/docs/MCP_DRIVEN_HOOKS.md`).
5. **Gate results with deterministic retry checks.** Like goose's `retry.checks`, the daemon runs shell checks (pytest, ruff) before accepting a `result`. Combine this with a per-ticket `available_tools` allowlist.
6. **Adopt a one-shot task contract.** Use a strict versioned JSON input and one JSON result record per run, with stable error codes and no automatic retry. Exit 0 is "not proof of task success" (`TASKS.md`).
7. **Reserve environment keys and pass MCP config at session start.** Write nothing to the agent's working directory. For Claude this means a daemon-generated, per-ticket MCP config (inference: check the `claude` CLI's flags).
8. **Treat presence as a lease.** A 180 s TTL keeps session status in the Warp board honest (`B/ARCHITECTURE.md`, Redis keys).

## What to avoid

- **Automatic approval:** `buzz-acp` defaults to `bypassPermissions` and automatically answers permission requests with `allow_once` (`B/crates/buzz-acp/src/config.rs#L473`, `acp.rs#L2014`).
- **Keys and API keys in agent environments:** the agent's private key reaches its MCP server and CLI, and LLM keys come from environment variables (`lib.rs#L6476`, `buzz-agent/README.md`).
- **Isolation by prompt:** Buzz enforces worktrees only through a prompt instruction. Narwarp should keep creating them in the daemon.
- **Gates judged by a model:** goose's Smart Approval falls back to an LLM that classifies calls as read-only (`G/crates/goose/src/permission/permission_inspector.rs#L182`).
- **A harness with no state:** Narwarp should keep the ticket-to-session mapping in SQLite.
- **A heavy stack:** Postgres, Redis, MinIO and a relay are overkill for one owner on a Mac (inference).
- **Claude through an ACP adapter:** Buzz and goose drive Claude through `claude-agent-acp`, which conflicts with Narwarp's rule 9 (only the official `claude` binary). That is an owner decision.

## Gaps

- `B/docs/agent-broker.md` is referenced by the broker contract but missing from the repo. I found no broker host implementation.
- `BUZZ_ACP_PERMISSION_POLICY` appears only in a doc comment (`B/crates/buzz-agent/src/permission.rs#L4`).
- Block's announcement page (https://block.xyz/inside/introducing-buzz-where-humans-and-agents-work-together) failed to fetch. Press reports give the launch version as 0.4.21 or 0.4.22.
- I did not deduplicate the contributor count, and GitHub does not show one.
- I ran nothing; behavior comes from docs and source. My goose clone omits `ui/desktop`.
- I did not check the internal Block build (squareup/buzz-releases) or the `claude` CLI flags.
