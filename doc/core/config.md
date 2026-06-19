# 配置系统

> 主文档: [DESIGN.md](../DESIGN.md)

## 简介

FileHelper 的配置分**两层**（全局 + per-vault），加上**命令行临时覆盖**。所有块共享同一套配置接口。

---

## 优先级（高 → 低）

1. **CLI flag**（命令行参数，临时覆盖。例：`filetidy scan --full`）
2. **Per-vault 配置**（`<vault>/.filehelper/config.yaml`，只对当前 vault 生效）
3. **全局默认**（`~/.filehelper/config.yaml`，所有 vault 共享）

高优先级覆盖低优先级。`config get --verbose` 可以查看每个值的来源。

---

## 全局默认配置（`~/.filehelper/config.yaml`）

每个配置项都加了**详细注释**，第一次看应该能看懂：

```yaml
# =============================================================================
# FileHelper 全局默认配置
# 路径: ~/.filehelper/config.yaml
# 作用: 所有 vault 共享的默认值。每个 vault 可以在自己的
#       <vault>/.filehelper/config.yaml 里覆盖这里的任何一项。
# =============================================================================

# -----------------------------------------------------------------------------
# scan - 扫描相关（控制 filetidy scan 的行为）
# -----------------------------------------------------------------------------
scan:
  # 是否对"大文件"启用采样哈希。
  # 大文件（如 1GB 视频）如果完整算 SHA-256 会很慢（几秒到几十秒）。
  # 启用采样后，只读文件的头 1MB + 中间 1MB + 尾部 1MB 来算哈希，
  # 速度快 100 倍，碰撞概率极低（实际使用几乎不会重复）。
  # 设为 false 则对所有文件都算完整哈希（更严谨但慢）。
  sample_hash: true

  # 触发采样的"大文件"阈值，单位：兆字节（MB）。
  # 文件大小 > 此值时启用采样哈希；<= 此值则全量哈希。
  # 例: 100 → 100MB 以上的文件采样，100MB 以下全量。
  sample_threshold_mb: 100

  # 单个文件大小上限，单位：兆字节（MB）。
  # 超过此大小的文件**直接跳过**，不索引、不算哈希。
  # 用途: 避免一个 10GB 的视频把整个扫描卡住几分钟。
  # 如果你想索引大视频/备份文件，调大这个值或设为 null（不限制）。
  max_filesize_mb: 500

# -----------------------------------------------------------------------------
# commit - 历史记录保留策略（控制文件历史占用的磁盘空间）
# -----------------------------------------------------------------------------
commit:
  # 已固化（已 commit）的历史记录保留多少天。
  # 超过此天数的 commit 会被 `filetidy check --fix` 自动清理。
  # 用途: 防止历史无限增长。30 天足够回溯大部分操作。
  # 如果你想永久保留所有历史，设为 null（不自动清理）。
  retention_committed_days: 30

  # 已撤销（已 revert）的 commit 保留多少天。
  # reverted 状态的 commit 已经"无效"，保留只是为了审计（查"曾经撤销过什么"）。
  # 7 天后自动清理，足够长以备事后检查。
  retention_reverted_days: 7

# -----------------------------------------------------------------------------
# output - 输出样式（控制命令行输出的呈现方式）
# -----------------------------------------------------------------------------
output:
  # 是否启用彩色输出。
  # auto:   自动检测（在终端里启用颜色，重定向到文件时不启用）—— 推荐
  # always: 永远启用（即使重定向到文件也带颜色代码，可能混乱）
  # never:  永不启用（适合 CI 日志或老终端）
  color: auto

  # 默认输出格式。
  # table: 人类友好表格（默认，类似 ls -l）
  # json:  机器可读 JSON（脚本/Agent 用）
  # tree:  树状视图（类似 Explorer 左侧导航）
  # 命令行可以用 --format <name> 临时覆盖。
  default_format: table
```

---

## Vault 配置（`<vault>/.filehelper/config.yaml`）

只对当前 vault 生效，会覆盖全局默认。示例：

