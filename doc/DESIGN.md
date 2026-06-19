# FileHelper 设计文档

> **主索引文档**。详细设计在每个子文档中（`design/` 目录下）。
> Status: Draft · Last updated: 2026-06-19

---

## 0. TL;DR

FileHelper 是一个 **vault-scoped** 的本地文件整理工具。

**灵感来源**：
- **Obsidian** 的 vault 模型（一个文件夹 = 一个项目）
- **Eagle** 的标签系统（位置 + 标签双维度）
- **Git** 的工作流（mv/rm → status/diff → commit/revert）
- **organize** 的声明式规则

**技术栈**：Python 3.11+ · Click · Rich · SQLite · 可选 Ollama / OpenAI / DeepSeek

---

## 1. 核心概念

| 概念 | 说明 | 详见 |
|---|---|---|
| **Vault** | 被管理的文件夹，根有 `.filehelper/` | [01-vault.md](design/01-vault.md) |
| **Folder Role** | 子文件夹角色：`inbox` / `tracked` / `ignored` | [11-folder.md](design/11-folder.md) |
| **Tag** | per-vault 独立，多级（`ai/ml/transformers`） | [03-tag.md](design/03-tag.md) |
| **Commit** | 整理操作的原子单元（git-style） | [07-commit.md](design/07-commit.md) |
| **Trash** | 删除回收站，可恢复 | [05-trash.md](design/05-trash.md) |

---

## 2. 整体架构

```
┌──────────────────────────────────────────────────────────┐
│  Interface Layer                                         │
│  ┌──────────────┐  ┌──────────────────────────────┐     │
│  │ CLI (Phase 1)│  │ TUI (Phase 2 - Textual)      │     │
│  │ - Click      │  │ - Explorer 风 (tree + 详情)  │     │
│  └──────┬───────┘  └──────────────┬───────────────┘     │
└─────────┼─────────────────────────┼─────────────────────┘
          │   共用同一套 Core API    │
┌─────────▼─────────────────────────▼─────────────────────┐
│  Core Engine  (package: `filetidy/`)                    │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐      │
│  │ Scanner  │→ │ HashIdx  │← │   TagStore       │      │
│  │ (walk +  │  │ (SQLite, │  │ (CRUD on tags)   │      │
│  │ hash)    │  │ 主键哈希)│  │                  │      │
│  └────┬─────┘  └────┬─────┘  └────────┬─────────┘      │
│       └──────────────┼────────────────┘                │
│                      ▼                                  │
│            ┌──────────────────┐                         │
│            │   Reconciler     │ ← 命脉（MOVE/EDIT 检测）│
│            └────────┬─────────┘                         │
│                     │                                   │
│  ┌──────────┐  ┌────▼─────┐  ┌──────────────────┐      │
│  │RuleEngine│  │Classifier│  │  CommitManager   │      │
│  │(YAML)    │  │(AI 可插拔│  │ (commit/revert)  │      │
│  └────┬─────┘  └────┬─────┘  └────────┬─────────┘      │
│       └───────┬─────┘                 │                │
│               ▼                       │                │
│        输出 actions → uncommitted.json ┘                │
│                            ↓                            │
│                     用户 commit                         │
│                            ↓                            │
│                  commits/cm-xxx.json                    │
└──────────────────────────────────────────────────────────┘
```

**核心设计原则**：
1. Core 对 CLI/TUI 一无所知 —— Interface 只是薄壳
2. Reconciler 是命脉，每次 scan 必跑
3. RuleEngine 和 Classifier 都输出 actions 序列，进 `uncommitted.json`
4. 用户用 `commit` 固化未提交区，可随时 `revert`
5. SQLite 是主存储，sidecar 是导出格式

---

## 3. 数据模型

核心 3 张表（详细 schema 见 [02-scan.md](design/02-scan.md) 和 [03-tag.md](design/03-tag.md)）：

- **`file_record`**：`id` (UUID) + `content_hash` + `filename` + `current_path` + `status` + `pending_migration_from`
- **`tag`**：`id` + `name`（含 `/` 分隔的层级，如 `ai/ml`）+ `color` + `usage_count`
- **`file_tag`**：`file_id` + `tag_id` + `source` (`manual|rule|ai`) + `confidence`

