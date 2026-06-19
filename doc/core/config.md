# 配置系统

> 主文档: [DESIGN.md](../DESIGN.md)

## 简介

5 层优先级配置加载，从 CLI flag 到全局默认。所有块共享同一套配置接口。

---

## 优先级（高 → 低）

1. **CLI flag**（`--backend ollama`，临时覆盖）
2. **环境变量**（`FILEHELPER_SCAN_FULL=true`）
3. **Per-folder override**（`config.yaml` 的 `folders:` 段）
4. **Per-vault 配置**（`<vault>/.filehelper/config.yaml`）
5. **全局默认**（`~/.filehelper/config.yaml`）

---

## 默认配置（global）

`~/.filehelper/config.yaml` 示例：

```yaml
# 全局默认，可被 per-vault 覆盖
scan:
  sample_hash: true
  sample_threshold_mb: 100
  max_filesize_mb: 500

commit:
  retention_committed_days: 30
  retention_reverted_days: 7

trash:
  max_size_gb: 10
  warn_older_than_days: 30

output:
  color: auto                # auto | always | never
  default_format: table      # table | json | tree
```

---

## Vault 配置（per-vault）

`<vault>/.filehelper/config.yaml` 示例：

```yaml
vault:
  name: MyFiles
  created_at: 2026-06-19

scan:
  skip_dirs: ['temp', '_trash']      # 追加到硬编码默认
  skip_files: ['*.tmp', '*.bak']

# 规则路径
rules:
  - rules/default.yaml
  - rules/personal.yaml
```

---

## 硬编码默认（不可配置）

扫描时永远跳过：

- `.filehelper/`（工具自身）
- `.git/`、`.hg/`、`.svn/`（版本控制）
- `node_modules/`、`__pycache__/`、`.venv/`、`venv/`（依赖/缓存）
- `~$*`（Office 临时文件）

用户**无法在 config 里关闭**这些硬编码跳过（确保安全）。但可以用 `filetidy scan --force-include <PATH>` 临时覆盖。

---

## 命令

```bash
filetidy config get [KEY]              # 读
filetidy config set <KEY> <VALUE>      # 写
filetidy config edit                   # 用 $EDITOR 打开
filetidy config path                   # 显示路径
```

`config get` 支持 `--verbose` 显示每个值的来源。

---

## 环境变量

| 变量 | 作用 |
|---|---|
| `FILEHELPER_VAULT` | 当前 vault 路径 |
| `FILEHELPER_LOG_LEVEL` | 日志级别 |
| `EDITOR` | `config edit` 用的编辑器 |

---

## 待定问题

- 配置 schema 严格验证还是宽松？
- 配置变更后的副作用（如改 `scan.skip_dirs` 后是否需要重扫）？
- 多用户共享机器的全局配置冲突？
