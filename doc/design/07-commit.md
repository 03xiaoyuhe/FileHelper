# 07 · commit 命令组

> 主文档: [DESIGN.md](../DESIGN.md) · 相关: [06-status-diff.md](06-status-diff.md) · [04-file-ops.md](04-file-ops.md)

## 简介

把当前未提交的整理操作（`uncommitted.json`）固化为一个 commit。commit 是 FileHelper 的"事务单元"，类比 git commit。可以列出、查看、撤销。

## 命令清单

| 命令 | 功能 |
|---|---|
| `filetidy commit -m "..."` | 把未提交区固化为 commit |
| `filetidy commit ls` | 列出所有 commit |
| `filetidy commit show <ID>` | 看某 commit 详情 |
| `filetidy commit diff <ID>` | 看某 commit 的 diff |
| `filetidy commit revert <ID>` | 撤销某 commit（反向操作） |
| `filetidy commit rm <ID>` | 删除 commit 记录（仅 committed/reverted） |

## 详细设计

### `filetidy commit -m "MESSAGE"`

**功能**：固化未提交区。

**参数**：`-m MESSAGE`（必填，描述这次整理）

**选项**：
- `--amend`：不创建新 commit，把未提交区追加到最近 commit
- `--allow-empty`：允许空 commit（默认禁止）

**行为**：
1. 读 `uncommitted.json`
2. 若空且未 `--allow-empty` → 报错退出
3. 生成 commit_id（如 `cm-20260619-001`）
4. 写入 `<vault>/.filehelper/commits/cm-xxx.json`（见格式下文）
5. 清空 `uncommitted.json`
6. 输出："Commit cm-xxx created: <message>"

**示例**：
```bash
$ filetidy commit -m "整理 inbox 的 PDF"
Commit cm-20260619-001 created
  3 moves, 2 deletes, 5 tag additions
  Use `filetidy commit revert cm-20260619-001` to undo.

$ filetidy tag extra.pdf ai/ml
$ filetidy commit --amend -m "整理 inbox 的 PDF（含 extra）"
Amended: cm-20260619-001
  +1 tag addition
```

### `filetidy commit ls`

**输出**：
```
ID                  Created              Actions  Message
cm-20260619-003     2026-06-19 17:45         8   整理 photos
cm-20260619-002     2026-06-19 17:30        12   清理 downloads
cm-20260619-001     2026-06-19 17:00         5   整理 inbox 的 PDF
cm-20260618-001     2026-06-18 09:00         3   初始导入 [reverted]
```

**选项**：
- `--limit N`：只看最近 N 个（默认 20）
- `--author` / `--since` / `--until`：过滤（Phase 2）
- `--json`

### `filetidy commit show <ID>`

**输出**（一个 commit 的完整信息）：
```
Commit: cm-20260619-001
Created: 2026-06-19 17:00
Status: committed
Message: 整理 inbox 的 PDF
Actions: 5
  move:      3
  delete:    2 (in trash)
  tag_add:   0

Files affected:
  01-Inbox/paper.pdf       → 02-Projects/ai/paper.pdf
  01-Inbox/report.docx     → 02-Projects/work/report.docx
  01-Inbox/broken.pdf      → [trash: tr-20260619-001]
  01-Inbox/draft.tmp       → [trash: tr-20260619-002]

Revert: filetidy commit revert cm-20260619-001
```

### `filetidy commit diff <ID>`

**功能**：展示某 commit 的详细 diff（类似 `git show`）。

**输出**：和 `filetidy diff` 同格式，但限定在该 commit 的 actions。

### `filetidy commit revert <ID>`

**功能**：撤销某 commit，反向执行其所有 actions。

**选项**：
- `--force`：忽略冲突强推
- `--skip`：跳过冲突项继续
- `--dry-run`：预演不真做

**行为**：
1. 加载 commit 文件
2. **冲突检测**（见下）
3. 对每个 action 按 `inverse` 反向执行：
   - `move` → 反向 move（DST → SRC）
   - `delete` → 从 trash restore
   - `tag_add` → `tag_remove`
   - `tag_remove` → `tag_add`