关键：
- UUID 是文件身份，content_hash 是查找键（两个内容相同的文件各有独立标签）
- status 三态：`active` / `missing`（用户主动 `check --fix` 才清）/ `deleted`
- 不存 `name_hash`，filename 直接字符串 + 索引

---

## 4. 子命令索引

### Vault & 扫描
- [01-vault.md](design/01-vault.md) — `init` / `vault list` / `vault unlink` / Vault 定位
- [02-scan.md](design/02-scan.md) — `scan` + Reconciler + Hash 计算 + 性能预算

### 标签
- [03-tag.md](design/03-tag.md) — `tag` / `untag` / `tag tree` / `merge` / `rename` / `ls`

### 文件操作
- [04-file-ops.md](design/04-file-ops.md) — `mv` / `rm`（含 uncommitted.json 格式）
- [05-trash.md](design/05-trash.md) — `trash ls` / `restore` / `empty`

### 工作流（git-style）
- [06-status-diff.md](design/06-status-diff.md) — `status` / `diff`（含 action 类型表）
- [07-commit.md](design/07-commit.md) — `commit -m` / `commit ls` / `show` / `diff` / `revert` / `rm` / `--amend`

### 智能化
- [08-classify.md](design/08-classify.md) — `classify` + AI 后端抽象 + Prompt 模板
- [09-rule.md](design/09-rule.md) — `rule add` / `ls` / `run` + YAML 规则语法

### 维护
- [10-check.md](design/10-check.md) — `check` / `--fix` / `--duplicates` / `--report`（重复检测）
- [13-config.md](design/13-config.md) — `config get` / `set` / `edit` + 5 层优先级

### 配置
- [11-folder.md](design/11-folder.md) — `folder ls` / `set` / `unset` + role 解析

### 查询
- [12-query.md](design/12-query.md) — `ls` / `find` / `tree` / `export` / `import`

---

## 5. 命令总览

```bash
# Vault 管理
filetidy init [PATH]
filetidy vault list
filetidy vault unlink [PATH]

# 扫描
filetidy scan [--full] [--watch]

# 文件操作（直接生效，进 uncommitted）
filetidy mv <SRC> <DST>
filetidy rm <PATH>...                    # 到 trash
filetidy trash ls / restore <ID> / empty

# 标签（多级）
filetidy tag   <PATH|GLOB> <TAG...>      # 多级用 / 分隔，如 ai/ml
filetidy untag <PATH|GLOB> <TAG...>
filetidy tag tree / merge / rename / ls

# 查看当前整理
filetidy status                          # 未提交变化
filetidy diff [PATH]                     # 详细 diff

# 提交
filetidy commit -m "..."                 # 固化
filetidy commit --amend -m "..."         # 追加到最近
filetidy commit ls / show <ID> / diff <ID> / revert <ID> / rm <ID>

# 智能化
filetidy classify [PATH] [--backend ...]
filetidy rule add <FILE> / ls / run

# 维护
filetidy check [--fix] [--duplicates] [--report]

# 文件夹角色
filetidy folder ls / set <PATH> <ROLE> / unset <PATH>

# 查询
filetidy ls [--tag T] [--path P] [--status S] [--duplicate ...]
filetidy find <TAG>                      # 支持 ai/* 前缀
filetidy tree [PATH]
filetidy export [PATH...] -o FILE
filetidy import <FILE>

# 配置
filetidy config get [KEY] / set <KEY> <VALUE> [--global|--vault]
filetidy config edit / path
```

---

## 6. 关键设计决策汇总

