# Block D · Tags（标签块）

> 主文档: [DESIGN.md](../DESIGN.md) · 协议: [actions.md](../core/actions.md) · 存储: [storage.md](../core/storage.md)

## 简介

提供**多级标签** + **快速查询**。`find` 走 SQLite 索引，O(log n) 而非全表扫，是 Harness 定位下的**查询加速器**。

---

## 内部组分

| 组分 | 职责 |
|---|---|
| `TagStore` | 多级标签 CRUD |
| `TagIndex` | 前缀查询索引（`ai/*` 等） |

---

## 对外命令

```bash
filetidy tag <PATH|GLOB> <TAG...>           # 加标签
filetidy untag <PATH|GLOB> <TAG...>         # 移除
filetidy tag tree                            # 树状视图
filetidy tag merge <SRC> <DST>              # 合并
filetidy tag rename <OLD> <NEW>             # 重命名
filetidy tag ls                              # 列出所有
filetidy find <TAG>                          # 按标签查（支持 ai/* 前缀）
```

---

## 对外 API

```python
class Tags:
    def tag(self, target: Path | FileId, tags: list[str], source: str = "manual") -> list[Action]
    def untag(self, target: Path | FileId, tags: list[str]) -> list[Action]
    def find(self, tag_query: str) -> list[FileRecord]    # 走索引
    def tree(self) -> TagTree
    def merge(self, src: str, dst: str) -> Action
    def rename(self, old: str, new: str) -> Action
    def list_tags(self) -> list[TagInfo]
```

---

## 多级标签

**存储扁平**：`tag.name` 直接存 `ai/ml/transformers`，不搞 `parent_id` 关联表。

**查询支持前缀**：

| 模式 | 含义 |
|---|---|
| `ai/ml` | 精确匹配 |
| `ai/*` | 前缀匹配（所有 `ai` 下的子标签） |
| `ai/ml/*` | 二级前缀 |

---

## TagStore.tag

```python
def tag(target: Path | FileId, tags: list[str], source: str = "manual") -> list[Action]:
    file_record = resolve_target(target)
    actions = []
    for tag_name in tags:
        # 自动创建不存在的 tag
        tag = db.get_or_create_tag(tag_name)

        # 已存在关联则跳过
        if db.has_file_tag(file_record.id, tag.id):
            continue

        db.add_file_tag(file_record.id, tag.id, source=source)
        actions.append(Action(
            type=ActionType.TAG_ADD,
            payload={"file": str(file_record.path), "tag": tag_name, "source": source},
            inverse=Action(type=ActionType.TAG_REMOVE, payload={"file": ..., "tag": tag_name})
        ))
    return actions
```

---

## Tags.find（核心：查询加速器）

```python
def find(tag_query: str) -> list[FileRecord]:
    """走索引，O(log n)"""
    if tag_query.endswith("/*"):
        # 前缀匹配: ai/* → name LIKE 'ai/%'
        prefix = tag_query[:-2]
        sql = """SELECT fr.* FROM file_record fr
                  JOIN file_tag ft ON fr.id = ft.file_id
                  JOIN tag t ON ft.tag_id = t.id
                  WHERE t.name LIKE ? || '/%' OR t.name = ?
                  GROUP BY fr.id"""
        return db.query(sql, prefix, prefix)
    else:
        # 精确匹配
        sql = """SELECT fr.* FROM file_record fr
                  JOIN file_tag ft ON fr.id = ft.file_id
                  JOIN tag t ON ft.tag_id = t.id
                  WHERE t.name = ?"""
        return db.query(sql, tag_query)
```

**为什么这是 Harness 的核心**：
- Agent 频繁调用 `find` 查找文件
- 没有 tag 索引：每次扫 `file_record` 全表
- 有 tag 索引：O(log n) 直接命中
- 大 vault（10k+ 文件）性能差几个数量级

---

## TagStore.merge

```python
def merge(src: str, dst: str) -> Action:
    """把 src 标签的所有文件合并到 dst"""
    # 1. 找所有有 src 标签的文件
    # 2. 给每个文件加 dst 标签
    # 3. 移除 src 标签关联
    # 4. 删除 src tag 行（如果用户确认）
    # 5. 产出 Action 序列（多个 TAG_ADD + TAG_REMOVE + tag 删除）
```

---

## TagStore.rename

支持多级重命名：

```bash
filetidy tag rename ai/ml ai/machine-learning
# 同时改: ai/ml/transformers → ai/machine-learning/transformers
#         ai/ml/pytorch       → ai/machine-learning/pytorch
```

实现：所有以 `ai/ml` 开头的 tag name 都前缀替换。

---

## 树状视图

```
$ filetidy tag tree
ai
├── ml
│   ├── transformers (3 files)
│   └── pytorch (5 files)
└── nlp
    ├── llm (2 files)
    └── translation (1 file)
finance
├── stocks (8 files)
└── crypto (3 files)
```

按 `/` 分隔渲染成树。

---

## Action 流转

```
Tags.tag() ──────┐
Tags.untag() ────┤
                 │
                 ▼
          Block C: History.record(action)
```

---

## 数据模型

详见 [storage.md](../core/storage.md) 的 `tag` 和 `file_tag` 表。

---

## 与其他块的关系

| 块 | 关系 |
|---|---|
| Block A (Index) | 关联 `file_record` |
| Block C (History) | **推送** Action |
| Block E (Rules) | **被调用**（rule 产出 TAG_ADD action 时） |

---

## 待定问题

- 标签最大长度？（建议 64 字符）
- 是否支持别名（`ai/ml` ≡ `machine-learning`）？YAGNI
- 标签颜色冲突解决？
- 大量标签时的 tree 渲染性能？
