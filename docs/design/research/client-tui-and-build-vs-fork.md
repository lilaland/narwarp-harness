# Narwarp client: build-vs-fork decision and terminal UI (TUI) framework choice (state as of 2026-10-13)

Scope: whether to keep building the harness inside the Narwarp fork (Rust, WarpUI), start a standalone harness, or do a hybrid, and which TUI framework fits. Local facts come from `/home/user/narwarp` (git commands or commit hashes are given in place of URLs). Remote git facts come from read-only `git ls-remote` and commits-only clones (`git clone --bare --filter=tree:0`) made in the session scratchpad and deleted afterwards. The GitHub REST API was not reachable from this session (HTTP 403 from the proxy), so star and issue counts come from the GitHub web pages. Notation: "LOC" = lines of code; "MB" = merge base `bbcdbc12`; "P4" = `origin/phase-4/watcher-and-polish`.

Corrections to the facts I was given:
- The repo is **not** shallow: `git rev-parse --is-shallow-repository` returns `false`.
- AGENTS.md says the workspace has 67 member crates (`git show upstream/main:AGENTS.md`). I count 64 `Cargo.toml` files at upstream/main. The figure I was given was 61. The numbers depend on how crates are counted. Either way it is about 2,959 `.rs` files.
- The Narwarp layer is **not** merged into `main`. `origin/main` and `main` are both `bbcdbc12`, and only P4 contains the layer (`git branch -a --contains b9b9e760`).
- The older `origin/phase-0/1/2` branches are **not** ancestors of P4 (`git merge-base --is-ancestor` → no). They are stale copies from before the June rebase.
- "unwarp" is the original name of this same fork: `62ae71c7` "chore(rebrand): unwarp Phase 0", later renamed to Narwarp in `55ffe36f` (2026-06-05). So "pull latest from unwarp" can only mean merging upstream Zap into P4 (or merging P4 into main).

---

## 1. Upstream health: Warp, Zap, and other forks

### Takeaway
Warp's official open-source repo is very active (596 commits from 2026-08-01 to 2026-10-13). It is also moving into the same space Narwarp targets: an Agent Client Protocol (ACP) harness runner, multi-agent orchestration, and a terminal-agnostic "Warp Agent CLI" TUI. Zap, Narwarp's direct upstream, stopped taking Warp changes on 2026-05-07 and has been nearly dormant since July 2026: 0 commits in August, 4 in September, 0 in October, and no tag since v2026.07.09.1. Nothing shows that the standalone harness in Zap's roadmap has started.

