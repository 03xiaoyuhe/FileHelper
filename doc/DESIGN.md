# FileHelper 设计文档

> **主索引文档**。详细设计在子文档中。
> Status: Draft · Last updated: 2026-06-19

---

## 0. TL;DR

FileHelper 是一个 **vault-scoped** 的本地文件整理工具，定位为**智能体工作基座（Agent Harness）**。

**核心定位**：
- 工具提供**可靠的文件操作原语**（mv/cp/rm/mkdir/touch）+ **变更管理**（commit/revert）+ **快速查询**（find）
- AI Agent（Claude Code、Cursor、Cline 等）**外部决策**整理逻辑，调用 CLI 执行
- 工具保证操作安全（trash 可恢复、commit 可 revert、所有操作可逆除 PURGE）

**灵感**：
- **Obsidian** vault 模型
- **Eagle** 标签系统
- **Git** 工作流（mv → status → commit → revert）
- **Unix core utils** 文件操作风格

**技术栈**：Python 3.11+ · Click · Rich · SQLite

---

## 1. 核心概念

| 概念 | 说明 |
|---|---|
| **Vault** | 被管理的文件夹，根有 `.filehelper/` |
| **File Identity** | `(content_hash, filename)`，**不是路径** |
| **Action** | 所有操作的统一数据单元（含 inverse） |
| **Commit** | 一次整理的固化（git-style） |
| **Trash** | 删除回收站（可恢复） |

---

## 2. 系统架构：5 大块平行设计

```
                       ┌──────────────────────────┐
                       │  CLI / Config (粘合层)   │
                       └────────────┬─────────────┘
                                    │ 分发命令
        ┌──────────────┬────────────┼────────────┬──────────────┐
        │              │            │            │              │
        ▼              ▼            ▼            ▼              ▼
  ┌──────────┐   ┌──────────┐ ┌──────────┐ ┌──────────┐  ┌──────────┐
  │  A.      │   │  B.      │ │  C.      │ │  D.      │  │  E.      │
  │  Index   │   │  Ops     │ │  History │ │  Tags    │  │  Rules   │
  │          │   │          │ │          │ │          │  │          │
  │ 知道有   │   │ 安全改   │ │ 记录+    │ │ 加速     │  │ 模板化   │
  │ 什么     │   │  文件    │ │ 回滚     │ │ 查询     │  │ 整理     │
  └──────────┘   └──────────┘ └──────────┘ └──────────┘  └──────────┘
```

### 5 大块速览

| 块 | 解决的问题 | 详见 |
|---|---|---|
| **A · Index** | 知道仓库里有什么文件、每个文件的状态 | [blocks/a-index.md](blocks/a-index.md) |
| **B · Operations** | 安全地改文件（mv/cp/rm/mkdir/touch），每个操作可逆 | [blocks/b-operations.md](blocks/b-operations.md) |
| **C · History** | 记录每次整理、可回滚、查看 diff | [blocks/c-history.md](blocks/c-history.md) |
| **D · Tags** | 多级标签 + 快速 find（O(log n) 走索引） | [blocks/d-tags.md](blocks/d-tags.md) |
| **E · Rules** | 模板化的批量整理规则（YAML 声明式，无 AI） | [blocks/e-rules.md](blocks/e-rules.md) |

### 跨块协议

| 协议 | 作用 | 详见 |
|---|---|---|
| **Action 协议** | 5 大块通信的统一数据单元 | [core/actions.md](core/actions.md) |
| **共享 SQLite** | `file_record` / `tag` / `file_tag` | [core/storage.md](core/storage.md) |
| **配置系统** | 3 层优先级（CLI flag > per-vault > global） | [core/config.md](core/config.md) |

---

## 3. 块间依赖与并行开发

```
A (Index) ────────┐
                  ├──► D (Tags) ─────────┐
                  │                       │
B (Ops) ──────────┤                       ├──► E (Rules)
                  │                       │     (上层)
C (History) ◄─────┴──── Action 协议 ──────┘
                  (4 块都通过它通信)
```

**关键路径**：
1. **Action 协议**（首先定下来，4 块都依赖）
2. **`file_record` schema**（A 定，D 和 E 间接依赖）
3. A、B、C **完全并行**实现（互不阻塞）
4. D 等 A 的 schema 定下来后并行
5. E 等 B 和 D 的 API 定下来后实现

---

## 4. 命令总览

```bash
# Vault 管理（Block A）
filetidy init [PATH]
filetidy scan [<PATH>...] [--full]
filetidy ls [filters]
filetidy tree [PATH]
filetidy vault ls / unlink
filetidy check [--fix] [--duplicates]

# 文件操作（Block B）
filetidy mv / cp / rm / mkdir / touch
filetidy trash ls / restore / empty

# 变更管理（Block C)
filetidy status
filetidy diff [PATH]
filetidy commit -m "..." [--amend]
filetidy commit ls / show / diff / revert / rm

# 标签（Block D）
filetidy tag / untag
filetidy tag tree / merge / rename / ls
filetidy find <TAG>                      # 支持 ai/* 前缀

# 规则（Block E）
filetidy rule add / ls / run / validate / rm

# 配置（粘合层）
filetidy config get / set / edit / path
```

---

## 5. 文件身份模型（核心）

**身份 = `(content_hash, filename)`**，**不是路径**。

- 同名同内容（不同路径）= 同一身份的"多实例"
- 系统**包容**：每个路径独立 `file_record`，**不合并**
- 用户**无法分辨**（分辨不出来），知道"有两条、对得上"即可
- `DuplicateDetector` 标记 🔴/🟡 仅作展示，不影响数据

