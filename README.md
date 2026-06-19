# FileHelper

A **vault-scoped** local file organizer designed as an **Agent Harness** — reliable file operation primitives + git-style change management + fast queries, so any AI agent (or human) can drive it.

## What it does

FileHelper manages files in a **vault** (any folder you designate). Each vault is an independent project with its own index, tag library, and history.

**The Harness philosophy**: the tool is reliable and dumb; the agent (Claude Code, Cursor, Cline, or human) is smart. FileHelper provides safe, undoable primitives; the agent decides what to do.

### Highlights

- **Vault-scoped** — One folder = one project. Explicit boundary, no accidental cross-project operations.
- **Agent Harness** — No built-in AI. Any agent can drive it via CLI. Decisions are external; tooling is reliable.
- **Git-style workflow** — `mv` / `cp` / `rm` → `status` → `commit` → `revert`. Every operation is fully undoable (except `PURGE`).
- **Recoverable deletes** — `rm` goes to a trash bin. `trash restore` brings it back. `trash empty` is the only irreversible op.
- **Multi-level tags** — Hierarchical labels like `ai/ml/transformers`. Flat storage, prefix queries (`ai/*`).
- **Fast queries** — `find` walks SQLite indexes (O(log n)), not full-table scans. Built for agents that query frequently.
- **Declarative rules** — YAML-based batch organizing (`rule run`). For templated, repeatable patterns. No AI.
- **Duplicate detection** — Non-intrusive heads-up: 🔴 same-name-same-content, 🟡 renamed-duplicates. Reported but never auto-removed.
- **File identity** — `(content_hash, filename)` is identity, **not path**. System tolerates duplicate identities across paths.

## Architecture: 5 parallel blocks

```
                  ┌──────────────────────────┐
                  │  CLI / Config (glue)     │
                  └────────────┬─────────────┘
          ┌────────┬───────────┼───────────┬────────┐
          ▼        ▼           ▼           ▼        ▼
       ┌─────┐ ┌─────┐    ┌─────────┐ ┌─────┐  ┌─────┐
       │ A.  │ │ B.  │    │   C.     │ │ D.  │  │ E.  │
       │Index│ │Ops  │    │ History │ │Tags │  │Rules│
       │     │ │     │    │         │ │     │  │     │
       └─────┘ └─────┘    └─────────┘ └─────┘  └─────┘
       What's  Safe file  Records +   Multi-   Templated
       there   ops        rollback    level    batch
                            tags      rules
```

| Block | Purpose |
|---|---|
| **A · Index** | Knows what files exist and their state |
| **B · Operations** | Safe file ops (`mv`/`cp`/`rm`/`mkdir`/`touch`), each reversible |
| **C · History** | Records every change, `commit`/`revert`/`diff` |
| **D · Tags** | Multi-level tags + fast `find` (indexed) |
| **E · Rules** | YAML declarative batch rules (no AI) |

## Status

🚧 **In design phase** — Phase 1 (CLI MVP) implementation pending.

| Phase | Scope |
|---|---|
| Phase 1 (MVP) | 5 blocks: Index / Operations / History / Tags / Rules |
| Phase 2 | ScanScheduler + Watch mode + Textual TUI + cross-machine sync |
| Phase 3 | Standalone binary (Nuitka) + shell completion + perf tuning |

## Documentation

📖 **[Full design document → doc/DESIGN.md](doc/DESIGN.md)** — start here

### Core protocols

| Topic | File |
|---|---|
| Action protocol (cross-block comms) | [doc/core/actions.md](doc/core/actions.md) |
| Shared SQLite schema | [doc/core/storage.md](doc/core/storage.md) |
| Configuration system | [doc/core/config.md](doc/core/config.md) |

### Block designs

| Block | File |
|---|---|
| A · Index | [doc/blocks/a-index.md](doc/blocks/a-index.md) |
| B · Operations | [doc/blocks/b-operations.md](doc/blocks/b-operations.md) |
| C · History | [doc/blocks/c-history.md](doc/blocks/c-history.md) |
| D · Tags | [doc/blocks/d-tags.md](doc/blocks/d-tags.md) |
| E · Rules | [doc/blocks/e-rules.md](doc/blocks/e-rules.md) |

## Tech stack

- **Language**: Python 3.11+
- **CLI framework**: Click
- **Terminal output**: Rich
- **Storage**: SQLite (shared by Block A & D)
- **History**: JSON files (commits/, similar to git)

## Quick start

> Implementation pending. Once Phase 1 is ready:

```bash
pip install filetidy

# Initialize a vault (creates .filehelper/ and scans baseline)
cd /path/to/your/files
filetidy init .

# Move / delete (operations enter uncommitted state, fully revertible)
filetidy mv inbox/paper.pdf projects/ai/
filetidy rm inbox/broken.tmp

# Tag files (fast find via SQLite index)
filetidy tag projects/ai/paper.pdf ai/ml paper
filetidy find "ai/*"

# Review and commit
filetidy status
filetidy diff
filetidy commit -m "Triage inbox PDFs"

# Undo if needed
filetidy commit revert HEAD
```

### As an Agent Harness

Any AI agent can drive FileHelper:

```bash
# Agent inspects state
filetidy ls --status missing
filetidy find "to-process"

# Agent makes decisions, executes via CLI (every op revertible)
filetidy mv 01-Inbox/paper.pdf 02-Projects/ai/
filetidy tag 02-Projects/ai/paper.pdf ai/ml paper

# Agent commits its work
filetidy commit -m "Agent: triaged inbox"

# Rollback on error
filetidy commit revert HEAD
```

## License

[MIT](LICENSE) © 2026 Terrence Chen
