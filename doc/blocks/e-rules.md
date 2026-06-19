# Block E · Rules（规则块）

> 主文档: [DESIGN.md](../DESIGN.md) · 协议: [actions.md](../core/actions.md)

## 简介

模板化、声明式的批量整理规则（YAML）。**上层调度者**：不直接改文件，而是组合调用 Block B (Ops) 和 Block D (Tags) 的 API。无 AI 决策，纯固定规则。

---

## 内部组分

| 组分 | 职责 |
|---|---|
| `RuleParser` | YAML 解析 + 语法验证 |
| `RuleMatcher` | 文件匹配（`when` 条件） |
| `RuleExecutor` | 执行 actions（**调用 B/D，产出 Action 序列**） |
| `RuleStore` | 规则文件管理（`rules/*.yaml`） |

---

## 对外命令

```bash
filetidy rule add <FILE>                              # 注册规则
filetidy rule ls                                       # 列出
filetidy rule run [--name N] [--dry-run|--apply]      # 执行
filetidy rule validate <FILE>                          # 语法验证
filetidy rule rm <NAME>                                # 移除
```

---

## 对外 API

```python
class Rules:
    def add(self, rule_file: Path) -> RuleId
    def list(self) -> list[RuleSummary]
    def run(self, paths: list[Path] | None, name: str | None = None, dry_run: bool = True) -> list[Action]
    def validate(self, rule_file: Path) -> ValidationResult
    def remove(self, rule_id: RuleId) -> None
```

---

## 规则语法（YAML）

```yaml
# rules/default.yaml
- name: "图片按 EXIF 月份归档"
  description: "照片按拍摄时间归档"
  enabled: true
  when:
    - ext: [jpg, png, heic]
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
    - delete: true              # 走 trash
```

---

## 两段结构

| 段 | 作用 |
|---|---|
| `when` | 匹配条件（AND/OR 嵌套） |
| `actions` | 直接执行（不需 AI） |

**注意**：没有 `classify` 段（去 AI）。Phase 2 如果加回 AI，规则可以扩展 `classify` 段。

---

## `when` 条件

| 字段 | 含义 | 示例 |
|---|---|---|
| `ext` | 扩展名（可多个） | `ext: pdf` 或 `ext: [jpg, png]` |
| `path` | 路径 glob | `path: "*/temp/*"` |
| `has_file` | 同目录有某文件 | `has_file: package.json` |
| `size` | 大小 | `size: ">100MB"` |
| `mtime` | 修改时间 | `mtime: ">7d"` |
| `filename` | 文件名 glob | `filename: "*draft*"` |
| `any` / `all` / `not` | 逻辑组合 | 见示例 |

---

## `actions` 操作

| 操作 | 调用 |
|---|---|
| `move: <DST>` | `Block B.mv()` |
| `rename: <NEW_NAME>` | `Block B.mv()`（同目录） |
| `delete: true` | `Block B.rm()` |
| `mkdir: <PATH>` | `Block B.mkdir()` |
| `tag: [LIST]` | `Block D.tag()` |
| `untag: [LIST]` | `Block D.untag()` |

---

## `{var}` 模板变量

| 变量 | 来源 |
|---|---|
| `{filename}` | 文件名（含扩展） |
| `{stem}` | 文件名（不含扩展） |
| `{ext}` | 扩展名 |
| `{path}` | 完整路径 |
| `{parent_dir}` | 父目录名 |
| `{size}` | 大小（字节） |
| `{mtime.year}` / `{mtime.month}` / etc. | 修改时间 |
| `{exif.year}` / `{exif.month}` etc. | EXIF（仅图片） |
| `{project_name}` | 同目录 package.json/pyproject.toml 推断 |
| `{detected_language}` | 文件扩展名推断（.py → python） |

---

## RuleExecutor 工作流

```python
def run(paths: list[Path] | None, name: str | None, dry_run: bool) -> list[Action]:
    rules = load_rules(name=name)
    candidates = paths or scan_all_files()

    all_actions = []
    for rule in rules:
        if not rule.enabled:
            continue
        for file_path in candidates:
            if rule.matches(file_path):
                # 渲染模板变量
                context = extract_context(file_path)
                rendered_actions = [render(a, context) for a in rule.actions]

                if not dry_run:
                    # 实际调用 B 和 D
                    for action_def in rendered_actions:
                        action = dispatch_to_block(action_def)
                        all_actions.append(action)
                else:
                    # 预演：构造虚拟 Action
                    all_actions.extend(make_preview_actions(rendered_actions))

    return all_actions
```

**关键**：
- `dispatch_to_block` 调用 `Block B.mv()` / `Block D.tag()` 等
- 这些 block 内部已经处理 Action 推送（自动进 uncommitted）
- `rule run` 返回的 Action 列表只是给用户看的预览

---

## 默认 `--dry-run`

```bash
filetidy rule run                # 预演，不写 uncommitted
filetidy rule run --apply        # 实际执行，写入 uncommitted
filetidy rule run --name "图片按 EXIF" --apply
```

---

## 规则冲突

多个规则匹配同一文件时，按 `rules.yaml` 中的顺序执行。可通过 `priority` 字段调整（Phase 2）。

---

## 与其他块的关系

| 块 | 关系 |
|---|---|
| Block A (Index) | 读 `file_record` 做匹配 |
| Block B (Ops) | **被调用**（执行 mv/rm 等） |
| Block D (Tags) | **被调用**（执行 tag/untag） |
| Block C (History) | 通过 B/D 间接写入 Action |

---

## 待定问题

- 规则优先级（`priority` 字段）？
- 规则测试框架（dry-run on sample）？
- 规则变量是否需要支持正则提取？