4. 标记 commit `status=reverted`，写 `reverted_at`
5. 这些反向操作也写入新的 `uncommitted.json`（用户可以再 commit 这次的 revert）

**示例**：
```bash
$ filetidy commit revert cm-20260619-001
Reverting 5 actions...
  ✓ move: 02-Projects/ai/paper.pdf → 01-Inbox/paper.pdf
  ✓ move: 02-Projects/work/report.docx → 01-Inbox/report.docx
  ✓ restore: tr-20260619-001 → 01-Inbox/broken.pdf
  ✓ restore: tr-20260619-002 → 01-Inbox/draft.tmp

Reverted: cm-20260619-001
Changes are in uncommitted. Run `filetidy commit -m "..."` to固化.
```

### `filetidy commit rm <ID>`

**功能**：删除 commit 文件（不 revert，只是清理历史记录）。

**约束**：只允许删 `reverted` 或 7 天前的 `committed`。

## Commit 文件格式

存储位置：`<vault>/.filehelper/commits/cm-xxx.json`

```json
{
  "id": "cm-20260619-001",
  "message": "整理 inbox 的 PDF",
  "created_at": "2026-06-19T17:00:00",
  "committed_at": "2026-06-19T17:00:00",
  "amended_at": null,
  "reverted_at": null,
  "status": "committed",                  // committed | reverted
  "amend_of": null,                       // 如果是 amend，指向被追加的 commit_id
  "actions": [
    {
      "type": "move",
      "from": "01-Inbox/paper.pdf",
      "to": "02-Projects/ai/paper.pdf",
      "content_hash": "sha256:abc...",
      "inverse": {"type": "move", "from": "02-Projects/ai/paper.pdf", "to": "01-Inbox/paper.pdf"}
    },
    {
      "type": "delete",
      "original_path": "01-Inbox/broken.pdf",
      "trash_id": "tr-20260619-001",
      "inverse": {"type": "restore", "trash_id": "tr-20260619-001", "to": "01-Inbox/broken.pdf"}
    },
    {
      "type": "tag_add",
      "file": "02-Projects/ai/paper.pdf",
      "tag": "ai/ml",
      "source": "manual",
      "inverse": {"type": "tag_remove", "file": "02-Projects/ai/paper.pdf", "tag": "ai/ml"}
    }
  ]
}
```

## 冲突检测

`revert` 前对每个 action 检查：

| Action 类型 | 冲突条件 | 处理 |
|---|---|---|
| `move` 反向 | 反向目标路径已有其他文件 | 报错（`--force` 覆盖 / `--skip` 跳过） |
| `move` 反向 | 原文件 content_hash 变了 | 报错（用户改过，不自动覆盖） |
| `delete` 反向（restore） | trash_id 已被 empty 清掉 | 报错（资源没了，无法恢复） |
| `delete` 反向（restore） | 原路径已被占用 | 报错（`--to` 或 `--force`） |
| `tag_add` 反向 | 文件已不在 vault 内 | 警告（跳过，记入日志） |
| `tag_remove` 反向 | 标签已被其他 commit 改过 | 报错 |

## 自动清理

| status | 保留期 |
|---|---|
| `committed` | 30 天 |
| `reverted` | 7 天（审计用） |

`filetidy check --fix` 主动清理过期。

## 与其他命令的关系

| 命令 | 与 commit 的关系 |
|---|---|
| `mv` / `rm` / `tag` / `untag` | 产生 action → 进 uncommitted |
| `status` / `diff` | 展示 uncommitted |
| `trash restore` | 也写 uncommitted |
| `classify` / `rule run` | 直接产生 actions 序列 → 进 uncommitted（不立即 commit） |
| `check` | 体检 commit 文件健康，清理过期 |

## 依赖关系

- **依赖**：[06-status-diff.md](06-status-diff.md)（uncommitted 来源）、[05-trash.md](05-trash.md)（revert 时用）、[04-file-ops.md](04-file-ops.md)
- **被依赖**：所有需要历史回溯的场景

## 待定问题

- commit amend 多次嵌套时如何追溯（链表？只记 amend_of 一次）？
- 跨机器同步 commit（git 化）的冲突解决？
- 大量小 commit 时是否需要 `commit squash`（合并）？