| # | 决策 | 详见 |
|---|---|---|
| 1 | per-vault 标签库（不跨 vault 共享） | [03](design/03-tag.md) |
| 2 | 文件移出 vault → missing（不追踪到外） | [02](design/02-scan.md) |
| 3 | AI 后端：全局默认 + per-vault/per-folder 覆盖 | [13](design/13-config.md) / [11](design/11-folder.md) |
| 4 | 文件夹角色：inbox / tracked / ignored（archive 留 Phase 2） | [11](design/11-folder.md) |
| 5 | 子文件夹默认继承父角色（最少配置） | [11](design/11-folder.md) |
| 6 | 大文件采样 hash 默认开 | [02](design/02-scan.md) |
| 7 | EDIT 用 pending_migration 延迟处理 | [02](design/02-scan.md) |
| 8 | 多级标签：扁平存储 + `/` 分隔 + 前缀查询 | [03](design/03-tag.md) |
| 9 | 哈希策略：`content_hash` + `filename` 双索引（不存 name_hash） | [02](design/02-scan.md) |
| 10 | 重复检测**告知性而非强制性**（🟡/🔴 仅提醒） | [10](design/10-check.md) |
| 11 | git-style 工作流：mv/rm 直接生效 → status → commit | [04](design/04-file-ops.md) / [06](design/06-status-diff.md) / [07](design/07-commit.md) |
| 12 | 删除走垃圾桶（可恢复） | [05](design/05-trash.md) |
| 13 | `check` 单一命令整合体检/清理/重复检测 | [10](design/10-check.md) |
| 14 | commit revert 时反向执行 actions，含冲突检测 | [07](design/07-commit.md) |
| 15 | 自动清理：committed 30d / reverted 7d | [07](design/07-commit.md) |
| 16 | 命令前缀 `filetidy` + 别名 `fth` | - |
| 17 | 核心技术栈：Python 3.11+ Click + Rich + SQLite（Phase 2 加 Textual） | - |

---

## 7. `.filehelper/` 目录结构

```
<vault>/
└── .filehelper/
    ├── index.db              # SQLite 主存储
    ├── config.yaml           # vault 配置（[13-config.md](design/13-config.md)）
    ├── rules/*.yaml          # 用户规则（[09-rule.md](design/09-rule.md)）
    ├── cache/hash_cache.db   # hash 缓存
    ├── commits/              # commit 文件（[07-commit.md](design/07-commit.md)）
    │   └── cm-xxx.json
    ├── uncommitted.json      # 当前未提交的变化（[06-status-diff.md](design/06-status-diff.md)）
    ├── trash/                # 垃圾桶（[05-trash.md](design/05-trash.md)）
    │   └── tr-xxx/<file>
    └── logs/scan.log
```

---

## 8. 实施路线图

### Phase 1 — MVP 核心闭环

目标：vault 创建 → 扫描索引 → mv/rm 整理 → commit 固化 → revert 撤销。

**模块**：
- `filetidy.cli` — Click 入口
- `filetidy.vault` — init / locate / register
- `filetidy.db` — SQLite schema + 基础 CRUD
- `filetidy.scanner` + `filetidy.reconciler` — 扫描 + 4 情况
- `filetidy.fileops` — mv / rm（产生 actions）
- `filetidy.trash` — 垃圾桶管理
- `filetidy.tags` — TagStore
- `filetidy.commit` — CommitManager（commit/revert/amend）
- `filetidy.status` — 未提交区 + diff
- `filetidy.output` — Rich 渲染

**能跑的命令**：`init` / `scan` / `mv` / `rm` / `trash ls|restore|empty` / `tag` / `untag` / `ls` / `find` / `tree` / `status` / `diff` / `commit` / `commit ls|show|revert` / `check` 基础

### Phase 2 — 智能化

- `classify` + 3 个 AI 后端（[08](design/08-classify.md)）
- `rule` 规则引擎（[09](design/09-rule.md)）
- Textual TUI（Explorer 风：左 tree + 右详情）
- `archive` folder role
- 监听模式（watchdog）

### Phase 3 — 扩展

- 定时调度
- 跨机器同步（sidecar + git）
- Nuitka 打包单 exe 分发
- 更多 AI 后端（Anthropic / Qwen）

---

## 9. 待解决问题（汇总）

详见各子文档末尾的"待定问题"。核心：

- SQLite WAL vs journal 模式
- Windows 路径规范化（`/` vs `\`、长路径、大小写）
- glob 库选择（`wcmatch` vs `fnmatch`）
- 多终端同时操作 vault 的加锁策略
- AI rate limit / retry 策略
- commit amend 嵌套追溯的表示
- 跨盘移动的原子性

---

## 10. 开发顺序建议

按依赖关系，建议实施顺序：

1. **基础**：`vault`（01）→ `db` schema → `scanner` + `reconciler`（02）
2. **文件操作**：`fileops`（04）→ `trash`（05）
3. **工作流**：`status`/`diff`（06）→ `commit`（07）
4. **标签**：`tags`（03）→ `query`（12）
5. **维护**：`check`（10）→ `config`（13）→ `folder`（11）
6. **智能化**（Phase 2）：`classifier`（08）→ `rule`（09）
