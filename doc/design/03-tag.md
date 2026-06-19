# 03 · 标签管理（Tag）

> 主文档: [DESIGN.md](../DESIGN.md) · 相关: [12-query.md](12-query.md)

## 简介

per-vault 独立的标签库。支持**多级标签**（`ai/ml/transformers`），扁平存储 + `/` 分隔 + 前缀查询。

## 命令清单

| 命令 | 功能 |
|---|---|
| `filetidy tag <PATH\|GLOB> <TAG...>` | 给文件加标签 |
| `filetidy untag <PATH\|GLOB> <TAG...>` | 移除标签 |
| `filetidy tag tree` | 标签层级树状视图 |
| `filetidy tag merge <SRC> <DST>` | 合并标签 |
| `filetidy tag rename <OLD> <NEW>` | 重命名标签 |
| `filetidy tag ls` | 列出所有标签 |

## 详细设计

### `filetidy tag <PATH|GLOB> <TAG...> [--source manual|rule|ai]`

**功能**：给文件加一个或多个标签。

**参数**：
- `PATH|GLOB`：文件路径或 glob 模式
- `TAG...`：标签名（多个，支持多级 `ai/ml`）

**选项**：
- `--source`：来源（默认 `manual`）
- `--confidence`：AI 来源时的置信度（0-1）
- `--dry-run`：预演

**行为**：
1. 解析 PATH/GLOB，匹配文件列表
2. 单文件：直接加标签 + 写 `uncommitted.json`
3. 批量（glob）：默认 `--dry-run`，需 `--apply` 确认
4. 标签不存在自动创建（写入 `tag` 表）
5. 已存在的标签关系：忽略（不报错）

**示例**：
```bash
$ filetidy tag 02-Projects/ai/paper.pdf ai/ml paper to-read
Tagged: 02-Projects/ai/paper.pdf
  + ai/ml
  + paper
  + to-read

$ filetidy tag "01-Inbox/*.pdf" inbox --apply
Tagged 12 files with 'inbox'
```

### `filetidy untag <PATH|GLOB> <TAG...>`

**功能**：移除文件的标签。

**行为**：删除 `file_tag` 行；标签本身保留（用 `tag rm` 才删标签）。

### `filetidy tag tree`

**功能**：树状展示所有标签。

**输出**：
```
ai
├── ml
│   ├── transformers
│   └── pytorch
└── nlp
    ├── llm
    └── translation
finance
├── stocks
└── crypto
photo
└── 2024
    └── 06
        └── travel
work
```

按命名空间（`/` 分隔）渲染成树，叶子节点后显示文件数 `(12)`。

### `filetidy tag merge <SRC> <DST>`

**功能**：把 SRC 标签的所有文件合并到 DST。

**示例**：
```bash
$ filetidy tag merge ai/ml/transformers ai/ml
Moved 8 files from 'ai/ml/transformers' to 'ai/ml'
Delete 'ai/ml/transformers'? [Y/n] y
```

### `filetidy tag rename <OLD> <NEW>`

**功能**：重命名标签（所有引用一并更新）。

支持批量重命名子标签：`tag rename ai/ml ai/machine-learning` 会同时改 `ai/ml/transformers` → `ai/machine-learning/transformers`。

### `filetidy tag ls`

**输出**：
```
Tag                          Files  Created
ai/ml                          12   2026-06-19
ai/ml/transformers              3   2026-06-19
photo                          45   2026-06-18
to-read                         8   2026-06-19
work                           23   2026-06-18
                                  ─────
                              88 tags
```

## 多级标签查询语法

`ls` / `find` / `tag ls` 等命令的 `--tag` 参数：

| 模式 | 含义 |
|---|---|
| `ai/ml` | 精确匹配 |
| `ai/*` | 前缀匹配（所有 `ai` 下的子标签） |
| `ai/ml/*` | 二级前缀 |
| `ai` | 精确匹配（不会自动包含子标签） |

## 数据模型

见 [DESIGN.md §4](../DESIGN.md#4-数据模型sqlite)：
- `tag` 表：`name`（含 `/` 分隔的路径）
- `file_tag` 表：多对多

不搞 `parent_id` 关联表，扁平存储 + 字符串解析。

## 依赖关系

- **依赖**：[01-vault.md](01-vault.md)、[02-scan.md](02-scan.md)（file_record）
- **被依赖**：[07-commit.md](07-commit.md)、[12-query.md](12-query.md)、[08-classify.md](08-classify.md)

## 待定问题

- 标签最大长度？建议 64 字符
- 是否支持标签别名（`ai/ml` ≡ `machine-learning`）？YAGNI
- 标签颜色冲突解决（多个命名空间同色）？
