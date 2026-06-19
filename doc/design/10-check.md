# 10 · 体检（check）

> 主文档: [DESIGN.md](../DESIGN.md)

## 简介

单一 `check` 命令整合所有维护操作：DB 健康、配置检查、孤儿标签、重复文件、过期 commit 清理。flag 控制范围。

## 命令清单

| 命令 | 功能 |
|---|---|
| `filetidy check` | 综合体检（默认全跑） |
| `filetidy check --fix` | 自动修复可修复项 |
| `filetidy check --duplicates` | 只查重复文件 |
| `filetidy check --report` | 只生成报告，不做任何动作 |

## 详细设计

### `filetidy check`

**功能**：综合体检 vault 健康。

**默认检查项**：
1. **DB 完整性**：SQLite 表结构、外键、索引
2. **配置有效性**：config.yaml 语法、AI 后端配置、规则文件
3. **孤儿标签**：`tag` 表中没有 `file_tag` 引用的标签
4. **孤儿 trash**：trash 里的文件不在任何 commit 中（孤立）
5. **重复文件**：检测 🟡/🔴 重复（见下）
6. **过期 commit**：超过保留期的 commit（见 [07-commit.md](07-commit.md)）
7. **未提交区健康**：uncommitted.json 中的 action 引用的文件是否还存在
8. **AI 后端连通性**：ping ollama / openai / deepseek

**输出**示例：
```
Vault: E:/FileDownload/MyFiles

✓ DB integrity:        OK
✓ Config:              OK
⚠ Orphan tags:         2 tags with no files
                         - inbox-temp
                         - test-1
⚠ Duplicate files:     3 items (1 red, 2 yellow)
                         See `filetidy check --duplicates`
⚠ Expired commits:     5 commits older than 30 days
                         Run `filetidy check --fix` to clean
✗ AI backend:          ollama unreachable (timeout)
✓ Trash:               450 MB / 12 items (oldest: 45 days)
✓ Uncommitted health:  OK

Issues: 3 warnings, 1 error
Run `filetidy check --fix` to address auto-fixable issues.
```

### `filetidy check --fix`

**功能**：自动修复可修复项。

**会修复**：
- 删除孤儿标签（`tag` 行）
- 清理过期 commit（删除 `commits/cm-xxx.json`）
- 删除孤儿 trash（不在任何 commit 中的 trash 项）

**不会修复**：
- 重复文件（告知性，不主动去重）
- AI 后端连通（需用户处理）
- DB 损坏（严重，提示用户手动）

### `filetidy check --duplicates`

**功能**：详细列出所有重复文件。

**输出**：
```
🔴 完全重复（同名同内容，不同路径）— 1 group:

  paper.pdf  (sha256:abc..., 2.3 MB)
    /01-Inbox/paper.pdf
    /02-Projects/ai/paper.pdf

🟡 内容重复（异名同内容）— 2 groups:

  sha256:def...  (5.1 MB)
    /03-Photos/IMG_001.jpg
    /04-Backup/vacation.jpg

  sha256:ghi...  (450 KB)
    /02-Projects/draft_v1.docx
    /02-Projects/draft_v2.docx

Note: duplicates are reported but NOT auto-removed.
Use `filetidy rm` to manually clean up if desired.
```

### `filetidy check --report`

**功能**：只生成报告，**不**做任何修复（即使 `--fix` 也忽略）。用于 CI / 审计场景。

## 重复检测算法

```python
def detect_duplicates():
    # 1. 按 content_hash 分组
    for content_hash, records in db.group_by_content_hash():
        if len(records) <= 1:
            continue

        # 2. 同内容组内按 filename 字符串分组
        name_groups = defaultdict(list)
        for r in records:
            name_groups[r.filename].append(r)

        # 3. 同名同内容（不同路径） → 🔴 完全重复
        for filename, same_name in name_groups.items():
            if len(same_name) > 1:
                yield Duplicate('red', content_hash, filename, same_name)

        # 4. 异名同内容 → 🟡 内容重复
        if len(name_groups) > 1:
            yield Duplicate('yellow', content_hash, records=records)
```

| 严重度 | 条件 | 含义 |
|---|---|---|
| 🔴 红 | `content_hash` 相同 **且** `filename` 相同 | 完全重复（备份/镜像） |
| 🟡 黄 | `content_hash` 相同 **但** `filename` 不同 | 内容重复（副本改名） |

**关键约束**：重复检测**告知性，不强制去重**。两条 file_record 各自独立、标签独立。文件系统允许的操作（备份、副本、镜像）FileHelper 全部支持。

## 依赖关系

- **依赖**：几乎所有模块（DB、config、commit、trash、scan 结果）
- **被依赖**：用户主动调用

## 待定问题

- check 的执行频率（自动 scan 后跑？还是只手动）？
- AI 后端连通性测试的 timeout（避免 check 卡住）？
- 是否需要 `check --auto-fix-monthly` 之类的定时？