```yaml
# =============================================================================
# FileHelper Vault 配置
# 路径: <vault>/.filehelper/config.yaml
# 作用: 只对当前 vault 生效，会覆盖全局默认配置。
# =============================================================================

# vault 元信息（init 时自动写入，一般不用手动改）
vault:
  name: MyFiles                    # vault 的显示名
  created_at: 2026-06-19           # 创建日期

# -----------------------------------------------------------------------------
# scan - 这个 vault 特有的扫描设置（覆盖全局）
# -----------------------------------------------------------------------------
scan:
  # 额外跳过的目录名（在硬编码默认基础上**追加**，不是替换）。
  # 例: 这个 vault 里有 temp/ 和 _trash/ 子目录，不想扫。
  # 注意: 不能关闭硬编码默认（.git/node_modules 等永远跳过，见下文）。
  skip_dirs: ['temp', '_trash', 'build']

  # 额外跳过的文件名 glob。
  # 例: 所有 .tmp 和 .bak 文件不索引。
  skip_files: ['*.tmp', '*.bak']

# -----------------------------------------------------------------------------
# rules - 这个 vault 使用的规则文件（相对 vault 根的路径）
# 规则文件本身是 YAML，详见 doc/blocks/e-rules.md
# -----------------------------------------------------------------------------
rules:
  - rules/default.yaml             # 默认规则
  - rules/personal.yaml            # 个人定制规则
```

---

## 硬编码默认（**不可配置**）

扫描时永远跳过以下目录/文件，**用户无法在 config 里关闭**（确保安全）：

- `.filehelper/`（工具自身的数据目录）
- `.git/`、`.hg/`、`.svn/`（版本控制元数据）
- `node_modules/`、`__pycache__/`、`.venv/`、`venv/`（依赖/Python 缓存）
- `~$*`（Office 临时文件，如 `~$report.docx`）

如果真的需要扫某个被硬编码跳过的目录（罕见），用命令行覆盖：

```bash
filetidy scan --force-include node_modules/my-lib/
```

---

## 命令

```bash
filetidy config get [KEY]              # 读配置项（不带 KEY 则读全部）
filetidy config set <KEY> <VALUE>      # 写配置项
filetidy config edit                   # 用 $EDITOR 打开当前 vault 的 config.yaml
filetidy config path                   # 显示配置文件路径
```

**示例**：

```bash
# 读所有配置
filetidy config get

# 读单项
filetidy config get scan.sample_hash
# → true

# 读单项 + 来源
filetidy config get scan.sample_hash --verbose
# → scan.sample_hash = true
#   source: ~/.filehelper/config.yaml (line 12)

# 改配置（写到当前 vault）
filetidy config set scan.max_filesize_mb 1000

# 改全局默认（影响所有 vault）
filetidy config set output.color never --global
```

---

## Vault 定位（无需环境变量）

git 风格的 vault 查找，**没有 `FILEHELPER_VAULT` 环境变量**：

1. `--vault PATH` 命令行显式指定
2. 从 CWD（当前工作目录）向上找最近的 `.filehelper/`
3. 都找不到 → 报错退出

例：
```bash
cd /E/FileDownload/MyFiles/01-Inbox/
filetidy scan           # 自动找到 /E/FileDownload/MyFiles/.filehelper/

filetidy scan --vault D:/Other/Vault    # 显式指定别的 vault
```

---

## 环境变量（仅这 2 个）

| 变量 | 作用 |
|---|---|
| `FILEHELPER_LOG_LEVEL` | 日志级别（`DEBUG`/`INFO`/`WARNING`/`ERROR`） |
| `EDITOR` | `filetidy config edit` 用的编辑器（默认 `vi`/`notepad`） |

**没有 vault 路径环境变量** —— 用 `--vault` 或 CWD 推断（见上）。

---

## 待定问题

- 配置 schema 严格验证还是宽松？（建议严格 + 提示错误位置）
- 配置变更后的副作用（如改 `scan.skip_dirs` 后是否需要重扫）？
- 多用户共享机器的全局配置冲突？
