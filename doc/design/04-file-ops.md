# 04 · 文件操作（mv / rm）

> 主文档: [DESIGN.md](../DESIGN.md) · 相关: [05-trash.md](05-trash.md) · [06-status-diff.md](06-status-diff.md)

## 简介

直接对 vault 内文件执行移动和删除操作。操作立即生效（文件系统层面），并自动记录到 `uncommitted.json`，等待 `commit`（见 [07-commit.md](07-commit.md)）。

## 命令清单

| 命令 | 功能 |
|---|---|
| `filetidy mv <SRC> <DST>` | 移动文件 / 文件夹 |
| `filetidy rm <PATH>...` | 删除文件（到垃圾桶，不真删） |

## 详细设计

### `filetidy mv <SRC> <DST>`

**功能**：在 vault 内移动文件或文件夹。

**参数**：
- `SRC`：源路径（vault 相对）
- `DST`：目标路径，可以是目录或新文件名

**行为**：
1. 解析 SRC / DST 为 vault 内绝对路径
2. 验证 SRC 在 vault 子树内（跨 vault 报错）
3. 验证 DST 父目录存在（不存在则创建）
4. 执行物理移动（`os.rename` / `shutil.move`）
5. **不主动同步索引**（下次 `scan` 时 Reconciler 会发现 MOVE，标签自动迁移）
6. 写入 `uncommitted.json`：`{type: "move", from, to, ts}`
7. 输出："Moved: <SRC> → <DST>"

**示例**：
```bash
$ filetidy mv 01-Inbox/paper.pdf 02-Projects/ai/
Moved: 01-Inbox/paper.pdf → 02-Projects/ai/paper.pdf

$ filetidy mv 02-Projects/ai/paper.pdf 02-Projects/ai/paper_v2.pdf
Moved (rename): paper.pdf → paper_v2.pdf
```

**支持 glob**：
```bash
$ filetidy mv "01-Inbox/*.pdf" 02-Projects/
Moved: 12 files
```

### `filetidy rm <PATH>...`

**功能**：删除文件到垃圾桶（不真删）。

**参数**：
- `PATH`：要删除的文件（可多个，支持 glob）

**选项**：
- `--permanent`：跳过垃圾桶，直接物理删除（**危险**，需 `--yes` 二次确认）

**行为**：
1. 对每个 PATH：
   - 生成 `trash_id`（如 `tr-20260619-001`）
   - 创建 `<vault>/.filehelper/trash/<trash_id>/`
   - 写 `meta.json`：原始路径、删除时间、文件大小
   - 移动文件到该目录
2. 不动 `file_record`（数据库层面文件还"在"，下次 scan 时 Reconciler 标记 missing）
3. 写入 `uncommitted.json`：`{type: "delete", original_path, trash_id, ts}`
4. 输出："Deleted: <PATH> (trash: <trash_id>)"

**示例**：
```bash
$ filetidy rm 01-Inbox/broken.pdf
Deleted: 01-Inbox/broken.pdf (trash: tr-20260619-001)

$ filetidy rm "01-Inbox/*.tmp"
Deleted: 8 files (trash: tr-20260619-002 ~ tr-20260619-009)
```

## uncommitted.json 格式

```json
{
  "vault": "<vault_root>",
  "updated_at": "2026-06-19T17:30:00",
  "actions": [
    {
      "type": "move",
      "from": "01-Inbox/paper.pdf",
      "to": "02-Projects/ai/paper.pdf",
      "ts": "2026-06-19T17:25:00"
    },
    {
      "type": "delete",
      "original_path": "01-Inbox/broken.pdf",
      "trash_id": "tr-20260619-001",
      "ts": "2026-06-19T17:28:00"
    }
  ]
}
```

`filetidy status` 展示这个文件。`filetidy commit` 把它打包成 commit 并清空。

## 关键约束

1. **所有操作必须在 vault 子树内**（防止误操作 vault 外文件）
2. **不主动同步索引**（mv 后下次 scan 自然发现；rm 后下次 scan 标记 missing）—— 简化逻辑
3. **rm 永远先到垃圾桶**（除非 `--permanent`，给用户后悔机会）
4. **支持 glob 批量**（基于 `wcmatch`，支持 `**`）

## 依赖关系

- **依赖**：[01-vault.md](01-vault.md)（vault 上下文）
- **被依赖**：[05-trash.md](05-trash.md)、[06-status-diff.md](06-status-diff.md)、[07-commit.md](07-commit.md)

## 待定问题

- 跨盘移动（`D:` → `E:`）的原子性？`shutil.move` 处理跨盘会 copy + delete，速度慢
- mv 目标已存在时的策略（覆盖 / 报错 / 重命名）？
- 大量文件 glob 批量时的进度反馈？
