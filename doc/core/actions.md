# Action 协议

> 主文档: [DESIGN.md](../DESIGN.md)

## 简介

Action 是 FileHelper 的**核心数据单元**。所有产生文件系统变化的操作（mv/cp/rm/mkdir/touch/tag/untag）都通过 Action 表达。Action 是 5 大块之间通信的统一协议。

---

## 数据结构

### Python 实现

```python
from dataclasses import dataclass
from datetime import datetime
from enum import Enum

class ActionType(str, Enum):
    MOVE = "move"
    COPY = "copy"
    DELETE = "delete"
    RESTORE = "restore"           # 从 trash 恢复
    MKDIR = "mkdir"
    TOUCH = "touch"
    TAG_ADD = "tag_add"
    TAG_REMOVE = "tag_remove"
    PURGE = "purge"               # 彻底从 trash 清空（不可逆）

@dataclass
class Action:
    id: str                       # UUID
    type: ActionType
    timestamp: datetime
    payload: dict[str, Any]       # type 特定的字段
    inverse: Action | None        # 反向操作（None 表示不可逆，如 PURGE）
    source: str = "manual"        # manual | rule | revert
```

---

## 各类型 payload

### MOVE
```python
{
    "from": "01-Inbox/paper.pdf",
    "to": "02-Projects/ai/paper.pdf",
    "content_hash": "sha256:abc...",
    "is_rename_only": False       # 同目录改名 vs 跨目录移动
}
# inverse: MOVE (from=to, to=from)
```

### COPY
```python
{
    "from": "templates/doc.docx",
    "to": "01-Inbox/new.docx",
    "content_hash": "sha256:..."  # 复制后两个文件 hash 相同
}
# inverse: DELETE (target=to, trash_id=...)
```

### DELETE
```python
{
    "original_path": "01-Inbox/broken.pdf",
    "trash_id": "tr-20260619-001",
    "content_hash": "sha256:abc...",
    "size_bytes": 2345678
}
# inverse: RESTORE (trash_id=..., to=original_path)
```

### RESTORE
```python
{
    "trash_id": "tr-20260619-001",
    "to": "01-Inbox/broken.pdf"
}
# inverse: DELETE (original_path=to)
```

### MKDIR
```python
{"path": "04-Photos/2024/06/"}
# inverse: no-op（目录为空可删，非空报错；暂不实现 RMDIR）
```

### TOUCH
```python
{
    "path": "01-Inbox/note.md",
    "create_if_missing": True,
    "update_mtime": True
}
# inverse: 新建 → DELETE；仅更新 mtime → no-op
```

### TAG_ADD / TAG_REMOVE
```python
# TAG_ADD
{
    "file": "02-Projects/ai/paper.pdf",   # 路径或 file_id
    "tag": "ai/ml",
    "source": "manual",                    # manual | rule | ai
    "confidence": None
}
# inverse: TAG_REMOVE

# TAG_REMOVE
{"file": "...", "tag": "ai/ml"}
# inverse: TAG_ADD
```

### PURGE
```python
{"trash_ids": ["tr-20260619-001", "tr-20260619-002"]}
# inverse: None（不可逆）
```

---

## 序列化（JSON）

```json
{
  "id": "act-001",
  "type": "move",
  "timestamp": "2026-06-19T17:25:00",
  "payload": {
    "from": "01-Inbox/paper.pdf",
    "to": "02-Projects/ai/paper.pdf",
    "content_hash": "sha256:abc..."
  },
  "inverse": {
    "id": "act-001-inv",
    "type": "move",
    "payload": {
      "from": "02-Projects/ai/paper.pdf",
      "to": "01-Inbox/paper.pdf"
    }
  },
  "source": "manual"
}
```

---

## 不可逆操作

只有 `PURGE` 是真正不可逆的。其他操作的 inverse 都能算出。

---

## Action 流转

```
[Block B: Ops]    [Block D: Tags]    [Block E: Rules]
     │                  │                   │
     │ 产生 Action      │ 产生 Action       │ 产生 Action 序列
     └──────────────────┴───────────────────┘
                        │
                        ▼
                [Block C: History]
                 - 写入 uncommitted.json
                 - 用户 commit 后固化
                 - 用户 revert 时反向执行 inverse
```

---

## 反向执行（Revert）

`revert` 时按 inverse 反向执行：
- 调用对应 Block 的 API（B 或 D）
- 反向操作也产生新的 Action，写入新的 uncommitted
- 这样 revert 本身也是可 revert 的（嵌套）

---

## 待定问题

- Action ID 全局唯一还是 per-commit 唯一？
- 大量 Action 的存储优化（压缩 / 分页）？
- inverse 嵌套深度（revert 一个 revert 一个 revert...）？
