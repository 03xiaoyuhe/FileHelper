# 11 · 文件夹角色（folder）

> 主文档: [DESIGN.md](../DESIGN.md) · 相关: [02-scan.md](02-scan.md)

## 简介

vault 内每个子文件夹一个**角色**，决定 scan / classify 行为。最少配置原则：根默认 `inbox`，子文件夹继承父角色，用户只在 `config.yaml` 声明例外。

## 命令清单

| 命令 | 功能 |
|---|---|
| `filetidy folder ls` | 列出所有文件夹角色 |
| `filetidy folder set <PATH> <ROLE>` | 设置文件夹角色 |
| `filetidy folder unset <PATH>` | 恢复默认（继承父级） |

## 角色定义

| 角色 | scan | classify | 典型场景 |
|---|---|---|---|
| `inbox` | ✓ | ✓ 自动 | 新文件落地、待整理 |
| `tracked` | ✓ | ✗ | 已整理但要索引 |
| `ignored` | ✗ | ✗ | 临时文件、缓存、`_trash` |

`archive`（已索引但不主动扫）Phase 2 再加。

## 默认策略

- vault 根 = `inbox`
- 子文件夹默认**继承父角色**
- 用户在 `config.yaml` 声明例外

## 详细设计

### `filetidy folder ls`

**输出**：
```
Path                          Role       Inherited From
.                             inbox      (root)
01-Inbox-Personal             inbox      root
01-Inbox-Work                 inbox      root (override: backend=deepseek)
02-Projects                   tracked    config
03-Reference                  tracked    config
04-Photos                     inbox      root
temp                          ignored    config
_trash                        ignored    config
node_modules                  ignored    config
```

支持 `--json`、`--role <ROLE>` 过滤。

### `filetidy folder set <PATH> <ROLE>`

**功能**：设置某文件夹的角色，写入 `config.yaml`。

**选项**：
- `--backend NAME`：同时设置 per-folder AI 后端覆盖
- `--recursive`：递归应用到所有子文件夹

**示例**：
```bash
$ filetidy folder set 04-Photos tracked
Updated: 04-Photos → tracked (config.yaml reloaded)

$ filetidy folder set 01-Inbox-Work inbox --backend deepseek
Updated: 01-Inbox-Work → inbox (backend: deepseek)

$ filetidy folder set _archive ignored --recursive
Updated: _archive and 12 subfolders → ignored
```

### `filetidy folder unset <PATH>`

**功能**：从 `config.yaml` 移除该文件夹的角色配置，恢复继承。

**示例**：
```bash
$ filetidy folder unset 04-Photos
Removed: 04-Photos (now inherits from root → inbox)
```

## Role 解析算法

```python
def get_role(path) -> FolderConfig:
    """对任意 vault 内路径，返回其 role 和 backend 配置"""
    rel = relative_to_vault(path)

    # 精确匹配
    if rel in config.folders:
        return config.folders[rel]

    # 父目录链向上找
    for parent in parents(rel):
        if parent in config.folders:
            return config.folders[parent].inherit_to_descendants()

    # 默认（vault 根）
    return FolderConfig(role='inbox', backend=config.backend.default)
```

## per-folder override

每个文件夹可以独立配置 AI 后端：

```yaml
folders:
  '01-Inbox-Personal':
    role: inbox
    backend: ollama               # 个人邮件用本地模型（隐私）
  '01-Inbox-Work':
    role: inbox
    backend: deepseek             # 工作文件用 DeepSeek（更强）
```

## config.yaml 片段

```yaml
folders:
  'temp': ignored
  '_trash': ignored
  'node_modules': ignored
  '02-Projects': tracked
  '03-Reference': tracked
  '01-Inbox-Personal':
    role: inbox
    backend: ollama
  '01-Inbox-Work':
    role: inbox
    backend: deepseek
  # 其他没声明的子文件夹 → 默认继承（根 = inbox）
```

## 依赖关系

- **依赖**：[13-config.md](13-config.md)（配置读写）、[01-vault.md](01-vault.md)
- **被依赖**：[02-scan.md](02-scan.md)（walk 时过滤）、[08-classify.md](08-classify.md)（只跑 inbox）

## 待定问题

- 文件夹被重命名后，`config.yaml` 里的路径如何同步？
- `archive` 角色（已索引但不主动扫）的语义细节？Phase 2
- 是否需要 `folder rename` 命令自动同步 config？
