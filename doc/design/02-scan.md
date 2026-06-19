# 02 · 扫描与 Reconciler

> 主文档: [DESIGN.md](../DESIGN.md) · 相关: [11-folder.md](11-folder.md)

## 简介

扫描 vault 内的文件，建立索引（content_hash + filename + path），并跑 Reconciler 检测 MOVE / RENAME / EDIT / NEW。

## 命令清单

| 命令 | 功能 |
|---|---|
| `filetidy scan [--full] [--watch]` | 扫描 + Reconciler，默认增量 |

## 详细设计

### `filetidy scan`

**参数**：
- `--full`：强制全量扫描（忽略 hash 缓存）
- `--watch`：监听模式（Phase 2，watchdog）
- `--path <PATH>`：只扫某个子目录

**行为**：
1. Walk vault 子树（受 folder role 影响，见 §Walk 策略）
2. 对每个文件计算 `content_hash`
3. 跑 Reconciler（4 情况，见下）
4. 更新 `last_seen` 时间戳
5. 未扫到的 `active` 文件 → 标记 `missing`
6. 结尾汇总：新增 / 移动 / 编辑 / missing 各多少
7. 报告重复文件（🟡/🔴，见 [10-check.md](10-check.md)）

**输出**（Rich Progress + 表格）：
```
Scanning... ━━━━━━━━━━━━━━━━━━━━ 100% 0:00:05

Summary:
  New:      12
  Moved:     3 (tags preserved)
  Edited:    1 (pending migration, run `check`)
  Missing:   2

Duplicates: 1 yellow, 0 red
```

## Walk 策略

**默认跳过**：`.filehelper`、`.git`、`node_modules`、`__pycache__`、`.venv`、Office 临时文件（`~$*`）。

**Folder role 影响**：
- `inbox` / `tracked` → 扫
- `ignored` → 跳过

可在 `config.yaml` 追加：
```yaml
scan:
  skip_dirs: ['.git', 'node_modules']
  skip_files: ['*.tmp', '*.bak']
  max_filesize_mb: 500
```

## Hash 计算

每个文件计算一个哈希：`content_hash = SHA-256(内容)`。

**三层加速**：
1. **mtime+size 快速路径**：缓存键 `(path, mtime, size)` 命中直接复用，不读文件
2. **大文件采样 hash**（默认开）：>100MB 文件只 hash 头+中+尾各 1MB
3. **并行 hash**：`concurrent.futures` 多线程

```python
def compute_content_hash(path, stat):
    key = (str(path), stat.st_mtime, stat.st_size)
    if key in hash_cache:
        return hash_cache[key]
    h = sample_hash(path) if stat.st_size > SAMPLE_THRESHOLD else full_hash(path)
    hash_cache[key] = h
    return h
```

**文件名**：直接存 `filename` 字段（不哈希），用于同名检测。

## Reconciler 算法

```python
def reconcile(path, content_hash):
    R_path = db.find_by_path(path)
    R_content = db.find_by_content_hash(content_hash)

    if R_content and R_content.path == path:
        return R_content                                # ① 命中

    if R_content and R_content.path != path:
        R_content.path = path                           # ② MOVE / RENAME
        R_content.filename = path.name
        db.save(R_content)
        log.info(f"MOVE/RENAME: tags preserved")
        return R_content

    if R_path and R_path.content_hash != content_hash:
        new = create_record(path, content_hash)         # ③ EDIT
        new.pending_migration_from = R_path.id
        return new

    return create_record(path, content_hash)            # ④ NEW
```

**vault 边界**：文件被移出 vault 子树 → 下次扫描时未命中 → 标记 `missing`。

## 数据模型

见 [DESIGN.md §4](../DESIGN.md#4-数据模型sqlite) 的 `file_record` 表。

关键字段：`content_hash`、`filename`、`current_path`、`status`、`pending_migration_from`。

## 性能预算

| Vault 规模 | 首次全量 | 增量 |
|---|---|---|
| 1k 文件 | ~5s | <1s |
| 10k 文件 | ~30s | <3s |
| 100k 文件 | ~5min | <30s |

## 依赖关系

- **依赖**：[01-vault.md](01-vault.md)（vault 上下文）、[11-folder.md](11-folder.md)（folder role 过滤）
- **被依赖**：所有需要 `file_record` 的命令（tag/query/classify/check）

## 待定问题

- SQLite WAL vs journal 模式？
- Windows 路径规范化（`/` vs `\`、大小写）？
- glob 库选择（Python `glob` vs `wcmatch`）？
