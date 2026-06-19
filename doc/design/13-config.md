# 13 · 配置系统（config）

> 主文档: [DESIGN.md](../DESIGN.md) · 相关: [11-folder.md](11-folder.md)

## 简介

5 层优先级的配置加载，AI 后端 / 扫描设置 / folder role / 规则路径。

## 命令清单

| 命令 | 功能 |
|---|---|
| `filetidy config get [KEY]` | 读配置 |
| `filetidy config set <KEY> <VALUE>` | 写配置 |
| `filetidy config edit` | 用 $EDITOR 打开当前 vault config |
| `filetidy config path` | 显示配置文件路径 |

## 配置层级（高 → 低优先级）

1. **CLI flag**（`--backend ollama`）
2. **环境变量**（`FILEHELPER_BACKEND=ollama`）
3. **Per-folder override**（`config.yaml` 的 `folders:` 段，见 [11-folder.md](11-folder.md)）
4. **Per-vault 配置**（`<vault>/.filehelper/config.yaml`）
5. **全局默认**（`~/.filehelper/config.yaml`）

高优先级覆盖低优先级。`config get` 显示合并后的最终值。

## 详细设计

### `filetidy config get [KEY]`

**功能**：读配置。无 KEY 时打印全部。

**示例**：
```bash
$ filetidy config get backend.default
ollama

$ filetidy config get
backend.default: ollama
backend.ollama.model: llama3.1
backend.ollama.base_url: http://localhost:11434
scan.max_filesize_mb: 500
scan.sample_hash: true
scan.sample_threshold_mb: 100
folders.02-Projects.role: tracked
folders.01-Inbox-Work.backend: deepseek
...

$ filetidy config get backend --verbose   # 显示来源
backend.default = ollama
  source: <vault>/.filehelper/config.yaml (line 3)
backend.openai.model = gpt-4o-mini
  source: <vault>/.filehelper/config.yaml (line 7)
```

### `filetidy config set <KEY> <VALUE>`

**功能**：写配置。

**选项**：
- `--global`：写到全局 `~/.filehelper/config.yaml`
- `--vault`：写到当前 vault（默认）
- `--type <TYPE>`：值类型（`string|int|bool|float`）

**示例**：
```bash
$ filetidy config set backend.default deepseek
Updated: backend.default = deepseek (<vault>/.filehelper/config.yaml)

$ filetidy config set scan.sample_hash false --type bool
Updated: scan.sample_hash = false

$ filetidy config set backend.default ollama --global
Updated: backend.default = ollama (~/.filehelper/config.yaml)
```

### `filetidy config edit`

**功能**：用 `$EDITOR` 打开当前 vault 的 config.yaml。

### `filetidy config path`

**输出**：
```
Vault config:    E:/FileDownload/MyFiles/.filehelper/config.yaml
Global config:   C:/Users/xiaoy/.filehelper/config.yaml
```

## Vault config.yaml 完整示例

```yaml
# <vault>/.filehelper/config.yaml

# Vault 元信息
vault:
  name: MyFiles
  created_at: 2026-06-19

# AI 后端（vault 级默认）
backend:
  default: ollama
  ollama:
    model: llama3.1
    base_url: http://localhost:11434
    timeout_sec: 30
  openai:
    model: gpt-4o-mini
    api_key_env: OPENAI_API_KEY         # 从环境变量读，不写入文件
  deepseek:
    model: deepseek-chat
    api_key_env: DEEPSEEK_API_KEY

# 扫描设置
scan:
  skip_dirs: ['.git', 'node_modules', '__pycache__', '.venv']
  skip_files: ['*.tmp', '*.bak', '~$*']
  max_filesize_mb: 500                   # 超过跳过
  sample_hash: true                      # 大文件采样 hash
  sample_threshold_mb: 100

# 文件夹角色（见 11-folder.md）
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

# 规则路径（见 09-rule.md）
rules:
  - rules/default.yaml
  - rules/personal.yaml

# Commit / Trash 维护
commit:
  retention_committed_days: 30
  retention_reverted_days: 7

trash:
  max_size_gb: 10                        # 超过提示清理（不自动）
  warn_older_than_days: 30

# 输出偏好
output:
  color: auto                            # auto | always | never
  pager: less
  default_format: table                  # table | json | tree
```

## 全局 config.yaml 示例

```yaml
# ~/.filehelper/config.yaml
# 这些是默认值，可被 per-vault 覆盖

backend:
  default: ollama

scan:
  sample_hash: true

user:
  name: Terrence Chen
  email: terrence741@gmail.com
```

## 环境变量

| 变量 | 作用 |
|---|---|
| `FILEHELPER_VAULT` | 当前 vault 路径（见 [01-vault.md](01-vault.md)） |
| `FILEHELPER_BACKEND` | AI 后端 |
| `FILEHELPER_LOG_LEVEL` | 日志级别 |
| `OPENAI_API_KEY` | OpenAI 密钥（**不入文件**） |
| `DEEPSEEK_API_KEY` | DeepSeek 密钥（**不入文件**） |
| `EDITOR` | `config edit` 用的编辑器 |

## 依赖关系

- **依赖**：[01-vault.md](01-vault.md)
- **被依赖**：几乎所有模块（都从 config 读设置）

## 待定问题

- 配置 schema 验证（启动时检查 YAML 合法性）？
- 配置变更触发的副作用（改 backend 后是否需要重新连接）？
- 多用户共享一台机器时的全局配置冲突？
