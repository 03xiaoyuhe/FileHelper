# 共享存储（SQLite Schema）

> 主文档: [DESIGN.md](../DESIGN.md)

## 简介

`.filehelper/index.db` 是 5 大块共享的 SQLite 数据库。每个块只读写自己负责的表。

---

## Schema 总览

```sql
-- Block A (Index) 负责
file_record           -- 文件记录（核心）
vault_meta            -- vault 元信息
schema_version        -- schema 版本管理

-- Block D (Tags) 负责
tag                   -- 标签定义
file_tag              -- 文件-标签多对多

-- Block C (History) 用 JSON 文件，不入 SQLite
-- Block B (Ops) 不存数据，只产生 Action
-- Block E (Rules) 不存数据，只读规则文件
```

---

## Block A · Index 表

### file_record

```sql
CREATE TABLE file_record (
    id              TEXT PRIMARY KEY,        -- UUID，每条记录独立
    content_hash    TEXT NOT NULL,           -- sha256(content)
    filename        TEXT NOT NULL,           -- 文件名（不含路径）
    current_path    TEXT NOT NULL,           -- vault-relative path
    size            INTEGER NOT NULL,
    mtime           INTEGER NOT NULL,        -- unix timestamp
    status          TEXT NOT NULL DEFAULT 'active',
                    -- active | missing | deleted | archived
    pending_migration_from TEXT,             -- EDIT 时指向旧 record id
    first_seen      TIMESTAMP NOT NULL,
    last_seen       TIMESTAMP NOT NULL
);

CREATE INDEX idx_file_content  ON file_record(content_hash);
CREATE INDEX idx_file_filename ON file_record(filename);
CREATE INDEX idx_file_identity ON file_record(content_hash, filename);
CREATE INDEX idx_file_path     ON file_record(current_path);
CREATE INDEX idx_file_status   ON file_record(status);

-- 注意：不约束 UNIQUE(content_hash, current_path)
-- 允许两个路径有同一身份（包容多实例）
```

### vault_meta

```sql
CREATE TABLE vault_meta (
    key   TEXT PRIMARY KEY,
    value TEXT NOT NULL
);
-- 例：vault_name, created_at, last_scan_at, file_count
```

### schema_version

```sql
CREATE TABLE schema_version (
    version    INTEGER PRIMARY KEY,
    applied_at TIMESTAMP NOT NULL
);
```

---

## Block D · Tags 表

### tag

```sql
CREATE TABLE tag (
    id          TEXT PRIMARY KEY,            -- UUID
    name        TEXT NOT NULL UNIQUE,        -- 含 / 分隔的层级
    color       TEXT,                        -- 可选，UI 用
    usage_count INTEGER NOT NULL DEFAULT 0,
    created_at  TIMESTAMP NOT NULL
);
CREATE INDEX idx_tag_name ON tag(name);
```

### file_tag

```sql
CREATE TABLE file_tag (
    file_id     TEXT NOT NULL REFERENCES file_record(id) ON DELETE CASCADE,
    tag_id      TEXT NOT NULL REFERENCES tag(id) ON DELETE CASCADE,
    source      TEXT NOT NULL,               -- manual | rule | ai
    confidence  REAL,                        -- AI 来源时 0-1
    added_at    TIMESTAMP NOT NULL,
    PRIMARY KEY (file_id, tag_id)
);
CREATE INDEX idx_file_tag_tag  ON file_tag(tag_id);    -- find 加速
CREATE INDEX idx_file_tag_file ON file_tag(file_id);
```

---

## Block C · History 存储（JSON 文件）

### uncommitted.json

```json
{
  "vault": "<vault_root>",
  "updated_at": "2026-06-19T17:30:00",
  "actions": [<Action>, <Action>, ...]
}
```

### commits/cm-xxx.json

每个 commit 一个文件，详见 [../blocks/c-history.md](../blocks/c-history.md)。

---

## 文件身份 vs 物理实例（关键设计）

**身份 = (content_hash, filename)**，**不是路径**。

例：
- `01-Inbox/paper.pdf` (hash=X) → file_record 1
- `02-Backup/paper.pdf` (hash=X) → file_record 2

两条记录的 `(content_hash, filename)` 相同，但 `id` 和 `current_path` 不同。**系统包容**这种情况，不强制合并。

DuplicateDetector 查询时识别"同身份多实例"，标记 🔴（完全重复）或 🟡（异名同内容）。

---

## 性能考量

- **SQLite WAL 模式**：高并发读写性能更好
- **`current_path` 索引**：加速 Reconciler 的 `find_by_path`
- **`content_hash` 索引**：加速 Reconciler 的 `find_by_hash` + DuplicateDetector
- **`file_tag.tag_id` 索引**：让 `find` 走索引，O(log n) 而非全表扫

---

## Schema 迁移

每次启动检查 `schema_version`，落后则跑迁移脚本。详见 [../blocks/a-index.md](../blocks/a-index.md) 的 `MigrationManager`。

---

## 待定问题

- 是否需要 FTS（全文搜索）？YAGNI
- WAL vs default journal？
- 大 vault（100k+ 文件）的索引大小？（估算：~10MB / 100k 文件）
