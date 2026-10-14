# AGENTS.md

How to work in this repository, for AI agents and for humans. Read this file first, then the design docs.

## What this is

Narwarp is a personal, local-first agentic harness made of four parts:

- **`narwarpd`:** a Python daemon that owns tickets, boards, the calendar plan and integrations.
- **`nw`:** the command-line client.
- **An MCP server:** the interface for Claude Code and local Ollama agents.
- **A Textual TUI:** the board in a terminal.

It will run on the owner's home Mac Studio (macOS, 36 GB). Development happens first, in cloud sessions; installation on the Mac comes after.

## Design sources

| File | Role |
|---|---|
| `docs/design/design-addendum.md` | Current decisions. Wins wherever it disagrees with the report |
| `docs/design/narwarp-agentic-harness-design.md` | The full design report, with 85 citations |
| `docs/design/seed-backlog.md` | Tickets NW-1 to NW-20 with acceptance criteria, plus a status block |
| `docs/design/api-contract.md` | The frozen socket API, CLI, MCP tool and DB schema contract (written in NW-9) |
| `docs/design/research/` | Evidence behind the decisions (eight research notes) |

Don't change an architectural decision without asking the owner. Propose it in the PR description instead.

## Architecture rules

1. `narwarpd` is the only writer of `narwarp.db` (SQLite in WAL mode, with FTS5). Everything else goes through its API.
2. Deterministic code owns state, scheduling and gates. Models classify, summarize, decompose, code and narrate, and code validates every model output before it changes state.
3. Agents never hold credentials. Tokens live in the macOS Keychain, behind an interface that has a file-backed fake for tests.
4. Approvals come only from the owner (trust tier T0). The daemon signs them and each can be used once. A message can never approve anything.
5. Board messages are typed and append-only. A `result` needs evidence, and only the owner marks a ticket Done.
6. Tusk, the conductor, only proposes. The reconciler applies accepted proposals.
7. Vault mirror: SQLite wins, only allowlisted fields are ingested, and nothing is ever written under the vault's `notes/`.
8. Work and Life never mix. Every row has a mode column, and routing between them happens in code.
9. Claude Code is driven only through the official `claude` binary: background sessions, hooks and `claude agents --json`. Never proxy, copy or store Claude OAuth tokens.

## Stack and conventions

- Python 3.12 or later, uv, ruff (lint and format), pytest with pytest-asyncio, and type hints throughout, checked in CI.
- The standard-library `sqlite3`, with migrations as numbered SQL files and no ORM.
- The official MCP Python SDK. Textual, pinned to an exact version. watchdog for file watching. keyring for secrets.
- Tests never touch the network, the real `claude` binary, Ollama, or the user's home directory. Use the fakes in `tests/fakes/`.
- macOS-only behavior (LaunchAgent, FSEvents, Keychain, privacy prompts) sits behind interfaces with Linux fallbacks, so CI runs on Ubuntu.
- Commands (filled in by NW-7): `uv sync`, `uv run pytest`, `uv run ruff check`, `uv run ruff format --check`.

## Working rules

- One branch and one PR per ticket. Branches are named `nw-<n>-<slug>`, and each PR title starts with the ticket key (for example `NW-8: schema and migrations`).
- Tick an acceptance criterion only with evidence: a test name, a CI run or a commit.
- Keep PRs small, and add tests with every change in behavior.
- Another session reviews each PR before the owner does. Don't merge to `main` without the owner's approval.
- Don't copy code from Warp, Zap or Toad, which are AGPL. Porting the owner's own redactor and chunker from `lilaland/narwarp` (branch `phase-4/watcher-and-polish`, `app/src/rag/redact.rs` and `chunk.rs`) is fine.
- No license file until the owner chooses one.
- Never commit secrets. The redactor runs on test fixtures too.

## Hardware and local models (Mac Studio, 36 GB)

| Role | Model |
|---|---|
| Triage and routing | `qwen3.5:9b`, with `nimble:9b` tested side by side |
| Digests | `gemma4:12b` or `qwen3.5:9b` |
| Quarantined reader | `qwen3.5:9b` |
| Embeddings | `qwen3-embedding:0.6b`, with `nomic-embed-text` kept for continuity tests |
| Coding | Claude only. There is no local coding lane at 36 GB |

- Set `OLLAMA_CONTEXT_LENGTH` explicitly. Don't rely on Ollama's memory-based default.
- Repos live in `~/code` and the vault in `~/vault`, never under `~/Documents`, `~/Desktop` or `~/Downloads`.
