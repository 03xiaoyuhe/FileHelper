# 06 · status / diff

> 主文档: [DESIGN.md](../DESIGN.md) · 相关: [07-commit.md](07-commit.md) · [04-file-ops.md](04-file-ops.md)

## 简介

查看当前未提交的整理操作。`status` 给概览，`diff` 给详细对比。类比 `git status` / `git diff`。

## 命令清单

| 命令 | 功能 |
|---|---|
| `filetidy status` | 未提交变化的汇总 |
| `filetidy diff [PATH]` | 详细 diff（before/after 对比） |

## 详细设计

### `filetidy status`

**功能**：列出 `uncommitted.json` 中的所有 action，按类型分组。

**输出**（默认）：
```
On vault: E:/FileDownload/MyFiles
Last commit: cm-abc123 (2 hours ago)

Changes to be committed:
  moved:    3 files
  deleted:  2 files (in trash)
  tagged:   5 files (8 tag additions)
  untagged: 1 file

Run `filetidy commit -m "..."` to固化 these changes.
Run `filetidy diff` for details.
```

**输出**（`--verbose`）：
```
On vault: E:/FileDownload/MyFiles
Last commit: cm-abc123 (2 hours ago)

Moved (3):
  01-Inbox/paper.pdf           →  02-Projects/ai/paper.pdf
  01-Inbox/report.docx         →  02-Projects/work/report.docx
  03-Photos/IMG_001.jpg        →  04-Photos/2024/06/IMG_001.jpg

Deleted (2):
  01-Inbox/broken.pdf          [trash: tr-20260619-001]
  01-Inbox/draft.tmp           [trash: tr-20260619-002]

Tagged (5):
  + ai/ml           → 02-Projects/ai/paper.pdf
  + work, urgent    → 02-Projects/work/report.docx
  + photo, 2024/06  → 04-Photos/2024/06/IMG_001.jpg
  ...

Untagged (1):
  - old-tag         → 02-Projects/work/report.docx
```

**选项**：
- `--verbose` / `-v`：展开每个文件
- `--json`：机器可读
- `--short` / `-s`：极简模式（只数字）

### `filetidy diff [PATH]`

**功能**：详细 diff，展示具体变化。

**参数**：`PATH`（可选，只看某个文件/目录的变化）

**输出**示例：
```diff
diff - 02-Projects/ai/paper.pdf
+++ 02-Projects/ai/paper.pdf
moved from: 01-Inbox/paper.pdf
content_hash: sha256:abc... (unchanged)
tags:
  + ai/ml
  + paper
  + to-read
size: 2.3 MB
mtime: 2026-06-19 17:25:00

diff - 01-Inbox/broken.pdf
deleted (trash: tr-20260619-001)
  was: 2.3 MB, tags: [inbox, broken]

diff - 01-Inbox/draft.tmp
deleted (trash: tr-20260619-002)
  was: 12 KB, no tags
```

**选项**：
- `--stat`：只显示统计（不展开 diff）
- `--json`：机器可读
- `-- <PATH>`：限定路径

## uncommitted.json 结构

存储位置：`<vault>/.filehelper/uncommitted.json`

```json
{
  "vault": "<vault_root>",
  "updated_at": "2026-06-19T17:30:00",
  "actions": [
    {
      "type": "move",                    // move | delete | restore | tag_add | tag_remove | rename
      "from": "01-Inbox/paper.pdf",
      "to": "02-Projects/ai/paper.pdf",
      "content_hash": "sha256:abc...",
      "ts": "2026-06-19T17:25:00"
    },
    {
      "type": "delete",
      "original_path": "01-Inbox/broken.pdf",
      "trash_id": "tr-20260619-001",
      "ts": "2026-06-19T17:28:00"
    },
    {
      "type": "tag_add",
      "file": "02-Projects/ai/paper.pdf",
      "tag": "ai/ml",
      "source": "manual",
      "ts": "2026-06-19T17:29:00"
    }
  ]
}
```

## Action 类型汇总

| type | 来源命令 | inverse（用于 revert） |
|---|---|---|
| `move` | `mv` | 反向 move |
| `rename` | `mv`（同目录改名） | 反向 rename |
| `delete` | `rm` | restore from trash |
| `restore` | `trash restore` | delete again |
| `tag_add` | `tag` | `tag_remove` |
| `tag_remove` | `untag` | `tag_add` |

## 依赖关系

- **依赖**：所有产生 action 的命令（mv/rm/tag/...）
- **被依赖**：[07-commit.md](07-commit.md)（commit 把 uncommitted 打包）

## 待定问题

- 大量 uncommitted action 时（>1000）的渲染性能？
- 是否需要 `filetidy reset` 清空 uncommitted？（YAGNI，先不做）