### Cited Findings
**Warp (warpdotdev/warp)**
- Warp announced its open-source client on 2026-04-28: "Warp's source code is now available at github.com/warpdotdev/warp with an AGPL license." OpenAI is "the founding sponsor." The contribution model is agent-first: agents do the coding and the community contributes "ideas, direction and verification". Warp's own Oz agent is preferred, and the product is described as "multi-model and multi-harness". — [Warp blog](https://www.warp.dev/blog/warp-is-now-open-source)
- The first public commit is `0dbd3d5` "Initial public release of Warp." (David Stern, 2026-04-28). The default branch is `master`, with 2,525 commits as of 2026-10-13 and tip `4f3360d` dated 2026-10-13. — commits-only clone of https://github.com/warpdotdev/warp, `git log --reverse`, `git rev-list --count master`
- License split: "Warp's UI framework (the `warpui_core` and `warpui` crates) are licensed under the MIT license. The rest of the code in this repository is licensed under the AGPL v3." — [Warp README](https://github.com/warpdotdev/warp). Zap has the same split: `crates/warpui/Cargo.toml` and `crates/warpui_core/Cargo.toml` declare `license = "MIT"`, the workspace declares `license = "AGPL-3.0-only"`, and the repo root has `LICENSE-AGPL` and `LICENSE-MIT` (`git show upstream/main:Cargo.toml`).
- Search-result summaries of third-party aggregator pages disagree on the open-sourcing date (Apr 22, Apr 28, May 7) and on the license (one summary even said Apache-2.0). I did not open each page, so I cannot say which page claims what. The primary sources (Warp's blog and git history) agree on 2026-04-28 and AGPL/MIT. — [It's FOSS](https://itsfoss.com/news/warp-goes-open-source/), [ai-tldr](https://ai-tldr.dev/tools/zap/), [Kilo blog](https://blog.kilo.ai/p/warp-finally-went-open-source)
- Warp master commits per month in 2026: Apr 131 (from the 28th), May 736, Jun 450, Jul 612, **Aug 345, Sep 181, Oct 1–13: 70**. From 2026-08-01 to 2026-10-13 there were 596 commits by 57 distinct author names. — `git log --since=2026-01-01 --format=%ad --date=format:%Y-%m master | sort | uniq -c` on the commits-only clone
- Warp's agent direction in October 2026, from commit subjects:
  - `02668f9` (2026-10-09) "Agent driver: ACP client, MAA mapping, and the ACP harness runner (#16332)"
  - `f386a4c` "Agent Mode: let an external harness drive a native conversation turn (#16331)"
  - `81e5532` "run an ACP harness inside the terminal session over a socket bridge (#16330)"
  - `7c31488` (2026-10-11) "Add first-class Muse Code CLI agent support (#15999)"
  - "Enable CloudAgentExecutionConfig on stable (#16429)" (2026-10-12)
  - The README says: "Use Warp's built-in coding agent, or bring your own CLI agent (Claude Code, Codex, Gemini CLI, and others)."
  - Sources: Warp commits-only clone; [Warp README](https://github.com/warpdotdev/warp)
- Warp built its own TUI, on its MIT-licensed UI core, after Zap stopped syncing:
  - `1ef980b` (2026-06-19) "Add TUI API surface to AppContext under `tui`"
  - `ce1a91c` (2026-06-22) "warpui_core: TUI element library"
  - `608bc53` (2026-06-24) "warpui_core(tui): build TUI element primitives on ratatui-core"
  - `fc77390` (2026-06-24) "Initial TUI crate"
  - `683d407` (2026-08-10) "Multi-level orchestration UI in the Warp TUI"
  - 262 commits mention TUI in their subject (Jun 10, Jul 223, Aug 29). The crate's `Cargo.toml` says `name = "warp_tui"`, `description = "Warp Agent CLI"` — [warp_tui Cargo.toml](https://raw.githubusercontent.com/warpdotdev/warp/master/crates/warp_tui/Cargo.toml).
  - Zap's `crates/warpui_core` has no TUI files (`git ls-tree -r upstream/main -- crates/warpui_core | grep -i tui` → none).
- Warp Agent CLI docs (last updated 2026-10-10):
  - "Because it's a regular command-line program, it works in any terminal emulator, including over SSH."
  - It "creates and maintains its own pseudoterminal (PTY)", which supports "prompts and full-screen terminal apps".
  - Conversations "save to your Warp account"; you can "bring your own provider API keys".
  - It offers "Cloud handoff and orchestration" to "coordinate multiple agents".
  - Source: [Warp docs](https://docs.warp.dev/agents/cli/)

**Zap (zerx-lab/zap)**
- Zap's root commit is Warp's `0dbd3d56` "Initial public release of Warp." (2026-04-28), and Zap main has 814 commits (`git log --reverse upstream/main`). Only **32** commits are shared with Warp master, and the newest shared one is `c325d146` (2026-04-28) — found by intersecting `git rev-list` outputs of both repos.
- After the fork, Zap cherry-picked 93 Warp PRs (subjects carrying Warp-range PR numbers #9xxx–#1xxxx): 56 in late April and 37 in May. **The last was `572e1fde` on 2026-05-07** (Warp PR #10004). Warp master has **2,193 commits after 2026-05-07** that Zap lacks. — `git log --since=2026-04-29 upstream/main | grep -E '\(#(9[0-9]{3}|1[0-9]{4})\)'`; `git rev-list --count --since=2026-05-08 master` (Warp clone)
- Zap commits per month: Apr 125, May 607, Jun 68, Jul 10, **Aug 0, Sep 4, Oct 0** (to 10-13) (`git log upstream/main`). The live `refs/heads/main` is `8954864e`, identical to the local `upstream/main` (`git ls-remote https://github.com/zerx-lab/zap`, 2026-10-13). There are 12 tags, from `v2026.05.29.1` to **`v2026.07.09.1`**, and none since (`git ls-remote --tags`).
- Zap's README says the project is "Currently based on Warp; evolving independently going forward." Zap removed its upstream ties early: `4e3853fe` (2026-05-14) "Remove OpenWarp upstream release update paths" and `3face78b` (2026-05-13) "Remove GraphQL cloud object sync stack". — [zerx-lab/zap](https://github.com/zerx-lab/zap); `git log upstream/main`
- GitHub page as fetched on 2026-10-13: **2.1k stars, 179 forks, 3 open PRs**. The page did not show a contributor count or an issue count. — [zerx-lab/zap](https://github.com/zerx-lab/zap)
- Since Narwarp's base (MB, 2026-06-04) Zap has 60 commits: 46 in June, 10 in July, 4 in September. They come from 19 author names (zerx-lab 21, zero 11, everyone else ≤4). By type: 37 `fix`, 12 `feat`. The main scopes are windows, ai, image-viewer, i18n and ssh. The most relevant commits:
  - `ea071c14` Russian UI (#340)
  - `e8bbe331` per-window themes (#279)
  - `b1a2fefe` CLI-agent settings UI (#210)
  - `4173c351` and `2c125079` BYOP provider-routing and tool-execution fixes (#305, #306)
  - `3250483b` omp CLI agent
  - `0aac7080` (2026-09-28) remember the CLI-agent floating-window size
  - Source: `git log bbcdbc12..upstream/main`
- Zap's roadmap, Phase 1: "Design and build a standalone open-source agent service from scratch — agent loop, tool runtime, … not tied to Warp's existing client code. Zap becomes its first consumer." It also says "The terminal becomes one client among many — a TUI, an IDE plugin or a cloud worker can all drive the same engine" and "Roadmap items are exploratory". — [docs/roadmap.md](https://github.com/zerx-lab/zap/blob/main/docs/roadmap.md)
- None of Zap's 33 remote branches looks like harness work. They are `claude/*` session branches, `fix/issue-*` branches, `feat/issue-triage-workflow`, `feat/ssh-remote-file-open` and `issue/116-rules-agent-context` (`git ls-remote --heads`).
- One directory site says OpenWarp/Zap "continuously merges upstream code" — [EveryDev](https://www.everydev.ai/tools/openwarp). **Git history contradicts this**: Zap has taken no Warp commits since 2026-05-07.
- Zap inherits Warp's CLI-agent integration. `app/src/terminal/cli_agent.rs` lists "Claude Code, Gemini CLI, Codex, Amp, Droid, OpenCode, Copilot, Pi, Auggie, Cursor, Goose", and there is a Claude plugin manager at `app/src/terminal/cli_agent_sessions/plugin_manager/claude.rs` (`git show upstream/main:…`).

### Inferences
- Pulling the latest Zap into Narwarp buys 60 commits of mostly Windows, i18n, SSH and image-viewer fixes. That has little strategic value for a macOS-only personal harness. The useful items are a few BYOP (bring-your-own-provider) fixes and the CLI-agent UI tweaks.
- Narwarp is now a fork of a hard fork that is going quiet. If the user keeps building in it, they effectively become the sole maintainer of a 1.14M-line Rust app, covering toolchain bumps, macOS SDK changes, dependency security fixes and Warp-era bugs.
- Switching Narwarp's base from Zap to Warp master is not realistic. Zap carries about 782 commits of its own (rebrand, BYOP/Ollama, SSH manager, removal of the cloud stack), and Narwarp's Ollama/BYOP features depend on them. Warp's own drift since the fork is 2,193 commits.
- Warp is building many harness-adjacent features in the open (ACP harness runner, orchestration TUI, cloud agents), backed by funding and a large team. Competing with that inside the same codebase is a poor bet for a solo developer. Staying terminal-agnostic lets the user run the harness alongside whichever terminal is best (official Warp, Zap/Narwarp, Ghostty, iTerm2).

### Gaps
- Zap's open-issue count and any maintainer statement about status: the GitHub API was blocked and the web page did not show issues. I found no announcement explaining the drop in activity after July.
- How other Warp forks have fared: searches turned up only OpenWarp/Zap. I found no reliable sources on other forks.
- Whether the Warp Agent CLI can run with no Warp account (BYOK or local models only): the docs are ambiguous.

---

## 2. Merge exposure: which upstream files the Narwarp layer touches, and how a merge with upstream/main goes

### Takeaway
A trial merge of `upstream/main` into P4 (2026-10-13) conflicts in **4 files: 9 hunks, about 467 lines inside conflict markers**. Six hunks (328 lines) are in `app/src/workspace/view.rs`, a 22,334-line hub file. Another 16 files changed on both sides auto-merged cleanly. Resolving the text would take hours, but the previous sync (232 Zap commits) needed a separate commit to fix **88 compile errors**. The real cost is compile and semantic breakage, which only a build reveals, and I did not run one.

### Cited Findings
- MB(P4, upstream/main) = `bbcdbc12` (2026-06-04). P4 has 33 commits touching 111 files: 61 added and 50 modified, +10,046/−1,836. — `git merge-base`, `git diff --name-status bbcdbc12 P4`
- Added code: `app/src/vault/` (14 files, 3,774 lines), `app/src/rag/` (11 files, 2,354 lines), `app/src/ollama_panel/` (3 files, 515 lines), `settings/unwarp.rs` (177), `settings_view/vault_page.rs` (209), `benches/rag.rs` (123), `tusk_persona.j2` (23), the migration `up.sql` (95), 11 narwhal sprites plus `script/slice_sprites.py` (97), and icons. — `git show P4:<file> | wc -l`
- Modified upstream files, with Narwarp's churn compared with upstream's churn since MB (`git diff --numstat`, `git rev-list --count bbcdbc12..upstream/main -- <file>`):

  | File | Narwarp +/− | Upstream commits | Upstream +/− |
  |---|---|---|---|
  | app/src/workspace/view.rs | +569/−618 | 8 | +323/−22 |
  | app/i18n/{en,ja,zh-CN}/warp.ftl (rebrand strings) | ~+215/−215 each | 6 | +131 each |
  | crates/persistence/src/schema.rs | +92 | 3 | +18 |
  | app/src/persistence/sqlite.rs | +75 | 3 | +24 |
  | app/src/app_state.rs | +4 | 3 | +13/−1 |
  | app/src/lib.rs | +13 | 2 | +11/−4 |
  | app/src/settings_view/mod.rs | +13 | 1 | +136/−8 |
  | app/src/workspace/view/left_panel.rs | +107/−2 | 1 | +13 |
  | app/src/autoupdate/linux.rs / mac.rs | +443/−235, +60/−235 | 1 each | +10/−10, +1/−1 |
  | app/src/bin/zap_oss.rs | +11/−12 | 1 | +12 |
  | Cargo.toml, app/Cargo.toml, app/build.rs, theme color.rs/mod.rs, ssh secrets.rs, autoupdate/mod.rs | small | 1 each | small |
  | 30 other modified files (slash commands +319, prompts *.j2, paths.rs, themes, icons) | — | 0 | 0 |

- 20 files changed on both sides. Upstream commits touching those 20 files: 12 in June, 1 in July, 2 in September. — `comm -12` of both name lists; `git log --format=%ad -- <files>`
- Upstream commits to `view.rs` since MB:
  - `eaa2f96a` (#228)
  - `c1c00f52` image viewer (#229)
  - `83793b45` onkey (#231)
  - `b1a2fefe` CLI-agent settings (#210)
  - `9f7d9c7c` (#256)
  - `e8bbe331` per-window themes (#279)
  - `4c928796` i18n (#339)
  - `0aac7080` (#322, 2026-09-28)
  - Source: `git log bbcdbc12..upstream/main -- app/src/workspace/view.rs`
- **Trial merge**, 2026-10-13, in the throwaway detached worktree at `…/scratchpad/merge-trial`, using `git merge --no-commit --no-ff upstream/main`; afterwards `merge --abort`, `worktree remove --force`, and `worktree prune` were run, and `git worktree list` shows only the main checkout:
  - Conflicts:

    | File | Hunks | Lines in conflict regions |
    |---|---|---|
    | `app/src/workspace/view.rs` | 6 | 328 |
    | `app/src/autoupdate/mac.rs` | 1 | 66 |
    | `app/src/autoupdate/linux.rs` | 1 | 56 |
    | `app/src/bin/zap_oss.rs` | 1 | 17 |

  - Clean auto-merges (16 files): Cargo.toml, app/Cargo.toml, build.rs, the three .ftl files, app_state.rs, autoupdate/mod.rs, lib.rs, persistence/sqlite.rs, settings_view/mod.rs, left_panel.rs, persistence/schema.rs, theme/color.rs, theme/mod.rs, warp_ssh_manager/secrets.rs.
- What the `view.rs` conflicts contain: upstream added CLI-agent install imports (`CLIAgentInstallEvent`), a `LeftPanelEvent::OpenRemoteFile` match arm, an SSH `startup_command_injector`, a save-logs-via-dialog method, and changed separators in the AI menu. One hunk is the SSH keychain secret lookup, which Narwarp's side keeps and upstream restructured. The conflict hunks sit around lines 108 to 6,021 of the merged file.
- The merged `schema.rs` auto-merged with Narwarp's `vault_files`, `vault_note_chunks` and `unwarp_messages` entries next to upstream's new tables. Upstream added 4 Diesel migrations since MB: 2026-06-08 `ssh_onekey_credentials`, 2026-06-09 `ssh_onekey_key_type`, 2026-06-24 `window_theme_override`, 2026-07-20 `cli_subagent_window_sizes`. — `git diff --diff-filter=A bbcdbc12 upstream/main -- crates/persistence/migrations`
- The previous sync, commit `6e14311e` (2026-06-06) "Fix 88 upstream compile breaks from 232-commit sync":
  - It explains: "All errors were introduced by the upstream Zap rebrand that deleted modules, renamed enums, and changed function signatures."
  - Fixes included stubbing deleted modals in view.rs, renaming WarpDrive→ZapDrive variants, a changed `CodeManager` API, and an autoupdate signature change.
  - Result: "Finished dev profile with 144 warnings, 0 errors."
- Industry study: a Microsoft Research study of the Microsoft Edge fork (ICSE 2020) found that keeping divergent forks in sync causes frequent merge conflicts, which it classified as textual conflicts, build breaks and test failures, with build breaks being costly. — [Microsoft Research](https://microsoft.com/en-us/research/?p=640647) (2020; old, but the mechanism is general)
- Practitioner opinion: for highly diverged out-of-tree forks, merging upstream one commit at a time keeps the history bisectable, at the cost of redoing conflicts; `git rerere` helps. — [LLVM Discourse](https://discourse.llvm.org/t/how-to-keep-out-of-tree-target-in-sync-with-upstream/68027) (practitioner; possibly stale)
- Opinion: a fork's cost falls when it touches fewer upstream files and isolates its changes. — [R-bloggers, 2025-02](https://www.r-bloggers.com/2025/02/the-dynamic-relationship-of-forks-with-their-upstream-repository/) (more than 6 months old)

### Inferences
- **Cost of merging now**, a rough estimate:
  - 2–4 hours to resolve the text. Autoupdate: keep Narwarp's rewrite. `zap_oss.rs`: trivial. `view.rs`: union both sides' additions carefully.
  - Then an unknown number of compile breaks, probably far fewer than 88, because these 60 commits are fixes rather than a rebrand.
  - Then a full rebuild and test pass.
  - Total: roughly 0.5–1.5 days with Claude Code. The payoff is about 60 low-relevance fixes.
- Exposure is concentrated in the hub files `view.rs`, `left_panel.rs`, `settings_view/mod.rs` and `schema.rs`. Every new Narwarp view (kanban, calendar, boards) would be wired through these, so the future conflict surface grows with each feature even while Zap is quiet.
- Because Zap is nearly dormant, future merge cost is low, but so is the value of merging. The bigger ongoing burden is maintaining the whole application alone.

### Gaps
- Compile-level breakage after the merge was not measured, because the constraints ruled out a full build.
- Whether existing local Narwarp databases migrate cleanly with upstream's four new migrations is untested.

---

## 3. Effort to build new views in WarpUI; build times; fit for a Python-first developer

### Takeaway
WarpUI has the primitives a kanban board needs (Flex, Scrollable/UniformList, Table, Draggable/DropTarget). It has no calendar or date component and no standalone documentation (only example apps), and every view has to be wired through the AGPL app crate (`view.rs`, `left_panel.rs`, settings, Diesel). The user's own panels show the scale: a tree browser with search and job buttons took 910 lines of Rust, and an Ollama status list took 515. Claude Code wrote most of the existing layer, so writing the code is not the bottleneck. Iteration speed, merge drag and being tied to one terminal are.

### Cited Findings
- `app/src/vault/explorer/panel.rs` is 910 lines: action and event enums, a `VaultPanel` struct, about 20 methods, `TypedActionView::handle_action`, `View::render`, and the render functions `render_uninitialized`, `render_message`, `render_tree`, `render_search_bar`, `render_jobs_header` (~140 lines) and `render_row`. Element use: 14 `Container`, 14 `Text`, 12 `MouseStateHandle` (one per clickable element), 7 `Hoverable`, 5 `Flex`. — `git show P4:app/src/vault/explorer/panel.rs | grep`
- `app/src/ollama_panel/`: `client.rs` 203 + `panel.rs` 301 + `mod.rs` 11 lines. `panel.rs` covers polling, a section heading, running/pulled model rows and an empty state. — `git show P4:app/src/ollama_panel/*`
- WarpUI element types in `warpui_core`: `Flex`, `Stack`, `Scrollable`, `ClippedScrollable`, `UniformList`, `Table` (`elements/table/mod.rs`), `Draggable` (`elements/drag/draggable.rs`), `DropTarget` (`elements/drag/drop_target.rs`), `Hoverable`, `SavePosition`. — `git grep 'pub struct …' upstream/main -- crates/warpui crates/warpui_core crates/ui_components`
- **No calendar or date-picker component** in `crates/warpui`, `crates/warpui_core` or `crates/ui_components` (`git grep -il 'calendar\|date_picker\|datepicker'` → no matches).
- WarpUI is large and lightly documented: `crates/warpui` has about 32,956 lines and `crates/warpui_core` about 69,616 (338 `.rs` files). The only learning material is `crates/warpui/examples/*`; I found no guide or README for the framework. — `git ls-tree -r upstream/main -- crates/warpui crates/warpui_core`
- AGENTS.md (written in Chinese) describes the architecture: an Entity-Handle system (the `App` owns all views, and views refer to each other by `ViewHandle<T>`), and a declarative Element tree plus Action events, "Flutter style". It also says `app/src/ai` has 389 files. — `git show upstream/main:AGENTS.md`
- Build setup:
  - The toolchain is pinned to Rust 1.92.0 (`rust-toolchain.toml`).
  - The dev profile uses `debug = "line-tables-only"` "to speed up dev builds", and `release-lto` uses thin LTO (`git show upstream/main:Cargo.toml`).
  - Zap's release workflow gives the macOS job `timeout-minutes: 360`, with a comment saying the macOS runners have no sccache and do a "full cargo build + bundle + create-dmg" (`git show upstream/main:.github/workflows/zap_release.yml`, lines 150–152). A timeout is a budget, not a measured build time.
  - The user's own note in `c3509bc2` (2026-05-14): "`cargo build --release --bin unwarp` clean (3m 47s)". This was probably an incremental build.
- I found no published clean-build times for Warp. A search returned nothing from Warp engineering. — [search results incl. Warp repo](https://github.com/warpdotdev/warp)
- Fit for a Python-first developer: **31 of the 33** Narwarp commits carry a `Co-Authored-By: Claude` trailer (`git log --format=%B`). **30 of the 33** have author dates between 2026-05-13 and 2026-05-15, and those 30 commits account for 9,535 of the 10,596 lines inserted across all 33 commits (summing `git show --numstat`). So most of the layer was produced in about three days of Claude-assisted work, followed by the June rebrand, sync and icon commits.

### Inferences
- Rough sizes for the planned views in WarpUI, scaled from the existing panels:
  - Kanban board (4 columns × project × Work/Life, cards, drag between columns): about 1.5–2.5k lines of Rust, plus models, a Diesel migration, `schema.rs` edits and `view.rs`/`left_panel.rs` wiring.
  - Week or month calendar grid with time blocks: about 1–2k lines, with no calendar prior art in the codebase to reuse.
  - Threaded per-ticket and per-project message board: about 1–1.5k lines, possibly reusing Warp's AI blocklist rendering.
  - Integrations (Linear, Google Calendar, Discord, GitHub, email) in Rust: workable, but the Python SDKs and MCP servers are much more mature. That area belongs to other researchers.
  - Total: about 4–7k more lines of Rust inside a 22k-line hub file and the AGPL app crate.
- Time to see a change: every edit rebuilds the `app` crate, measured in minutes, compared with seconds for a small Python/Textual app. With an AI writing the code, a faster check-and-fix loop matters more than how fast code gets typed.
- The user has shown they can ship Rust with Claude, so skill is not a hard blocker. The real costs are slow iteration, debugging WarpUI semantics (entity handles, `ViewContext`/`AppContext` lifetimes) without documentation, and having the harness exist only inside one GUI terminal.

### Gaps
- Actual clean and incremental build times on the user's Mac were not measured, per the constraints.
- I did not evaluate whether Warp's new WarpUI TUI layer (`608bc53`, MIT, on ratatui-core) could be reused on its own. It is not in Zap.

---

## 4. Porting the existing Narwarp work to Python

### Takeaway
Most of the ~10k-line layer carries over as design rather than code. The retrieval-augmented generation (RAG) core (schema, chunker, a redactor with 7 regex patterns plus Luhn and entropy checks, the Ollama embedding client, the vault-note indexer) maps directly onto Python with the `sqlite-vec` package, and would shrink to an estimated 1–1.5k lines. The WarpUI-specific UI code (~1.7k lines) is largely replaced by Textual's built-in `DirectoryTree`, `MarkdownViewer`, `DataTable` and `Input` widgets. The command-block indexer depends on Warp's terminal Blocks and does not port.

### Cited Findings
- The schema (`crates/persistence/migrations/2026-05-15-000000_unwarp_vault_rag/up.sql`, 95 lines) defines:
  - Tables `vault_files`, `seed_commands`, `unwarp_conversations`, `unwarp_messages` and `unwarp_conversation_blocks`.
  - Three sqlite-vec `vec0` virtual tables declared `embedding float[768]`: `vec_vault_notes`, `vec_command_blocks` and `vec_messages`. Each has a companion chunk table that shares its rowid.
  - `ALTER TABLE blocks ADD COLUMN redacted_output TEXT`, which modifies Warp's own `blocks` table.
  - Source: `git show P4:…/up.sql`
- Embedding settings in `rag/embed.rs` (308 lines):
  - `DEFAULT_EMBED_MODEL = "nomic-embed-text"` (768-dimensional).
  - `DEFAULT_OLLAMA_BASE_URL = "http://localhost:11434/"`, calling `{base_url}api/embed`.
  - `EMBED_PARALLELISM = 2`, because Ollama serves one model request at a time.
  - A `ModelNotFound` error tells the user to run `ollama pull <model>`; calls go through the `genai` crate.
  - Source: `git show P4:app/src/rag/embed.rs`
- Redaction in `rag/redact.rs` (416 lines):
  - 7 built-in regexes: AWS `AKIA…` keys, GitHub tokens `gh[poushr]_…`, `Bearer` tokens, JWTs, `ENV_VAR=` assignments, PEM private keys, and credit-card candidates.
  - A Luhn check on the card candidates, a Shannon-entropy pass, and user-supplied extra patterns.
  - 18 unit tests.
  - Source: `git show P4:app/src/rag/redact.rs`
- Chunking in `rag/chunk.rs` (310 lines): a `ChunkConfig` with a paragraph-aware sliding window, plus `chunk_command_block`, which prepends the command to every output chunk. 11 tests.
- Other modules: `store.rs` 371, `query.rs` 158, `index/{vault_notes 233, command_blocks 156, conversations 148, mod 64}`, `index_manager.rs` 171. Vault: `manager.rs` 333, `layout.rs` 201, `lock.rs` 302, `config.rs` 183, `mirror/job.rs` 673, `brew/job.rs` 489, and UI (`explorer/panel.rs` 910, `tree.rs` 342, `markdown_view/viewer.rs` 254). There are about 100 `#[test]`/`#[tokio::test]` functions across the added `rag` and `vault` files. — `git show P4:<file> | wc -l`, `grep -c '#\[test\]'`
- Commit `31d5df89` describes the redaction module and chunker as "standalone, fully tested", meaning they have no dependency on Warp internals.
- sqlite-vec from Python:
  - Install with `pip install sqlite-vec`, then call `db.enable_load_extension(True); sqlite_vec.load(db)`.
  - `serialize_float32()` packs lists of floats; NumPy arrays work if cast with `.astype(np.float32)`.
  - macOS caveat: "The default SQLite library that is bundled with Mac operating systems do not include support for SQLite extensions". The docs suggest Homebrew Python or `pysqlite3` instead.
  - SQLite 3.41 or later is recommended.
  - Source: [sqlite-vec Python docs](https://alexgarcia.xyz/sqlite-vec/python.html)
  - The latest PyPI release is 0.1.9 (2026-03-31). It is pre-1.0, and the release is more than 6 months old. — [PyPI JSON](https://pypi.org/pypi/sqlite-vec/json)
- Textual's built-in widgets include DirectoryTree, Tree, Markdown, MarkdownViewer, DataTable, TextArea, Input, MaskedInput, ListView, OptionList, SelectionList, TabbedContent, Collapsible, ContentSwitcher, RichLog, Log, Sparkline, ProgressBar, Select and Tooltip. **There is no calendar, kanban or terminal widget.** — [textual/widgets/__init__.py](https://github.com/Textualize/textual/blob/main/src/textual/widgets/__init__.py)
- Other assets worth keeping:
  - The Tusk persona: a 23-line minijinja template covering voice, traits and "never does" rules (`git show P4:app/src/ai/agent_providers/prompts/system/tusk_persona.j2`).
  - 11 narwhal sprite files in `assets/sprites/narwhal/` plus `script/slice_sprites.py`, which is already Python.
  - The `/search /docs /chat /sugg` slash-command semantics: `ae6c9354`, `b4cd600c`, 319 lines in `slash_commands/mod.rs`.

### Inferences
- Rough Python port sizes:

  | Component | Estimated Python LOC | Notes |
  |---|---|---|
  | Redactor | ~120–160 | The patterns use only inline flags (`(?i)`, `(?m)`, `(?s)`) that Python's `re` also supports, so they should transfer almost verbatim. |
  | Chunker | ~80–120 | |
  | Store and query | ~150–250 | sqlite-vec plus the stdlib `sqlite3` module or `sqlite-utils`. |
  | Embedding client | ~50–80 | `httpx` or the `ollama` package. |
  | Vault-note indexer, index manager and file watcher | ~200–300 | |
  | Vault manager, layout, lock and config | ~400–600 | |
  | Mirror and brew jobs | ~400–600 | |
  | **Total** | **~1.5–2.3k** | The ~100 Rust tests can serve as test vectors. |

- The schema carries over nearly verbatim (768-dim `vec0` tables, metadata tables sharing the rowid) except the `ALTER TABLE blocks` line. The command-block indexer needs a different source of shell history, such as shell hooks or atuin (not researched). The conversation indexer becomes the harness's own message-board and conversation store, which Narwarp needs anyway.
- About 1.7k lines of WarpUI UI code (explorer, tree, viewer, settings page, Ollama panel) are dropped rather than ported. Textual built-ins cover them in a few hundred lines.
- Worth keeping: the Tusk persona text, the sprites, the vault folder layout and its mirror/brew job behaviour, the redact-before-embed rule, and the choice of nomic-embed-text at 768 dimensions.

### Gaps
- Whether the Python builds that uv installs (python-build-standalone) on macOS support `enable_load_extension` was not verified.
- The Python line counts are estimates, not measurements.

---

## 5. TUI frameworks, existing apps to borrow from, and showing live agent sessions

### Takeaway
For a Python-first developer, Textual is still the most productive fit: a large widget set, CSS layouts, async workers, browser serving through textual-serve, and an existing Textual kanban app that also runs as an MCP server. But Textual's development has slowed sharply since June 2026 under a single volunteer maintainer. Ratatui (Rust), Bubble Tea v2 (Go), Ink 8 and OpenTUI (TypeScript) all have recent releases. No Python option embeds a live Claude Code terminal well. Running sessions in tmux panes (the same mechanism Claude Code's agent teams use), or rendering headless event streams, is the practical path.

### Cited Findings
**Framework comparison** (versions from registry APIs, fetched 2026-10-13)

| Framework | Language | Latest release (date) | Activity | Relevant facts | Sources |
|---|---|---|---|---|---|
| **Textual** | Python ≥3.9 | 8.2.8 (2026-06-30); 24 releases in 2026 | Commits/month on main: Mar 59, Apr 53, May 70, **Jun 19, Jul 2, Aug 0, Sep 0, Oct 1** (to 10-13) | MIT; CSS-style layouts, including grid; `@work` async and thread workers with `exclusive=True`, cancellation, and `call_from_thread`/`post_message` to update the UI; the widgets listed in §4 | [PyPI](https://pypi.org/pypi/textual/json); commit counts from a commits-only clone of https://github.com/Textualize/textual; [workers guide](https://textual.textualize.io/guide/workers/); [layout guide](https://textual.textualize.io/guide/layout/) |
| textual-serve | Python | 1.1.3 (2025-11-01) | No 2026 releases (older than 6 months) | Serves Textual apps in a browser; kanban-tui (`--web`) and Toad (`toad serve`) use it | [PyPI](https://pypi.org/project/textual-serve/) |
| **Ratatui** | Rust | 0.30.2 (2026-06-19); 0.30.0 (2025-12-26) | 59.5M downloads | Immediate-mode rendering; the `tui-term` PTY widget is at 0.3.4 (2026-04-07); Warp's own TUI is built on ratatui-core (`608bc53`) | [crates.io](https://crates.io/crates/ratatui), [tui-term](https://crates.io/crates/tui-term), [Ratatui rendering](https://ratatui.rs/concepts/rendering/) |
| **Bubble Tea v2** | Go | v2.1.0 (2026-10-08); v2.0.0 (2026-02-24) | v1 line last released v1.3.10 (2025-09-17) | Module path `charm.land/bubbletea/v2` | [Go proxy @latest](https://proxy.golang.org/charm.land/bubbletea/v2/@latest), [v2.0.0 info](https://proxy.golang.org/charm.land/bubbletea/v2/@v/v2.0.0.info) |
| **Ink** | TypeScript (React) | 8.0.0 (2026-10-03); 7.0.0 (2026-04-08) | Major releases every ~6–14 months (4.0.0 in 2023-03 → 8.0.0 in 2026-10) | Gemini CLI depends on a fork, `"ink": "npm:@jrichman/ink@6.6.9"`, with React 19.2.4 | [npm](https://registry.npmjs.org/ink), [gemini-cli package.json](https://github.com/google-gemini/gemini-cli/blob/main/packages/cli/package.json) |
| **OpenTUI** | Zig core + TypeScript bindings (React, Solid) | @opentui/core 0.6.3 (2026-10-13) | Commits/month Mar–Oct 2026: 81, 84, 92, 45, 29, 111, 41, 68; 181 commits by Simon Klee since July | "OpenCode uses OpenTUI in production for millions of users"; developing it requires Bun ≥1.4.1 and Zig 0.16.0; MIT; 13.5k stars; pre-1.0 | [GitHub](https://github.com/sst/opentui), [npm](https://registry.npmjs.org/@opentui/core) |

- Textual's maintenance situation:
  - The company behind it wound down in May 2025: "Textualize, the company, will be wrapping up in the next few weeks… Textual will live on as an Open Source project." — [Textual blog, 2025-05-07](https://textual.textualize.io/blog/2025/05/07/the-future-of-textualize/) (older than 6 months)
  - On 2026-10-09 the maintainer added a note to the bug-report template: "I have maintained Rich and Textual for many years, and I will continue to do so. But please be aware that it takes a lot of my time … you may have to wait for a response … For funded Enterprises, the best way to get a quick response it to sponsor me." — [bug_report.md](https://github.com/Textualize/textual/blob/main/.github/ISSUE_TEMPLATE/bug_report.md) (commit `5e5b7ef`)
  - Rich 15.0.0 was released 2026-04-12. — [PyPI](https://pypi.org/project/rich/)
  - A search summary claimed Textual's last release was on 2026-05-19. The PyPI JSON API shows 8.2.8 on 2026-06-30, which is the primary source.
- Claude Code:
  - Practitioner and secondary reports say Anthropic kept React as the component model but replaced Ink's renderer with one that diffs terminal cells, shipping around v2.0.72 in December 2025. — [steipete.me](https://steipete.me/posts/2025/signature-flicker), [AIM](https://analyticsindiamag.com/ai-news-updates/claude-code-uses-react-to-render-its-tui) (secondary, older than 6 months)
  - The npm package `@anthropic-ai/claude-code` 2.1.296 (2026-10-09) has no regular dependencies, only six platform-specific optional dependencies (for example `…-darwin-arm64`), which points to native-binary distribution. — [npm registry](https://registry.npmjs.org/@anthropic-ai/claude-code/latest)
- Other agent CLIs: OpenCode `opencode-ai` 1.19.0 (2026-10-11) and Gemini CLI 0.63.0 (2026-10-06). — [npm](https://registry.npmjs.org/opencode-ai), [npm](https://registry.npmjs.org/@google/gemini-cli)
- Notable Textual apps and how active they are (PyPI):
  - Posting 2.11.1 (2026-10-05) and Harlequin 2.16.1 (2026-10-02): active.
  - kanban-tui 0.21.2 (2026-08-28): active.
  - **Toad** (`batrachian-toad`) 0.6.20 (2026-05-26): the repo's last commit was also 2026-05-26, so it has been quiet for 4.5 months. It requires Python 3.14.
  - **Elia** (`elia-chat`) 1.10.0 (2024-09-14): stale.
  - Sources: [Posting](https://pypi.org/project/posting/), [Harlequin](https://pypi.org/project/harlequin/), [kanban-tui](https://pypi.org/project/kanban-tui/), [Toad](https://pypi.org/project/batrachian-toad/), [Elia](https://pypi.org/project/elia-chat/)
- Toad's README:
  - It talks to agents over ACP, and anyone can add an agent "via the Agent Client Protocol".
  - It "integrates a fully working shell with full-color output, interactive commands, and tab completion".
  - `toad serve` runs it as a web app; it supports Linux and macOS, with Windows through WSL.
  - **License: AGPL**, with commercial licenses available.
  - Source: [Toad README](https://github.com/batrachianai/toad)

**Existing TUI kanban and calendar apps to borrow from**
- **kanban-tui** (Zaloog, Python/Textual, MIT, 286 stars):
  - Multiple boards; default columns Ready/Doing/Done/Archive, which can be extended or reordered.
  - Move cards with H/L/J/K or "Mouse drag-and-drop"; task dependencies with blocking checks; categories; plotext charts; an audit table filled by triggers.
  - Backends: SQLite by default, Jira via JQL, and a `claude` backend that reads `~/.claude/tasks/*.json`.
  - "`kanban-tui` can be run as a local mcp server" (`ktui mcp`), and `ktui skill init` generates an agent SKILL.md.
  - `--web` mode through textual-serve; Python ≥3.11.
  - Source: [GitHub](https://github.com/Zaloog/kanban-tui)
- **Backlog.md** (MrLesk, MIT, 7.0k stars; npm 1.53.0, 2026-09-24):
  - Each task is a Markdown file in `backlog/`.
  - `backlog board` shows a terminal kanban; `backlog browser` serves a local web board with drag-and-drop on 127.0.0.1.
  - An optional MCP server works with Claude Code, Codex, Gemini CLI and Kiro, and "finds the active project from your client's MCP roots".
  - Sources: [GitHub](https://github.com/MrLesk/Backlog.md), [npm](https://registry.npmjs.org/backlog.md)
- **calcure** 3.4 (2026-08-29) and **khal** 0.14.1 (2026-08-21) are both maintained terminal calendars. — [PyPI calcure](https://pypi.org/project/calcure/), [PyPI khal](https://pypi.org/project/khal/)
- **taskwarrior-tui** (Rust, built on taskwarrior) has latest tag v0.27.0, and **lazygit**, a useful reference for multi-pane keyboard-driven layouts, has latest tag v0.66.1. — [taskwarrior-tui](https://github.com/kdheepak/taskwarrior-tui), [lazygit](https://github.com/jesseduffield/lazygit) (`git ls-remote --tags`)

**Showing live agent sessions inside a TUI**
- Embedding a terminal in Python is weak:
  - Textual has no terminal widget (see the widget list in §4).
  - `textual-terminal` was last released as 0.3.0 on 2023-01-29, and `pyte`, the VT emulator it relies on, as 0.8.2 on 2023-11-12. Both are stale. — [PyPI](https://pypi.org/project/textual-terminal/), [PyPI](https://pypi.org/project/pyte/)
  - Toad has its own integrated shell, but it is AGPL (see above).
- In Rust, `tui-term` 0.3.4 provides a maintained PTY widget for Ratatui. — [crates.io](https://crates.io/crates/tui-term)
- Claude Code's own agent teams use either "in-process" display or "split panes", where each teammate gets its own pane; split panes need tmux or iTerm2. `teammateMode` accepts in-process, auto or tmux, and `iterm2` was added in v2.1.186. — [Claude Code agent teams docs](https://code.claude.com/docs/en/agent-teams)
- Headless mode: Claude Code can run non-interactively (`claude -p`) and stream structured output, which a TUI can render as message-board events. — [Claude Code headless docs](https://code.claude.com/docs/en/headless)
- tmux can create and inspect panes from a script (`split-window`, `new-window`, `capture-pane`, `send-keys`). — [tmux(1)](https://man.openbsd.org/tmux.1)
- The Warp Agent CLI takes a third approach and runs its own PTY inside its TUI. — [Warp docs](https://docs.warp.dev/agents/cli/)

### Inferences
- **Pick Textual for v1**, and make the TUI a thin client of a local daemon API. If Textual stalls further, the UI can be swapped (to Ratatui, Bubble Tea, OpenTUI or a small web UI) without touching the core.
  - Textual is MIT-licensed and at major version 8, so its API is stable.
  - Active downstream apps (Posting, Harlequin, kanban-tui) still ship on it.
  - It matches the user's skills and Python's integration ecosystem.
  - The commit cadence since June 2026 and the 2026-10-09 note are real risk signals. Expect slow responses to upstream bugs, and pin the version.
- Building the views in Textual:
  - A kanban board can follow kanban-tui's patterns (MIT, so code can be borrowed with attribution): `Horizontal` containers of scrollable `ListView`s or custom card widgets, with key- and mouse-driven moves.
  - A calendar week grid needs a custom widget on the CSS grid layout or a `DataTable`, about 400–800 lines.
  - Message boards can be built from `ListView`/`RichLog` plus `Markdown` and `TextArea`.
- Ranking of options for live sessions:
  1. Run each ticket's Claude Code or Ollama agent in a named tmux window or pane. The TUI shows status and the latest output via `capture-pane`, and attaches on demand. Of the three, this is the most robust.
  2. Run agents headless and render the structured events into the ticket's message board.
  3. Embed a PTY inside Textual. In Python this is fragile and unmaintained.
- If the user later wants the board inside a terminal, Zap/Narwarp's existing CLI-agent blocks and notification center already show Claude Code sessions, so they can stay the shell while the harness lives outside.

### Gaps
- No public Textual roadmap or explanation for the post-June slowdown was found.
- Bubble Tea v2 and Ink 8 changelogs were not reviewed in detail.
- No primary source documents Claude Code's current renderer.
- Feature details of calcure, khal and taskwarrior-tui (CalDAV/ICS support, licenses) were not verified beyond their release metadata.
- Zellij was not researched as an alternative to tmux for pane management.

---

## 6. AGPL implications

### Takeaway
Purely personal, unpublished use of the AGPL fork creates no obligation to release source. Obligations start only when the user distributes the program to others or lets others use a modified version over a network (AGPL §13). A new standalone repo containing only the user's own original code, or a Python rewrite of their own designs, can use any license. Copying in any Warp or Zap AGPL file makes the combined work AGPL when it is shared. WarpUI (`warpui`, `warpui_core`) is MIT and can be copied with its notice.

### Cited Findings
- Private use: "The GPL does not require you to release your modified version, or any part of it." Private use of modifications is allowed, and releasing them is the user's choice. — [GNU GPL FAQ](https://www.gnu.org/licenses/gpl-faq.en.html)
- Network use: the AGPL requires that "modified versions of the software offer all users interacting with it over a computer network" a way to get the source. — [GNU GPL FAQ](https://www.gnu.org/licenses/gpl-faq.en.html); the clause is AGPL-3.0 §13 — [AGPL-3.0 text](https://www.gnu.org/licenses/agpl-3.0.html)
- Combined works: "The GPL says that the whole combined program has to be released under the GPL." Permissive code can go into GPL works: "The X11 license is compatible with the GPL". MIT is the X11-style license. — [GNU GPL FAQ](https://www.gnu.org/licenses/gpl-faq.en.html)
- The repo's split: `warpui` and `warpui_core` are MIT and everything else is AGPL-3.0-only. — [Warp README](https://github.com/warpdotdev/warp); `git show upstream/main:crates/warpui/Cargo.toml`
- Licenses of projects that might be borrowed from: Toad is AGPL with a commercial option ([README](https://github.com/batrachianai/toad)); kanban-tui is MIT ([GitHub](https://github.com/Zaloog/kanban-tui)); Backlog.md is MIT ([GitHub](https://github.com/MrLesk/Backlog.md)); OpenTUI is MIT ([GitHub](https://github.com/sst/opentui)); Textual is MIT ([GitHub](https://github.com/Textualize/textual)).
- Narwarp's `origin` is `https://github.com/lilaland/narwarp` (`git remote -v`). Whether that repo is public was not checked.

### Inferences
- **Option (a)**: any public Narwarp repo or binary must stay AGPL-3.0, keeping the license files and providing source. If the user exposes a modified Narwarp to other people over a network, §13 applies. Purely personal use carries no obligation.
- **Option (b) or (c)**: a fresh Python repo is not a derivative work if it reimplements the user's own designs (schema, redaction rules, chunker) without copying Warp or Zap code, so it can be MIT or private. Modules the user wrote that do not depend on Warp internals (commit `31d5df89` calls redact/chunk "standalone") are the user's own copyright to relicense. Code that calls into Zap's app internals is better rewritten than copied.
- Borrowing from Toad would make the new harness AGPL. Borrowing from kanban-tui or Backlog.md only requires keeping their MIT notices.
- These notes are not legal advice. Who owns the copyright in code Claude co-authored is an open question.

### Gaps
- Not reviewed by a lawyer.
- The copyright status of AI-co-authored code (31 of 33 commits) and the exact "derivative work" boundary for modules that use Zap internals were not analysed.

---

## 7. Recommendation: (a) keep building in the fork, (b) standalone harness, or (c) hybrid

### Takeaway
**Choose (c), weighted heavily toward (b).**
- Build the harness as a standalone Python core: a local daemon, a SQLite + sqlite-vec store, and an MCP server so Claude Code and Ollama agents can read and write tickets and boards.
- Make a Textual TUI its first client, so it runs in any terminal.
- Freeze Narwarp at P4 as the user's daily shell.
- Do **not** spend about a day merging upstream now unless a specific Zap fix is needed. Zap is nearly dormant, and the 60 commits are mostly Windows, i18n and SSH fixes.
- Revisit Narwarp/Zap integration later only as a thin client of the daemon API, for example a left panel or slash commands that call it. This matches Zap's own roadmap principle that "the terminal becomes one client among many".

### Cited Findings
- Zap stopped taking Warp changes on 2026-05-07, had 0 commits in August, 4 in September and 0 in October 2026, and last tagged a release on 2026-07-09. — `git log`, `git ls-remote --tags` (§1)
- Warp itself is building ACP harness support, a Warp Agent CLI TUI and multi-agent orchestration. — `02668f9`, `683d407`; [Warp docs](https://docs.warp.dev/agents/cli/) (§1)
- A merge today conflicts in 4 files (9 hunks, about 467 lines), and the last sync needed 88 compile fixes. — trial merge; `6e14311e` (§2)
- WarpUI has no calendar component and no documentation, and the existing panels cost 515–910 lines each. — `git grep`; `git show` (§3)
- Zap's roadmap: a "standalone open-source agent service … not tied to Warp's existing client code". — [roadmap](https://github.com/zerx-lab/zap/blob/main/docs/roadmap.md)
- There is prior art for the harness's board-plus-MCP shape: kanban-tui runs as an MCP server and reads Claude task files, and Backlog.md offers Markdown tasks plus an MCP server plus a terminal and web board. — [kanban-tui](https://github.com/Zaloog/kanban-tui), [Backlog.md](https://github.com/MrLesk/Backlog.md)
- The main risk of choosing Textual: commit activity has been near zero since July 2026, and the maintainer note of 2026-10-09 warns of slow responses. — §5 sources

### Inferences
**Why not (a):** Every planned feature is mainly about data, orchestration and integrations, with the UI as the smaller part. Putting all of it inside a solo-maintained, 1.14M-line Rust GUI fork:
- ties the harness to one terminal binary;
- leaves agents unable to reach it except through bespoke code (they reach a Python MCP server directly);
- grows the conflict surface in the hub files with every feature;
- means waiting minutes for each Rust rebuild before seeing a change.

The upstream that would justify paying the merge cost has gone quiet, and its own upstream (Warp) is out of reach because of 2,193 commits of drift.

**Why (b) or (c):**
- It is the user's strongest language and has the richest integration ecosystem: the Google, Linear, Discord, GitHub and IMAP SDKs and the MCP Python SDK (covered by other researchers).
- The RAG and vault design ports in about 1.5–2.3k lines.
- Textual covers kanban, list, markdown, tree and table views out of the box, and kanban-tui shows drag-and-drop and MCP working in Textual.
- It runs in any terminal, including Narwarp, official Warp, Ghostty and iTerm2, and can be served to a browser with textual-serve.
- It keeps the option to add a thin Narwarp client later.

**Rough effort comparison** (inference; assumes Claude Code writes most of the code, as it did for the 31/33 co-authored commits):

| Work item | (a) Narwarp/WarpUI (Rust) | (b)/(c) Python core + Textual |
|---|---|---|
| Sync upstream now | 0.5–1.5 days, then the ongoing merge tax | Optional; Narwarp stays frozen |
| Kanban (4 states × project × Work/Life) | ~1.5–2.5k LOC plus a Diesel migration and hub-file wiring | ~0.4–0.8k LOC, borrowing patterns from kanban-tui (MIT) |
| Calendar week grid | ~1–2k LOC, no prior art in WarpUI | ~0.4–0.8k LOC custom widget |
| Threaded ticket and project boards | ~1–1.5k LOC | ~0.3–0.6k LOC |
| Store, daemon and MCP server for agents | ~1–2k LOC of Rust, or a separate service anyway | ~0.8–1.5k LOC |
| RAG and vault | Already exists | Port ~1.5–2.3k LOC |
| Time to see a change | Minutes (Rust rebuild of the `app` crate) | Seconds |
| Ongoing maintenance | Solo upkeep of the whole fork | Small codebase; Textual dependency risk |

**Risks and mitigations for (b)/(c):**

| Risk | Mitigation |
|---|---|
| Textual slows further | Pin the version; keep all logic in the daemon behind a versioned JSON-RPC/MCP API; fallbacks are OpenTUI (most active, but pre-1.0 and TypeScript), Bubble Tea v2 (Go, stable v2.1.0), Ratatui 0.30 (Rust, reuses Rust skills) or a small local web UI |
| Live agent sessions in a TUI | Use tmux panes or headless event streams rather than embedding a PTY |
| sqlite-vec on macOS (no extension loading in the system SQLite) | Use Homebrew Python or `pysqlite3`; check uv-managed Python builds |
| Losing Narwarp conveniences while the new tool is built (vault explorer, `/search /docs /chat /sugg`) | Keep using the frozen Narwarp until the Python equivalents (TUI commands, MCP tools) exist |
| sqlite-vec is pre-1.0 | Its alpha releases have come regularly through 2026; keep the vector store behind an interface |

**What to salvage from Narwarp:**
- The SQL schema, minus the Warp `blocks` change.
- The redaction patterns, Luhn and entropy logic, and their tests as test vectors.
- The chunker parameters and the rule of prepending the command to command-block chunks.
- The Ollama embedding defaults (nomic-embed-text, 768 dimensions, parallelism 2) and the model-not-found handling.
- The vault layout, lock, mirror (README/AGENTS/CLAUDE files mirrored into the vault) and brew-docs jobs.
- The Ollama panel's endpoints for running and pulled models.
- The Tusk persona prompt, the narwhal sprites and `slice_sprites.py`.
- The slash-command semantics, which become TUI commands and MCP tools.

**When to revisit:**
- If Zap actually ships its Phase 1 harness, or activity resumes: consider becoming a client of it, or contributing to it.
- If Warp's ACP harness runner makes official Warp or Zap a good host for Narwarp's agents: integrate through ACP or MCP rather than forking.
- If Textual becomes unmaintained: swap the client and keep the core.

### Gaps
- No build or test of the merged tree was run.
- Developer velocity in Python versus Rust for this user was not measured. The only data point is that 30 commits (~9.5k inserted lines) of the Rust layer were written in 3 days with Claude.
- Out of scope here and covered by other researchers: the orchestrator runtime architecture, message-board design, calendar-scheduling algorithms, app integrations, and Claude Code/Ollama session control.
