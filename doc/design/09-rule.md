# 09 · 规则引擎（rule）

> 主文档: [DESIGN.md](../DESIGN.md) · 相关: [08-classify.md](08-classify.md)

## 简介

声明式 YAML 规则，匹配文件并输出 actions。和 AI 分类器协作：规则提供结构，AI 填充语义。**输出 actions 序列**，进 `uncommitted.json`。

## 命令清单

| 命令 | 功能 |
|---|---|
| `filetidy rule add <FILE>` | 注册一个规则文件 |
| `filetidy rule ls` | 列出所有规则 |
| `filetidy rule run [--name N]` | 执行规则 |
| `filetidy rule validate <FILE>` | 验证规则语法 |

## 详细设计

### `filetidy rule add <FILE>`

**功能**：把 YAML 规则文件复制到 `<vault>/.filehelper/rules/`，写入 config 的 `rules:` 列表。

### `filetidy rule ls`

**输出**：
```
Name                        Source                    Actions  Classify
pdf-by-content              rules/pdf.yaml                  0    yes
image-by-exif               rules/image.yaml                2    no
code-project                rules/code.yaml                 2    no
```

### `filetidy rule run [--name N] [--dry-run|--apply]`

**功能**：执行规则。

**参数**：`--name N`（可选，只跑指定规则）

**选项**：
- `--dry-run`：预演（默认）
- `--apply`：写入 uncommitted

**行为**：
1. 加载所有规则（或 `--name` 指定）
2. 对 inbox 文件按规则顺序匹配
3. 命中 `actions` → 展开
4. 命中 `classify` → 调 AI（见 [08-classify.md](08-classify.md)）
5. 输出预演或写 uncommitted

## Rule 语法（YAML）

```yaml
# rules/default.yaml
- name: "PDF 按内容分类"
  description: "PDF 文件按 AI 分类到 Projects/{label}"
  enabled: true
  when:
    - ext: pdf
    - role: inbox              # 只对 inbox 角色文件夹生效
  classify:
    backend: ollama
    candidate_labels: [ai, finance, legal, personal]
    target_folder: "02-Projects/{label}"
    target_tags: ["{label}", "auto"]
    confidence_threshold: 0.7

- name: "图片按 EXIF 月份归档"
  when:
    - ext: [jpg, png, heic]
    - role: inbox
  actions:
    - move: "04-Photos/{exif.year}/{exif.month}/"
    - tag: ["photo", "by-exif"]

- name: "代码项目识别"
  when:
    - any:
      - has_file: "package.json"
      - has_file: "pyproject.toml"
  actions:
    - tag: ["code", "{detected_language}"]
    - move: "05-Code/{project_name}/"

- name: "临时文件清理"
  when:
    - ext: [tmp, bak, swp]
    - path: "*/temp/*"
  actions:
    - delete: true             # 走 trash
```

## 三段结构

| 段 | 作用 |
|---|---|
| **`when`** | 匹配条件（AND/OR 嵌套） |
| **`actions`** | 直接执行（不需 AI） |
| **`classify`** | 交给 AI 出 label，再用 label 模板生成 actions |

## `when` 支持的条件

| 字段 | 含义 | 示例 |
|---|---|---|
| `ext` | 扩展名（可多个） | `ext: pdf` 或 `ext: [jpg, png]` |
| `role` | folder role | `role: inbox` |
| `path` | 路径 glob | `path: "*/temp/*"` |
| `has_file` | 同目录有某文件 | `has_file: package.json` |
| `size` | 大小 | `size: ">100MB"` |
| `mtime` | 修改时间 | `mtime: ">7d"` |
| `filename` | 文件名 glob | `filename: "*draft*"` |
| `any` / `all` / `not` | 逻辑组合 | 见示例 |

## `actions` 支持的操作

| 操作 | 含义 |
|---|---|
| `move: <DST>` | 移动文件 |
| `rename: <NEW_NAME>` | 重命名 |
| `tag: [LIST]` | 加标签 |
| `untag: [LIST]` | 移除标签 |
| `delete: true` | 删除到 trash |

`{var}` 占位符：`{label}`（来自 classify）、`{exif.year}`、`{detected_language}`、`{filename}`、`{ext}` 等。

## 依赖关系

- **依赖**：[02-scan.md](02-scan.md)（file_meta）、[08-classify.md](08-classify.md)（classify 调用）
- **被依赖**：`rule run` 输出到 uncommitted

## 待定问题

- 规则冲突（多个规则匹配同一文件）的处理顺序？
- 规则条件是否需要正则支持？
- 规则测试框架（dry-run on sample）？