详见 [blocks/a-index.md](blocks/a-index.md) §"文件身份模型"。

---

## 6. `.filehelper/` 目录结构

```
<vault>/
└── .filehelper/
    ├── index.db              # SQLite（Block A/D 共享）
    ├── config.yaml           # vault 配置
    ├── rules/*.yaml          # 用户规则（Block E）
    ├── cache/hash_cache.db   # hash 缓存（Block A）
    ├── commits/              # commit 历史（Block C）
    │   └── cm-xxx.json
    ├── uncommitted.json      # 未提交操作（Block C）
    ├── trash/                # 垃圾桶（Block B）
    │   └── tr-xxx/<file>
    └── logs/
```

---

## 7. 关键设计决策汇总

| # | 决策 | 详见 |
|---|---|---|
| 1 | 定位为 Agent Harness（不内置 AI） | §10 |
| 2 | 5 大块平行设计（A/B/C/D/E） | §2 |
| 3 | per-vault 独立（不跨 vault） | [a-index.md](blocks/a-index.md) |
| 4 | 文件身份 = `(content_hash, filename)`，不是路径 | §5 |
| 5 | 同身份多实例**包容**（不合并） | [a-index.md](blocks/a-index.md) |
| 6 | 重复检测**告知性而非强制性**（🟡/🔴 仅展示） | [a-index.md](blocks/a-index.md) |
| 7 | git-style 工作流：mv → status → commit → revert | [c-history.md](blocks/c-history.md) |
| 8 | 删除走 trash（可恢复），PURGE 才真删 | [b-operations.md](blocks/b-operations.md) |
| 9 | Action 协议（含 inverse）是 5 块通信基础 | [actions.md](core/actions.md) |
| 10 | 多级标签扁平存储 + 前缀查询 | [d-tags.md](blocks/d-tags.md) |
| 11 | `find` 走索引（O(log n)），是查询加速器 | [d-tags.md](blocks/d-tags.md) |
| 12 | 规则块只调度（调 B 和 D），不直接改文件 | [e-rules.md](blocks/e-rules.md) |
| 13 | 大文件采样 hash 默认开 | [a-index.md](blocks/a-index.md) |
| 14 | EDIT 用 `pending_migration` 延迟处理 | [a-index.md](blocks/a-index.md) |
| 15 | commit amend 链表追溯 | [c-history.md](blocks/c-history.md) |
| 16 | 自动清理：committed 30d / reverted 7d | [c-history.md](blocks/c-history.md) |
| 17 | 命令前缀 `filetidy` + 别名 `fth` | - |

---

## 8. 实施路线图

### Phase 1 — MVP（5 大块完整闭环）

**目标**：vault 创建 → 扫描索引 → mv/rm 整理 → commit 固化 → revert 撤销 → find 查询 → rule 批量

**模块顺序**（按依赖）：
1. `core/actions` — Action 协议（首先定）
2. `core/storage` — SQLite schema（其次）
3. Block A — Index（基础）
4. Block B — Operations（并行）
5. Block C — History（等 Action 协议）
6. Block D — Tags（等 A schema）
7. Block E — Rules（等 B 和 D API）
8. CLI + Output（粘合层）

### Phase 2 — 扩展

- `ScanScheduler`（cron-like 定期扫描）
- 监听模式（watchdog）
- 文本 TUI（Textual，Explorer 风）
- 跨机器同步（sidecar + git）

### Phase 3 — 分发

- Nuitka 打包单 exe
- 更多 CLI 别名 / shell completion
- 性能优化（大 vault 支持）

---

## 9. 文档结构

```
doc/
├── DESIGN.md                    # 本文件（主索引）
├── core/
│   ├── actions.md               # Action 协议
│   ├── storage.md               # SQLite schema
│   └── config.md                # 配置系统
└── blocks/
    ├── a-index.md               # Block A 详细设计
    ├── b-operations.md          # Block B 详细设计
    ├── c-history.md             # Block C 详细设计
    ├── d-tags.md                # Block D 详细设计
    └── e-rules.md               # Block E 详细设计
```

---

## 10. 作为 Agent Harness 使用

FileHelper 是 **Agent Harness** —— 工具可靠，Agent 智能。

### 工作模式

```bash
# Agent 1. 看仓库当前状态
filetidy ls --status missing
filetidy find "to-process"

# Agent 2. 决策并执行整理（每个操作可逆）
filetidy mv 01-Inbox/paper.pdf 02-Projects/ai/
filetidy tag 02-Projects/ai/paper.pdf ai/ml paper

# Agent 3. 提交
filetidy commit -m "Agent: triaged inbox"

# Agent 4. 出错时回滚
filetidy commit revert HEAD
```

### 为什么不内置 AI

- **职责单一** —— 工具只做安全可撤销的文件操作
- **AI 可替换** —— 任何 Agent 都能用（Claude Code / Cursor / Cline / 自研）
- **不被规则限制** —— Agent 决策比 YAML 灵活
- **代码量减半** —— 无 prompt 工程 / API client / rate limit

---

## 11. 待解决问题（汇总）

详见各子文档末尾的"待定问题"。核心：

- SQLite WAL vs journal
- Windows 路径规范化
- glob 库选择
- 多终端并发加锁
- commit amend 嵌套追溯
- 跨盘 mv 原子性
- 大 vault（100k+）性能
