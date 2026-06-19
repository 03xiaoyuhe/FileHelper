# 05 · 垃圾桶（Trash）

> 主文档: [DESIGN.md](../DESIGN.md) · 相关: [04-file-ops.md](04-file-ops.md)

## 简介

`filetidy rm` 删除的文件不会立刻消失，而是进入 `<vault>/.filehelper/trash/`，可以恢复或彻底清空。类比 macOS 废纸篓。

## 命令清单

| 命令 | 功能 |
|---|---|
| `filetidy trash ls` | 列出垃圾桶内容 |
| `filetidy trash restore <ID>` | 恢复某项到原路径 |
| `filetidy trash empty [--older-than DAYS]` | 清空垃圾桶 |

## 详细设计

### `filetidy trash ls`

**输出**：
```
Trash ID              Original Path                Size      Deleted At
tr-20260619-001       01-Inbox/broken.pdf          2.3 MB    2026-06-19 17:28
tr-20260619-002       01-Inbox/draft.docx          450 KB    2026-06-19 17:30
tr-20260619-003       04-Photos/blurry.jpg         5.1 MB    2026-06-19 17:32
                                                              ─────────────
                                            Total:  7.85 MB    3 items
```

支持 `--json`、`--filter "*.pdf"`、`--sort size|date|path`。

### `filetidy trash restore <ID>...`

**功能**：把 trash 里的文件移回原路径。

**选项**：
- `--to <PATH>`：恢复到其他路径（原路径已被占用时用）
- `--force`：覆盖现有文件

**行为**：
1. 读 `meta.json` 拿到原路径
2. 检查原路径是否存在：
   - 不存在 → 直接移回
   - 存在 → 报错，提示用 `--to` 或 `--force`
3. 物理移动文件回原路径
4. 删除 `<trash_id>/` 目录
5. 写入 `uncommitted.json`：`{type: "restore", trash_id, to, ts}`

**示例**：
```bash
$ filetidy trash restore tr-20260619-001
Restored: 01-Inbox/broken.pdf

$ filetidy trash restore tr-20260619-002 --to 02-Projects/draft_v2.docx
Restored: tr-20260619-002 → 02-Projects/draft_v2.docx
```

### `filetidy trash empty`

**选项**：
- `--older-than DAYS`：只清空 N 天前的（默认全清）
- `--yes`：跳过确认提示

**行为**：
1. 列出将被清空的项目，请求确认（除非 `--yes`）
2. 物理删除文件和 `<trash_id>/` 目录
3. 写入 `uncommitted.json`：`{type: "purge", trash_ids: [...], ts}`

## 垃圾桶目录结构

```
<vault>/.filehelper/trash/
├── tr-20260619-001/
│   ├── meta.json              # 原路径、大小、删除时间、来源命令
│   └── broken.pdf             # 实际文件
├── tr-20260619-002/
│   ├── meta.json
│   └── draft.docx
└── ...
```

### meta.json 格式

```json
{
  "trash_id": "tr-20260619-001",
  "original_path": "01-Inbox/broken.pdf",
  "filename": "broken.pdf",
  "size_bytes": 2345678,
  "content_hash": "sha256:abc...",
  "deleted_at": "2026-06-19T17:28:00",
  "deleted_by": "rm",          // rm / rule / classify
  "source_commit": null        // 如果是 commit revert 时被移入的，记 commit_id
}
```

## 自动清理策略

垃圾桶**不自动清理**（用户决定何时 empty）。但 `check` 会报告：
- "垃圾桶占用 1.2 GB，3 项超过 30 天，建议 empty --older-than 30"

## 与 Commit 的关系

- 删除操作进 `uncommitted.json` → `commit` 固化
- commit revert 时：如果是 delete 操作，把文件从 trash 恢复回原路径
- trash empty 后，相关的 commit revert 可能失败（资源不存在）→ 报错提示

## 依赖关系

- **依赖**：[04-file-ops.md](04-file-ops.md)（rm 产生 trash）、[01-vault.md](01-vault.md)
- **被依赖**：[07-commit.md](07-commit.md)（revert 时用）

## 待定问题

- 垃圾桶最大容量限制？（满了自动 LRU 清理？）
- 跨盘 trash 的存储策略？
