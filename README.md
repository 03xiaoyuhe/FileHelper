# FileHelper

A **vault-scoped** local file organizer with multi-level tags, AI classification, and git-style commits.

## What it does

FileHelper helps you organize files in a "vault" (any folder you designate). Each vault is an independent project with its own tag library, configuration, and rules.

### Highlights

- **Vault-scoped** — One folder = one project. Explicit boundary, no accidental cross-project operations.
- **Multi-level tags** — Hierarchical labels like `ai/ml/transformers`. Flat storage, prefix queries (`ai/*`).
- **Folder roles** — `inbox` (auto-organize), `tracked` (index only), `ignored` (skip). Sub-folders inherit by default.
- **Git-style workflow** — `mv` / `rm` → `status` → `commit` → `revert`. Every batch operation is fully undoable.
- **AI classification** — Pluggable backends: Ollama (local, privacy-friendly) / OpenAI / DeepSeek. Per-folder overrides.
- **Declarative rules** — YAML-based, optionally combined with AI for hybrid logic.
- **Safe by default** — Deletes go to a recoverable trash bin. `commit revert` reverse-applies all actions.
- **Duplicate detection** — Non-intrusive heads-up: 🔴 same-name-same-content, 🟡 renamed-duplicates. Reported but never auto-removed.

## Status

🚧 **In design phase** — Phase 1 (CLI MVP) implementation pending.

| Phase | Scope |
|---|---|
| Phase 1 (MVP) | vault / scan / mv / rm / tag / status / commit / check |
| Phase 2 | AI classify + rule engine + Textual TUI |
| Phase 3 | Watch mode + cross-machine sync + standalone binary |

## Documentation

📖 **[Full design document → doc/DESIGN.md](doc/DESIGN.md)** — start here

### Sub-design documents

| # | Topic | File |
|---|---|---|
| 01 | Vault management | [doc/design/01-vault.md](doc/design/01-vault.md) |
| 02 | Scanner + Reconciler | [doc/design/02-scan.md](doc/design/02-scan.md) |
| 03 | Multi-level tags | [doc/design/03-tag.md](doc/design/03-tag.md) |
| 04 | File operations (mv/rm) | [doc/design/04-file-ops.md](doc/design/04-file-ops.md) |
| 05 | Trash bin | [doc/design/05-trash.md](doc/design/05-trash.md) |
| 06 | status / diff | [doc/design/06-status-diff.md](doc/design/06-status-diff.md) |
| 07 | Commit workflow | [doc/design/07-commit.md](doc/design/07-commit.md) |
| 08 | AI classification | [doc/design/08-classify.md](doc/design/08-classify.md) |
| 09 | Rule engine | [doc/design/09-rule.md](doc/design/09-rule.md) |
| 10 | Health check + duplicates | [doc/design/10-check.md](doc/design/10-check.md) |
| 11 | Folder roles | [doc/design/11-folder.md](doc/design/11-folder.md) |
| 12 | Query commands | [doc/design/12-query.md](doc/design/12-query.md) |
| 13 | Configuration | [doc/design/13-config.md](doc/design/13-config.md) |

## Tech stack

- **Language**: Python 3.11+
- **CLI framework**: Click
- **Terminal UI**: Rich (Phase 1) · Textual (Phase 2 TUI)
- **Storage**: SQLite
- **AI backends**: Ollama / OpenAI / DeepSeek (optional, pluggable)

## Quick start

> Implementation pending. Once Phase 1 is ready:

```bash
pip install filetidy

# Initialize a vault
cd /path/to/your/files
filetidy init .

# Scan and index
filetidy scan

# Move / delete files (operations enter uncommitted state)
filetidy mv inbox/paper.pdf projects/ai/
filetidy rm inbox/broken.tmp

# Tag files
filetidy tag projects/ai/paper.pdf ai/ml paper

# Review and commit
filetidy status
filetidy diff
filetidy commit -m "Triage inbox PDFs"

# Undo if needed
filetidy commit revert HEAD
```

## License

TBD
