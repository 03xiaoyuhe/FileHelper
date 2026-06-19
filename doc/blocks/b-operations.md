# Block B · Operations（操作块）

> 主文档: [DESIGN.md](../DESIGN.md) · 协议: [actions.md](../core/actions.md)

## 简介

提供统一的文件操作原语：`mv` / `cp` / `rm` / `mkdir` / `touch`。每个操作**立即生效**（文件系统层面）+ 产出 `Action` 推送给 Block C。

---

## 内部组分

| 组分 | 职责 |
|---|---|
| `FileOps` | mv/cp/rm/mkdir/touch 统一接口 |
| `TrashManager` | 删除中转、restore、empty |

---

## 对外命令

```bash
filetidy mv   <SRC> <DST>           # 移动
filetidy cp   <SRC> <DST>           # 复制
filetidy rm   <PATH>...             # 删除到 trash
filetidy mkdir <PATH>               # 创建目录
filetidy touch <PATH>               # 创建空文件 / 更新 mtime
filetidy trash ls                   # 看垃圾桶
filetidy trash restore <ID>...      # 恢复
filetidy trash empty [--older-than DAYS]
```

---

## 对外 API

```python
class Operations:
    def mv(self, src: Path, dst: Path) -> Action
    def cp(self, src: Path, dst: Path) -> Action
    def rm(self, paths: list[Path]) -> list[Action]
    def mkdir(self, path: Path, parents: bool = True) -> Action
    def touch(self, path: Path, update_mtime: bool = True) -> Action

class TrashManager:
    def list(self) -> list[TrashItem]
    def restore(self, trash_ids: list[str], to: Path | None = None) -> list[Action]
    def empty(self, older_than_days: int | None = None) -> Action
```

---

## FileOps.mv

```python
def mv(src: Path, dst: Path) -> Action:
    # 1. 验证 src 在 vault 内（防误操作 vault 外）
    # 2. 验证 dst 父目录存在（不存在则报错，不自动 mkdir）
    # 3. 物理移动（shutil.move 处理跨盘）
    # 4. 产出 Action:
    return Action(
        type=ActionType.MOVE,
        payload={"from": str(src), "to": str(dst), "content_hash": "..."},
        inverse=Action(type=ActionType.MOVE, payload={"from": str(dst), "to": str(src)})
    )
    # 5. 推送给 Block C
```

**关键约束**：
- 不主动同步索引（下次 scan 时 Reconciler 发现 MOVE，标签自动迁移）
- glob 批量时返回 Action 列表

---

## FileOps.rm

```python
def rm(paths: list[Path]) -> list[Action]:
    actions = []
    for path in paths:
        trash_id = generate_trash_id()
        # 1. 创建 <vault>/.filehelper/trash/<trash_id>/
        # 2. 写 meta.json（原路径、删除时间、size、hash）
        # 3. 物理移动文件到 trash 目录
        # 4. 产出 Action:
        actions.append(Action(
            type=ActionType.DELETE,
            payload={"original_path": str(path), "trash_id": trash_id, "content_hash": "..."},
            inverse=Action(type=ActionType.RESTORE, payload={"trash_id": trash_id, "to": str(path)})
        ))
    return actions
```

---

## TrashManager

垃圾桶结构：

```
<vault>/.filehelper/trash/
├── tr-20260619-001/
│   ├── meta.json              # 原路径、size、hash、删除时间
│   └── <filename>             # 实际文件
├── tr-20260619-002/
└── ...
```

```python
def restore(trash_ids: list[str], to: Path | None = None) -> list[Action]:
    # 1. 读 meta.json 拿原路径
    # 2. 检查原路径冲突（占用则报错或用 to）
    # 3. 移回文件，删 trash 目录
    # 4. 产出 RESTORE Action

def empty(older_than_days: int | None = None) -> Action:
    # 物理删除（不可逆！）
    # 产出 PURGE Action（inverse=None）
```

---

## 关键约束

1. **所有操作必须在 vault 子树内**（防止误操作 vault 外）
2. **不主动同步索引**（mv 后下次 scan 自然发现；rm 后下次 scan 标 missing）
3. **rm 永远先到 trash**（除非 `--permanent`，需确认）
4. **每个操作产出 Action**（含 inverse，唯一例外是 PURGE）

---

## Action 流转

```
FileOps.mv() ──────┐
FileOps.rm() ──────┤
TrashManager.restore() ─┤
                   │
                   ▼
            Block C: History.record(action)
                   │
                   ▼
            写入 uncommitted.json
```

---

## glob 支持

所有命令支持 glob（基于 `wcmatch`）：

```bash
filetidy mv "01-Inbox/*.pdf" 02-Projects/    # 批量移动
filetidy rm "temp/**/*.tmp"                  # 批量删除
```

glob 模式匹配多个文件时返回 Action 列表。

---

## 待定问题

- 跨盘 mv 的原子性？`shutil.move` 跨盘会 copy + delete
- mv 目标已存在时的策略（覆盖 / 报错 / 重命名）？
- cp 是否产出 DELETE inverse（复制产生的新文件可删）？
- 大批量 glob 的进度反馈？
