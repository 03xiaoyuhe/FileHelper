# 12 · 查询（ls / find / tree）

> 主文档: [DESIGN.md](../DESIGN.md) · 相关: [03-tag.md](03-tag.md)

## 简介

文件查询命令，把索引库里的数据用 Rich 渲染成 Explorer 风格。所有命令支持 `--json`（TUI/脚本复用）。

## 命令清单

| 命令 | 功能 |
|---|---|
| `filetidy ls [filters]` | 列出文件（默认表格视图） |
| `filetidy find <TAG>` | 按标签查文件 |
| `filetidy tree [PATH]` | 树状视图（Explorer 左侧导航风） |
| `filetidy export [PATH...] -o FILE` | 导出 sidecar |
| `filetidy import <FILE>` | 导入 sidecar |

## 详细设计

### `filetidy ls [filters]`

**功能**：按条件列出文件。

**过滤器**：
- `--tag <TAG>`：标签（支持前缀 `ai/*`，见 [03-tag.md](03-tag.md)）
- `--path <GLOB>`：路径 glob
- `--ext <EXT>`：扩展名（可多个）
- `--status <active|missing|deleted>`：状态（默认 `active`）
- `--size <RANGE>`：大小（如 `>10MB`）
- `--mtime <RANGE>`：修改时间（如 `>7d`）
- `--has-tag` / `--no-tag`：有/无标签
- `--duplicate <red|yellow|any>`：重复文件

**输出**（默认表格）：
```
Path                                Size   Tags              mtime              DUP
01-Inbox/paper.pdf                  2.3MB  ai/ml, paper      2026-06-19 17:25
02-Projects/ai/paper.pdf            2.3MB  ai/ml, paper      2026-06-19 17:25   🔴
03-Photos/IMG_001.jpg               5.1MB  photo, 2024/06    2026-06-18 09:00
04-Backup/vacation.jpg              5.1MB  photo             2026-06-18 09:00   🟡
                                                                              ─────
                                                                              4 files
```

**选项**：
- `--json`：机器可读
- `--limit N`：限制数量（默认 100）
- `--sort <field>`：排序（`path|size|mtime|tags`）
- `--no-dup`：隐藏 DUP 列
- `--long` / `-l`：详情模式（含 hash、first_seen 等）

### `filetidy find <TAG>`

**功能**：按标签查文件（`ls --tag TAG` 的快捷方式）。

**示例**：
```bash
$ filetidy find ai/ml
1. 02-Projects/ai/paper.pdf       [ai/ml, paper, to-read]
2. 02-Projects/ai/transformers.pdf [ai/ml, transformers]
3. 03-Reference/llm-survey.pdf    [ai/ml, paper, survey]

3 files tagged 'ai/ml'

$ filetidy find "ai/*"               # 前缀匹配
12 files tagged under 'ai/*'
...

$ filetidy find "paper to-read"      # AND 多标签
3 files tagged [paper AND to-read]
```

### `filetidy tree [PATH]`

**功能**：树状视图（Explorer 左侧导航风）。

**参数**：`PATH`（可选，默认 vault 根）

**选项**：
- `--depth N`：最大深度（默认 3）
- `--show-files`：显示文件（默认只目录）
- `--tag <TAG>`：高亮有该标签的文件

**输出**：
```
<vault>/
├── 01-Inbox/                            [inbox]
│   ├── paper.pdf                       [ai/ml 📄]
│   └── draft.docx
├── 02-Projects/                         [tracked]
│   ├── ai/
│   │   ├── paper.pdf                   [ai/ml 📄 🔴 dup]
│   │   └── transformers.pdf            [ai/ml]
│   └── finance/
├── 03-Photos/                           [inbox]
│   └── 2024/
│       └── 06/
└── _trash/                              [ignored, hidden]
```

### `filetidy export [PATH...] -o FILE`

**功能**：把索引数据导出为 sidecar JSON。

**用途**：跨 vault 迁移、git 同步、备份。

**示例**：
```bash
$ filetidy export 02-Projects/ -o projects.sidecar.json
Exported 45 files, 234 tags to projects.sidecar.json
```

**Sidecar 格式**：
```json
{
  "schema_version": 1,
  "exported_at": "2026-06-19T17:30:00",
  "vault": "<source vault>",
  "files": [
    {
      "content_hash": "sha256:abc...",
      "filename": "paper.pdf",
      "path": "02-Projects/ai/paper.pdf",
      "size": 2345678,
      "mtime": "2026-06-19T17:25:00",
      "tags": [
        {"name": "ai/ml", "source": "manual"},
        {"name": "paper", "source": "ai", "confidence": 0.92}
      ]
    }
  ]
}
```

### `filetidy import <FILE>`

**功能**：导入 sidecar，**只导入标签**（文件本身不会被创建）。

**行为**：
- 按 `content_hash` 匹配现有 file_record
- 找到 → 合并标签（已存在的跳过）
- 找不到 → 标记为 "pending"（下次 scan 时如果文件出现就关联）

**示例**：
```bash
$ filetidy import projects.sidecar.json
Importing 45 files, 234 tags...
  42 matched existing files (added 198 tags)
  3 pending (file not in vault, will associate on next scan)
```

## 输出渲染约定

- **颜色**：按标签命名空间着色（`ai/*` 一种色、`work/*` 另一种）
- **图标**：在 tree 视图中标记 dup（🔴/🟡）、tag（🏷）、tag 数
- **截断**：长路径用 `…` 中间省略
- **表头**：可点击（Phase 2 TUI 排序）

## 依赖关系

- **依赖**：[02-scan.md](02-scan.md)（file_record）、[03-tag.md](03-tag.md)（tag 查询）、[10-check.md](10-check.md)（dup 标记）
- **被依赖**：用户日常使用

## 待定问题

- 大量结果（>10k）时的分页策略？
- find 是否需要支持 OR 查询？
- export 是否需要加密 / 压缩选项？
