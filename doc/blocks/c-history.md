# Block C · History（历史块）

> 主文档: [DESIGN.md](../DESIGN.md) · 协议: [actions.md](../core/actions.md)

## 简介

记录所有文件操作的**历史**，提供 `commit`（固化）/ `revert`（回滚）/ `status` / `diff` 能力。是 5 大块中**唯一接收所有 Action 的块**。

---

## 内部组分

| 组分 | 职责 |
|---|---|
| `ActionLog` | 未提交操作序列（`uncommitted.json`） |
| `CommitManager` | 固化 / 历史 / `revert` / `amend` |
| `DiffEngine` | 状态对比（before/after） |

---

## 对外命令

```bash
filetidy status                     # 未提交概览
filetidy diff [PATH]                # 详细 diff
filetidy commit -m "..."            # 固化
filetidy commit --amend -m "..."    # 追加到最近 commit
filetidy commit ls                  # 历史
filetidy commit show <ID>           # 详情
filetidy commit diff <ID>           # commit 内 diff
filetidy commit revert <ID>         # 撤销
filetidy commit rm <ID>             # 删除（仅 reverted/老 committed）
```

---

## 对外 API

```python
class History:
    def record(self, action: Action) -> None              # Block B/D/E 调用
    def status(self, verbose: bool = False) -> StatusReport
    def diff(self, path: Path | None = None) -> Diff
    def commit(self, message: str, amend: bool = False) -> CommitId
    def list_commits(self, limit: int = 20) -> list[CommitSummary]
    def show_commit(self, commit_id: CommitId) -> CommitDetail
    def revert(self, commit_id: CommitId, force: bool = False) -> list[Action]
    def remove_commit(self, commit_id: CommitId) -> None
```

---

## ActionLog（uncommitted 区）

存储位置：`<vault>/.filehelper/uncommitted.json`

```json
{
  "vault": "<vault_root>",
  "updated_at": "2026-06-19T17:30:00",
  "actions": [<Action>, <Action>, ...]
}
```

- **`record(action)`**：追加 action 到 `actions` 列表，更新 `updated_at`
- **`status()`**：按 Action 类型分组统计 + 详细列表（verbose 模式）

---

## CommitManager

每个 commit 一个文件：`<vault>/.filehelper/commits/cm-<timestamp>-<seq>.json`

```json
{
  "id": "cm-20260619-001",
  "message": "整理 inbox 的 PDF",
  "created_at": "2026-06-19T17:00:00",
  "committed_at": "2026-06-19T17:00:00",
  "amended_at": null,
  "reverted_at": null,
  "status": "committed",
  "amend_of": null,
  "actions": [<Action>, ...]
}
```

### commit(message, amend=False)

```python
def commit(message: str, amend: bool = False) -> CommitId:
    uncommitted = read_uncommitted()
    if not uncommitted.actions:
        raise EmptyCommitError()

    if amend:
        # 找最近 commit，追加 actions
        last = get_latest_commit()
        last.actions.extend(uncommitted.actions)
        last.amended_at = now()
        save_commit(last)
    else:
        # 创建新 commit
        commit_id = generate_commit_id()
        commit = Commit(id=commit_id, message=message, actions=uncommitted.actions)
        save_commit(commit)

    # 清空 uncommitted
    write_uncommitted(actions=[])
    return commit_id
```

### revert(commit_id, force=False)

```python
def revert(commit_id: CommitId, force: bool = False) -> list[Action]:
    commit = load_commit(commit_id)

    # 冲突检测
    conflicts = detect_conflicts(commit)
    if conflicts and not force:
        raise ConflictError(conflicts)

    # 反向执行 inverses
    revert_actions = []
    for action in reversed(commit.actions):
        if action.inverse is None:
            continue   # 不可逆（如 PURGE）

        # 按 inverse.type 调对应 Block
        new_action = dispatch_inverse(action.inverse)
        revert_actions.append(new_action)

    # 标记原 commit 为 reverted
    commit.reverted_at = now()
    commit.status = "reverted"
    save_commit(commit)

    return revert_actions   # 这些反向操作也写入新的 uncommitted
```

### 冲突检测

| Action 反向 | 冲突条件 | 处理 |
|---|---|---|
| MOVE | 反向目标路径已有其他文件 | 报错（`force=覆盖`） |
| MOVE | 原文件 content_hash 变了 | 报错 |
| DELETE → RESTORE | trash_id 已被 empty 清掉 | 报错（资源没了） |
| DELETE → RESTORE | 原路径已被占用 | 报错 |
| TAG_ADD → TAG_REMOVE | 文件已不在 vault 内 | 警告（跳过） |

---

## DiffEngine

对比两个状态，生成 diff 报告。

```python
def diff(path: Path | None) -> Diff:
    """对比当前文件系统 vs 上次 commit 后的状态"""
    # 读 uncommitted.json 的 actions
    # 按文件聚合变化
    # 生成 diff（before/after）

def diff_commit(commit_id: CommitId) -> Diff:
    """展示某 commit 的 diff（类似 git show）"""
```

输出示例：

```diff
diff - 02-Projects/ai/paper.pdf
+++ 02-Projects/ai/paper.pdf
moved from: 01-Inbox/paper.pdf
content_hash: sha256:abc... (unchanged)
tags:
  + ai/ml
  + paper
size: 2.3 MB

diff - 01-Inbox/broken.pdf
deleted (trash: tr-20260619-001)
  was: 2.3 MB, tags: [inbox]
```

---

## 自动清理

| status | 保留期 |
|---|---|
| `committed` | 30 天（可配置） |
| `reverted` | 7 天 |

`filetidy check --fix` 主动清理过期。

---

## amend 链表

`amend_of` 字段形成链表：

```
cm-001 (original)
  ↑ amended by cm-002 (amend_of=cm-001)
    ↑ amended by cm-003 (amend_of=cm-002)
```

查询时只展示最新版本，但保留历史链表用于审计。

---

## 与其他块的关系

| 块 | 关系 |
|---|---|
| Block B (Ops) | **接收** Action（mv/rm 等） |
| Block D (Tags) | **接收** Action（tag_add/tag_remove） |
| Block E (Rules) | **接收** Action 序列（一次 rule run 多个 Action） |
| Block A (Index) | revert 时通过 B 改文件 → 下次 scan 更新索引 |

---

## 待定问题

- amend 嵌套深度限制？
- 跨机器同步 commit（git push/pull）的冲突解决？
- 大量小 commit 是否需要 squash？
- 冲突检测的精确度（content_hash 变 vs path 变）？
