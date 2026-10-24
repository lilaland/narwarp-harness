# Warp upstream since 2026-05-07: what matters for the Narwarp terminal

Checked 2026-10-23 against `warpdotdev/warp` master `1e6c3ae` (2026-10-22). Method: commits-only, blob-less, shallow and full clones under the scratch `clones/` directory; `git log/show/diff`; `git merge-tree` simulations of cherry-picks onto P4 (`b9b9e760`) and of P4's 33 commits onto Warp master. Nothing was built or run, and conflict counts are textual only. "Inference" marks my judgement; everything else cites a commit, path or URL.

## Takeaway

- **Scale.** Master has 2,586 commits, 2,298 of them after `4dbf875` (#10004, 2026-05-07), the last Warp commit Zap took. Zap's real fork point is `c325d146` (2026-04-28). It took only about 93 of the 256 Warp commits between then and May 7, matched by PR number. Its sync log records "15 merged, 3 blacklisted, 133 deferred" (Zap `docs/openwarp-upstream-sync.md`, `8bdeba9`). Commits per month after the cut: May 560, Jun 470, Jul 609, Aug 347, Sep 181, Oct 1–22 131 (64 of them since Oct 13). Bot-authored commits rose from 10 of 560 in May to 76 of 131 in October.
- **Most of the new code is cloud and agent code.** Rust under `app/src/ai` grew from 6.6 to 12.9 MB since the fork point, and `crates/warp_tui` (3.5 MB) is new. Warp's strategy post is about cloud "software factories" ([blog, 2026-09-28](https://www.warp.dev/blog/adapting-for-a-world-of-software-factories)).
- **What helps the harness is small and local.** That means deep links that open a tab config or focus a pane, `WARP_FOCUS_URL` in every pane, and new CLI-agent events (needs-input, stop-failure, pending background work). It also includes two features P4 already contains but has switched off, and `warpctrl` as a model for local IPC.
- **ACP shipped only inside `warp agent run`.** That runner requires login and every run is tracked by Warp's server. ACP was enabled in release builds on 2026-10-21 (`349cdd6`). No GUI ACP client exists, and [#7326](https://github.com/warpdotdev/warp/issues/7326) is still open. Claude runs through the `claude-code-acp` adapter, which arguably breaks harness rule 9 (inference).
- **Official BYOK does not make Zap's BYOP redundant.** Your keys and endpoints are sent with each request, and Warp's server makes the model call. Anonymous users are excluded.
- **Re-fork.** P4's own layer replays well: 21 of 33 commits apply cleanly. The cost is redoing Zap's work: account de-gating, hiding the cloud features, the rebrand and, above all, BYOP. My estimate is 1.5–2.5 weeks without the Ollama agent and 3–8 weeks with it (inference).
- **Recommendation (inference).** Stay on P4 for Phase 1, apply the small cherry-pick bundle, and build the board as new modules. Revisit a re-fork when the board works or when Warp ships a GUI ACP client.

## Relevant features

| Feature | Evidence | Paths and size | Cloud dependency | Port difficulty |
|---|---|---|---|---|
| Deep links and the pane focus variable | #9655 `27f4933b` (`warp://session/<uuid>` focuses a pane; it predates the cut but Zap never took it). #9379 `207f9d5` (`warp://tab_config/<name>`). #10698 `a017b9a` (tab-config commands run sequentially). #13103 `482f63a` (launch-config tab commands). #10233 `e1be417` (`action/open_file_editor`). #11130 `679e426d` (`WARP_TERMINAL_SESSION_UUID` and `WARP_FOCUS_URL` set in every pane) | `app/src/uri/mod.rs`, `app/src/terminal/focus_env.rs`; 6 commits, +964 lines | None | **Easy.** 14 conflict hunks in 7 files when applied in sequence. P4 already has the pane `session_uuid()` and `hex` they need |
| CLI-agent status over OSC 777 (`warp://cli-agent`) | The v1 parser is already in P4. New upstream: StopFailure #13784 `a5242e6`, NeedsInput #16232 `9ccb4e3`, pending background work #16367 `42e2a7d`, protocol types moved to `crates/warp_core/src/cli_agent_protocol.rs`, plus more agents and the Codex plugin (#11871) | `app/src/terminal/cli_agent_sessions/` grew from 92 KB at the cut to 196 KB; 44 commits, +3,542/−464 | None for local sessions | **Easy by hand.** Cherry-picking the 21 relevant commits hits 110 hunks; hand-porting the three event types is about 200 lines (inference). P4's `RawEvent` ignores unknown fields and maps unknown events to `Unknown` (`event/v1.rs`), so a `ticket` field can ride along |
| `warpdotdev/claude-code-warp` plugin | MIT licensed. `f8c6287` (2026-10-05) forwards `agent_needs_input`; `53ab8d6` (2026-10-08) reports pending background work. Warp now requires version 2.4.0 or later (`plugin_manager/claude.rs:24`); P4 requires 2.0.0 | `plugins/warp/hooks/hooks.json`, `emit-terminal-sequence.sh` | None | **Use as is** (external repo) |
| `warpctrl` local control | `specs/warp-control-cli/`; #11772 `5967abf0`, #12327 `aa0a2c21`, #12705 `1d65e362`, #14158 `d4c4cf9b`, #14832 `7469abe3` | `crates/local_control` 92 KB, `app/src/local_control` 185 KB, `crates/warp_cli/src/local_control` 86 KB, tests included. 84 actions over a loopback HTTP endpoint, with a Unix-socket credential broker | None. The `WarpControlCli` flag is on only in dogfood builds (`DOGFOOD_FLAGS`), and Settings > Scripting is off by default in OSS builds. It cannot run commands: there is no `input.run`, and `tab.create` takes only `tab_type` (`protocol.rs`) | **Medium.** 7 commits, 57 hunks. Most of it predates the switch to the Rust 2024 edition |
| ACP harness transport | #16329 `08a845f`, #16330 `81e5532`, #16331 `f386a4c` (the external-harness controller API), #16332 `02668f9`, #16393 `37fcf68` (runs persisted on the server), #16599 `349cdd6` (release) | `app/src/ai/agent_sdk/driver/harness/acp/`: 24 files, 373 KB, about 215 KB of it non-test; `blocklist/controller/external_harness.rs` | **High.** Reachable only through `warp agent run`, which requires login (`command_requires_auth` in `agent_sdk/mod.rs`). The server chooses the transport (`WARP_HARNESS_TRANSPORT`). Claude runs through `claude-code-acp` (`acp/manifest.rs:100`) | **Re-fork only.** 20 commits; 17 conflict, with 136 hunks, and they build on agent-SDK code Zap trimmed (311 KB in Zap vs 3.0 MB upstream). When Zap tried harness commits it got "85+ compile errors" (sync doc) |
| Third-party harness runs and orchestration | `AgentHarness` reached production in #11031. The local Claude/Codex child flag is in `LOCAL_FLAGS` only. About 114 commits mention orchestration or child agents; the `warp_multi_agent_client` crate came in #12887 | `agent_sdk` grew from 1.0 to 3.0 MB; `pane_group/pane/local_harness_launch.rs` needs an `AmbientAgentTaskId` | **High** (server tasks, Oz) | No; study only |
| Agent Management view and inbox | `app/src/ai/agent_management/` (272 KB); the view fetches tasks from the server (`view.rs`). Zap deleted it (`002ce46`) and rebuilt a local inbox (`c757f27`, `app/src/notifications/`, present in P4) | n/a | View: server. Inbox: local | **Skip.** P4 already has the local inbox |
| Tab groups, pinned tabs, vertical-tab summary | #11486 `fc11033` through #14690 `9fb32ed` (34 commits); pinned tabs stable in #13767; tab groups saved in launch configs in #14624; summary mode in #10793 | `tab_grouping.rs` 31 KB; vertical tabs grew from 262 to 331 KB | None | Groups and pins: **hard** (26 of 34 commits conflict, 112 hunks), so they come with a re-fork. Summary mode: **flag flip**, since P4 wires `vertical_tabs_summary_mode` but leaves it out of `default` |
| Code review | Git operations reached stable in #11716 (P4 wires `git_operations_in_code_review` but leaves it off); remote review #11537; send-to-agent shortcut #12982 | `app/src/code_review` grew from 793 to 985 KB | None (PRs are created with `gh`) | Git operations: **flag flip**. The rest: medium (#12230 alone has 41 hunks) |
| MCP | Upstream `rmcp` adopted in #10754; runtime moved into `crates/mcp` (#11378); OAuth fixes #9460, #10984, #16314; log rotation #10874 | `crates/mcp` 167 KB. File-based configs: `.warp/.mcp.json`, `.mcp.json`, `.agents/.mcp.json` (`app/src/ai/mcp/mod.rs:549`) | Local, except managed and Factory MCP (#12754, #14657) | Pick fixes one by one. File-based MCP is already in P4 |
| BYOK, custom endpoints, model routers, ChatGPT sign-in | Solo BYOK reached stable in #11437; anonymous users are excluded (#10609); routers in #12775; ChatGPT plans in #16216 | `specs/byok-byoe-enterprise/PRODUCT.md`: the client attaches the key and "the server uses it transiently" | **Total** | Not useful for local-first use |
| `warp_tui` (Warp Agent CLI) | #13006 `fc773902`, 312 commits; login with `--api-key` (#13583) or device auth (#14670) | 190 files, 3.4 MB; depends on the whole `warp` crate (`crates/warp_tui/Cargo.toml`) | Server-side agent | No |
| Terminal lifecycle recovery | #12852–#12859; production in #13788 `19ebec9d` | Terminal core and shell bootstrap | None | **Medium** (46 hunks); optional, for stability |

There is no plugin or extension API for custom panels; no commit subject mentions one, so the board has to be compiled into the fork. Warp also has no support for `claude --bg`, `claude attach` or `claude agents`. Detection matches on the command's first token, so `claude attach` should register as Claude Code (inference).

## Account and cloud gating

- **OSS build.** `app/src/bin/oss.rs` talks to the production server and sets no telemetry, crash-reporting or autoupdate config (all `None`). Its URL scheme is `warposs`. Nothing has changed since the cut except formatting.
- **AI requires login.** `AISettings::is_any_ai_enabled` returns false for anonymous or logged-out users (`app/src/settings/ai.rs:2277`), and has since the first public commit `0dbd3d56`. Zap hard-codes `true` (P4 `app/src/settings/ai.rs:1933`). The function has about 200 call sites, including automatic opening of rich input (`terminal/view.rs`) and conversation history. The CLI-agent footer is deliberately independent of it (`use_agent_footer/mod.rs`).
- **Other gates.** There are about 60 `is_anonymous_or_logged_out()` calls in 32 files and 34 `is_logged_in()` calls.
- **Onboarding.** "Warp's AI features require a Warp account" (#12992 `dbde9bc`; changelog 2026.07.03). Account-first onboarding (#14075, stable in #14223) still offers "Skip for now". The `SkipFirebaseAnonymousUser` flag leaves skipped users "fully logged out" with login-gated features disabled (flag doc in `crates/warp_features/src/lib.rs`).
- **Depends on Oz or the server.** ACP and harness runs, orchestration, Agent Management, managed and Factory MCP, `/remote-control` ("Log in to use /remote-control") and BYOK.

**Public statements**
- README: "bring your own CLI agent (Claude Code, Codex, Gemini CLI, and others)", "OpenAI is the founding sponsor…", and Factories sections (`f349ffe8`, `8c6a00c8`).
- `FAQ.md`, unchanged on these points since release:
  - ACP: "Warp's built-in agent harness runs server-side… we plan to support ACP".
  - Local use: "Some functionality works fully locally; other features (Drive sync, hosted-model agents, team features) require Warp's backend".
  - The server, Drive backend and Oz stay proprietary.
  - Forks: "Can someone fork Warp? Yes — … open derivatives are welcome."
- [Roadmap #9233](https://github.com/warpdotdev/warp/issues/9233) (closed 2026-09-01) committed to ACP and said it "will unlock local model support through harnesses that support it (like OpenCode)".
- [#7326](https://github.com/warpdotdev/warp/issues/7326) is still open. The external ACP-client [PR #9380](https://github.com/warpdotdev/warp/pull/9380) contains only a spec and has stalled.
- [Changelog](https://docs.warp.dev/changelog/2026/):
  - tab_config deep link and sequential tab-config commands (2026.05.18)
  - solo BYOK (2026.05.20)
  - `warpctrl` wrappers (2026.06.24)
  - AI tied to an account (2026.07.03)
  - unattended CLI agents "end as blocked" (2026.10.07)
- I found no Warp statement about OpenWarp, Zap or other third-party forks.

## Re-fork estimate

**Structure check**
- Every upstream file that P4 modifies still exists at the same path.
- P4's anchors are all still there:
  - `ToolPanelView` (`workspace/view/left_panel.rs:149`)
  - `LeftPanelDisplayedTab` (`app_state.rs:312`)
  - `SettingsSection` (`settings_view/mod.rs:312`)
  - the slash-command registry
  - the diesel-based `crates/persistence`
- Commits touching each hub file since the fork point:

| File | Commits | Note |
|---|---|---|
| `workspace/view.rs` | 242 | +18,808/−11,841; now 1.26 MB |
| `agent_sdk/driver.rs` | 134 | |
| `app/src/lib.rs` | 133 | |
| `slash_commands/mod.rs` | 66 | |
| `settings_view/mod.rs` | 45 | |
| `left_panel.rs` | 12 | |
| `persistence/src/schema.rs` | 7 | |

**Replay simulation.** Replaying P4's 33 commits onto master gives 21 clean commits and 12 that conflict, with 174 hunks in all. 127 of those hunks are in `62ae71c7`, the Phase 0 rebrand, 89 of them in `view.rs`. Only about 21 of P4's added `view.rs` lines are Narwarp features (the Ollama and Vault panel wiring). The rest is OpenWarp/Zap version skew (see `6e14311e`), so that commit should be redone rather than replayed (inference).

**API drift to fix**
- 229 `t!()` i18n calls. Warp has no i18n.
- `database_file_path()` became `database_file_path_for_scope` and `database_file_path_for_current_scope` (`persistence/sqlite.rs:453–469`).
- `genai` is not in Warp's workspace. A direct call to Ollama's `/api/embed` can replace it.
- Warp has 141 migrations, the latest dated 2026-08-26.
- Warp moved to the Rust 2024 edition (`abea51cd`, #13990, 1,893 files, 2026-07-20), so ported code needs `cargo fix --edition`.
- No change needed for sqlite-vec: `libsqlite3-sys 0.33.0` and toolchain 1.92.0 match.

**Pieces to redo**

| Piece | In Zap today | On a re-fork |
|---|---|---|
| Account and AI de-gating | `is_any_ai_enabled` returns true. Zap made 91 cloud-removal commits, deleting about 153k lines (summed across commits) | Patch the gates and drop cloud features from `default` instead of deleting code: 2–4 days (inference) |
| Rebrand | `a00ae6cf` relied on `app/i18n` (1 MB of `.ftl`) | Identity only (app id, URL scheme, directories, icons, binary): 1 day. Renaming every string means about 758 literals in 211 app files plus 114 in crates, and more merge conflicts later |
| BYOP and Ollama agent | `agent_providers`: 83 files, 803 KB. `byop_compaction` 59 KB and `byop_readiness` 60 KB. Vendored `lib/rust-genai` 1 MB. Hooks in 23 files, including `blocklist/controller.rs` | (a) Re-port onto the refactored controller: 3–6 weeks. (b) Write a new local driver on the `ExternalHarness` API (#16331): 1–3 weeks. (c) Defer it, since conductor Tusk already runs as `claude --bg -n tusk` (addendum §1) |
| SSH manager | `crates/warp_ssh_manager` 139 KB plus `zap_sftp` | Optional |
| P4 layer | 61 new files (about 7.5k lines) plus about 700 lines of edits to upstream files (slash commands 319, left panel 107, schema and sqlite 167) | 3–5 days. The vault and RAG can wait for Phase 2 |

**Totals (inference)**
- Minimal re-fork (de-gate, identity rebrand, P4 essentials, BYOP deferred): 1.5–2.5 weeks.
- With option (b): 3–5 weeks.
- With option (a): 5–8 weeks or more.
- After that, a monthly upstream merge of about 1–2 days, provided Narwarp's edits to hub files stay a few lines each.

## Options

**A. Stay on Zap/P4 and cherry-pick (2–4 days)**

What to apply:
1. The deep-link bundle: `27f4933b`, `207f9d5`, `a017b9a`, `679e426d`, `e1be417`, `482f63a`.
2. Two flag flips: `vertical_tabs_summary_mode` and `git_operations_in_code_review`.
3. Hand-port handling for NeedsInput, StopFailure and pending background work (#13784, #16232, #16367).
4. Optional: `warpctrl`, the MCP OAuth and log fixes, and lifecycle recovery.

Pros:
- Cheapest.
- Keeps Tusk on Ollama, the cloud-free build and P4 unchanged.
- The bundle covers §7.2's session panes. Hooks report `$WARP_FOCUS_URL` to `narwarpd`, the board and `nw open` jump back to a pane, and the board opens generated tab configs.

Cons:
- The base stays about 2,460 commits behind, on the Rust 2021 edition.
- Every later upstream fix, security patches included, has to be ported by hand.
- No tab groups, no ACP.

**B. Re-fork now (1.5–2.5 weeks before board work)**

Pros:
- Current CLI-agent support, tab groups and pinned tabs, `warpctrl` and lifecycle fixes.
- Cheaper syncs later.

Cons:
- Tusk on Ollama is gone until BYOP is rebuilt.
- About 20 MB of extra Rust, much of it cloud code.
- Login gates have to be re-patched as Warp adds new ones.
- Upstream growth now goes mostly into Factories code that Narwarp would hide.

**C. Hybrid (recommended, inference)**

1. Do A now.
2. Build `narwarp_client`, the board panel and the session panes as new modules, with each hub registration listed in `NARWARP_PATCHES.md`.
3. Re-fork when the board is stable, or once Warp ships a GUI ACP client. Such a client could replace BYOP with an ACP agent that supports Ollama, such as OpenCode, which is the route Warp's roadmap describes.

The board relies only on stable anchors (left panel, pane groups, URI handler), so it would move almost unchanged.

For a Python-first solo developer who has Claude Code write the Rust, A and C keep every task session-sized and verifiable. B is a multi-week Rust migration that can only be checked on the Mac, where the addendum already says terminal work must happen. `warpctrl` is optional in every option. The addendum has the terminal subscribe to daemon events and open panes itself, and deep links already cover `nw open`.

## Gaps

- Nothing was built or run. The simulations count textual conflicts only. Compile breakage (renamed APIs, let-chains) was not measured, and the `-X theirs` carry-forward can misstate later counts.
- I did not verify what network calls a logged-out re-forked OSS build makes. That needs a proxy capture on the Mac.
- The proxy blocked the GitHub API for `warpdotdev`, so PR and issue details come from commit messages and WebFetch. The blog index showed only September and October posts.
- I did not review individually the roughly 163 commits from Apr 28 to May 7 that Zap skipped, except #9655.
- Untested on the Mac:
  - whether panes running `claude attach` receive OSC 777 through `terminalSequence` (NW-6)
  - whether detection tags them as `CLIAgent::Claude`
- Whether ACP adapters such as `claude-code-acp` satisfy rule 9 is the owner's decision.
- The repo has no trademark guidance. The FAQ addresses forks only under AGPL.
