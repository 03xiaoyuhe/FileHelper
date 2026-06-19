# 01 · Vault 管理

> 主文档: [DESIGN.md](../DESIGN.md)

## 简介

Vault 是 FileHelper 管理的基本单元 —— 一个被工具索引的文件夹。类比 Obsidian vault 或 git repo。

## 命令清单

| 命令 | 功能 |
|---|---|
| `filetidy init [PATH]` | 在 PATH 创建 `.filehelper/`，注册到全局 |
| `filetidy vault list` | 列出所有已注册 vault |
| `filetidy vault unlink [PATH]` | 解除注册（不删 `.filehelper/`） |

## 详细设计

### `filetidy init [PATH]`

**功能**：把一个文件夹变成 vault。

**参数**：`PATH`（默认当前目录）

**行为**：
1. 检查 PATH 是否已是 vault（有 `.filehelper/`）→ 是则报错退出
2. 创建 `.filehelper/` 子目录（见下）
3. 写入默认 `config.yaml`
4. 初始化 SQLite `index.db`（建表）
5. 创建空 `uncommitted.json`
6. 注册到全局 `~/.filehelper/vaults.json`
7. 输出："Vault initialized. Run `filetidy scan` to index files."

### `filetidy vault list`

**输出**：
```
ID    Path                                Files  Last Scan
1     E:/FileDownload/MyFiles             1,234  2026-06-19 17:30
2     D:/Projects/MyVault                   452  2026-06-18 09:12
```

支持 `--json`。

### `filetidy vault unlink [PATH]`

**功能**：从全局注册表移除 vault，**不删除** `.filehelper/` 目录。

**用途**：用户想"忘记"某个 vault 但保留本地数据。

## Vault 定位（git-style）

每次命令执行前需确定当前 vault，按优先级：

1. `--vault PATH` 显式指定
2. `FILEHELPER_VAULT` 环境变量
3. 从 CWD 向上查找最近的 `.filehelper/`
4. 都找不到 → 报错退出

全局注册表 `~/.filehelper/vaults.json` **仅用于** `vault list` 展示，不参与运行时决策。

## `.filehelper/` 目录结构

```
<vault>/
└── .filehelper/
    ├── index.db              # SQLite 主存储
    ├── config.yaml           # vault 配置（见 13-config.md）
    ├── rules/*.yaml          # 用户规则（见 09-rule.md）
    ├── cache/hash_cache.db   # hash 缓存（见 02-scan.md）
    ├── commits/              # commit 文件（见 07-commit.md）
    │   └── cm-abc123.json
    ├── uncommitted.json      # 当前未提交的变化（见 06-status-diff.md）
    ├── trash/                # 垃圾桶（见 05-trash.md）
    │   └── tr-xxx/<file>
    └── logs/scan.log
```

## 依赖关系

- **依赖**：无（最基础模块）
- **被依赖**：所有其他子命令（执行前先定位 vault）

## 待定问题

- Windows 长路径（`\\?\` 前缀）支持？
- Vault 整体移动后（用户把 vault 目录迁到别处）如何更新注册表？
