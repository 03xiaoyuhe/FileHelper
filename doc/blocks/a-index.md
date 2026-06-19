# Block A · Index（索引块）

> 主文档: [DESIGN.md](../DESIGN.md) · 协议: [actions.md](../core/actions.md) · 存储: [storage.md](../core/storage.md)

## 简介

知道仓库里**有什么文件**，每个文件的**状态**。是其他所有块的基础。

---

## 内部组分

| 组分 | 职责 |
|---|---|
| `VaultManager` | vault 生命周期（init/locate/unlink） |
| `Storage` | SQLite 包装 + schema 管理 |
| `Hasher` | content_hash 计算 + 缓存 |
| `Scanner` | Walk + 文件发现 |
| `Reconciler` | 比对算法（4 情况） |
| `DuplicateDetector` | 红黄检测 |
| `MigrationManager` | schema 版本管理 |

---

## 对外命令

| 命令 | 功能 |
|---|---|
| `filetidy init [PATH]` | 创建 vault + 全量扫描建基线 |
| `filetidy scan [<PATH>...]` | 扫描指定路径（默认全 vault） |
| `filetidy scan --full` | 强制全量重扫 |
| `filetidy ls [filters]` | 列文件（路径/状态/重复等过滤） |
| `filetidy tree [PATH]` | 树状视图 |
| `filetidy vault ls / unlink` | vault 管理 |
| `filetidy check [--fix] [--duplicates]` | 体检 |

---

## 对外 API

```python
class Index:
    def init_vault(self, path: Path) -> VaultInfo
    def locate_vault(self, start: Path | None = None) -> Path | None
    def scan(self, paths: list[Path] | None = None, full: bool = False) -> ScanReport
    def get(self, path: Path) -> FileRecord | None
    def find(self, query: Query) -> list[FileRecord]
    def list(self, filters: Filters) -> list[FileRecord]
    def detect_duplicates(self) -> list[Duplicate]
    def tree(self, path: Path | None = None) -> TreeReport
```

---

## Hasher

```python
def compute_content_hash(path: Path, stat: Stat) -> str:
    """SHA-256(content)"""
    key = (str(path), stat.st_mtime, stat.st_size)
    if key in hash_cache:
        return hash_cache[key]

    if stat.st_size > SAMPLE_THRESHOLD:
        h = sample_hash(path)        # 头+中+尾各 1MB
    else:
        h = full_hash(path)

    hash_cache[key] = h
    return h
```

**三层加速**：
1. `(path, mtime, size)` 缓存 → 不读文件
2. 大文件采样 hash（默认开，>100MB）
3. 并行 hash（`concurrent.futures`）

---

## Scanner

```python
def scan(paths: list[Path] | None, full: bool) -> ScanReport:
    # 1. Walk 文件树（受硬编码 + config skip 影响）
    # 2. 对每个文件计算 content_hash
    # 3. 调 Reconciler 比对
    # 4. 更新 last_seen
    # 5. 未扫到的 active → missing
    # 6. 汇总报告（新增 / 移动 / 编辑 / missing / 重复）
```

**关键**：scan 不自动跑（除 init 阶段）。用户/Agent 显式触发。

---

## Reconciler 算法

```python
def reconcile(path: Path, content_hash: str) -> FileRecord:
    R_path = db.find_by_path(path)
    R_content = db.find_by_content_hash(content_hash)

    if R_content and R_content.path == path:
        return R_content                                # ① 命中

    if R_content and R_content.path != path:
        R_content.path = path                           # ② MOVE / RENAME
        R_content.filename = path.name
        db.save(R_content)
        return R_content

    if R_path and R_path.content_hash != content_hash:
        new = create_record(path, content_hash)         # ③ EDIT
        new.pending_migration_from = R_path.id
        return new

    return create_record(path, content_hash)            # ④ NEW
```

**vault 边界**：文件移出 vault → 下次扫描未命中 → 标记 `missing`。

---

## DuplicateDetector

按用户的设计：**身份 = (content_hash, filename)**，**不是路径**。

```python
def detect_duplicates() -> list[Duplicate]:
    # 1. 按 content_hash 分组
    for content_hash, records in db.group_by_content_hash():
        if len(records) <= 1:
            continue

        # 2. 同 hash 组内按 filename 字符串分组
        name_groups = defaultdict(list)
        for r in records:
            name_groups[r.filename].append(r)

        # 3. 同名同内容 → 🔴 完全重复（同一身份的多实例）
        for filename, same_name in name_groups.items():
            if len(same_name) > 1:
                yield Duplicate('red', content_hash, filename, same_name)

        # 4. 异名同内容 → 🟡 内容重复
        if len(name_groups) > 1:
            yield Duplicate('yellow', content_hash, records=records)
```

| 严重度 | 条件 | 含义 |
|---|---|---|
| 🔴 红 | `content_hash` 相同 **且** `filename` 相同 | 完全重复（同身份多实例） |
| 🟡 黄 | `content_hash` 相同 **但** `filename` 不同 | 内容重复（异名副本） |

**关键约束**：
- **告知性而非强制性** —— 仅提醒，不主动去重
- **不合并记录** —— 每个路径独立 `file_record`（用户要求"包容"）

---

## 文件身份模型（核心）

| 字段 | 是身份？ | 含义 |
|---|---|---|
| `id` | 物理 | UUID，每条记录独立 |
| `(content_hash, filename)` | **逻辑身份** | 用户视角的"文件身份" |
| `current_path` | 位置 | 文件所在路径，可变 |

**包容多实例**：
- `(hash=X, name=A.pdf)` 在路径 P1 → file_record 1
- `(hash=X, name=A.pdf)` 在路径 P2 → file_record 2
- 两条记录独立存在，**不合并**
- 用户**无法分辨**（content + name 完全一样），知道"有两条、对得上"即可

---

## Walk 策略

**硬编码跳过**（不可配置）：
```
.filehelper/, .git/, .hg/, .svn/
node_modules/, __pycache__/, .venv/, venv/
~$*  (Office 临时)
```

**用户配置追加**（`config.yaml`）：
```yaml
scan:
  skip_dirs: ['temp', '_trash', 'build']
  skip_files: ['*.tmp', '*.bak']
```

**强制包含**（CLI override）：
```bash
filetidy scan --force-include node_modules/my-lib/
```

---

## 性能预算

| Vault 规模 | 首次全量 | 增量 |
|---|---|---|
| 1k 文件 | ~5s | <1s |
| 10k 文件 | ~30s | <3s |
| 100k 文件 | ~5min | <30s |

---

## 待定问题

- WAL 模式 vs default journal？
- Windows 路径规范化（`/` vs `\`、长路径、大小写）？
- glob 库选择（`wcmatch` vs `fnmatch`）？
- 大 vault 的 hash 缓存大小控制？
