# 08 · AI 分类（classify）

> 主文档: [DESIGN.md](../DESIGN.md) · 相关: [09-rule.md](09-rule.md)

## 简介

用 AI 理解文件内容，输出标签 / 目标文件夹建议。可插拔后端（Ollama / OpenAI / DeepSeek）。**输出是 actions 序列**，进 `uncommitted.json`，等用户 `commit`。

## 命令清单

| 命令 | 功能 |
|---|---|
| `filetidy classify [PATH]` | 对 inbox 文件跑 AI 分类 |
| `filetidy classify --backend NAME` | 指定后端 |
| `filetidy classify --dry-run` | 预演（默认） |

## 详细设计

### `filetidy classify [PATH] [--dry-run|--apply]`

**功能**：对 PATH（或所有 `inbox` 角色文件夹）的文件跑 AI 分类。

**参数**：`PATH`（可选，限定范围）

**选项**：
- `--backend NAME`：覆盖 config 默认（`ollama` / `openai` / `deepseek`）
- `--dry-run`：只预演，不写 `uncommitted.json`（**默认**）
- `--apply`：直接写 uncommitted（不立即 commit）
- `--rule NAME`：用指定规则集
- `--limit N`：最多处理 N 个文件
- `--threshold FLOAT`：AI 置信度阈值（低于则不自动加，默认 0.7）

**行为**：
1. 收集 inbox 文件（受 folder role 影响）
2. 对每个文件：
   - 顺序匹配规则（见 [09-rule.md](09-rule.md)）
   - 命中 `actions` 规则 → 直接展开
   - 命中 `classify` 规则 → 调 AI 后端拿 label → 注入模板生成 actions
   - 收集 actions
3. 输出预演报告（`--dry-run`）或写入 `uncommitted.json`（`--apply`）

**示例**：
```bash
$ filetidy classify --dry-run
Classifying 15 inbox files with backend 'ollama'...

Preview (15 actions, 5 distinct files):
  01-Inbox/paper.pdf      → ai/ml      (confidence: 0.92)
                          + tag: ai/ml, paper
                          + move: 02-Projects/ai/
  01-Inbox/report.docx    → finance    (confidence: 0.85)
                          + tag: finance, quarterly
                          + move: 02-Projects/finance/
  ...

Stats: 12 confident, 3 below threshold (will be skipped)
Run `filetidy classify --apply` to write to uncommitted.
```

## Classifier 后端抽象

```python
class ClassifierBackend(ABC):
    @abstractmethod
    def classify(
        self,
        meta: FileMeta,
        content_sample: str,
        candidate_labels: list[str]
    ) -> ClassificationResult:
        ...

@dataclass
class ClassificationResult:
    label: str
    confidence: float
    reasoning: str             # 给用户看的可解释性
    alternatives: list[dict]   # 备选 [{label, confidence}]
```

三个实现：
- `OllamaBackend`：本地（默认，隐私友好）
- `OpenAIBackend`：`gpt-4o-mini`
- `DeepSeekBackend`：`deepseek-chat`

通过 `config.yaml` 的 `backend.default` + per-folder override 切换。

## Prompt 模板（统一）

```
SYSTEM:
You are a file classifier. Output STRICT JSON.
Schema: {"label": str, "confidence": float, "reasoning": str}

USER:
File: {filename}
Size: {size} | Type: {mime}
Content sample (first 2KB):
---
{content_sample}
---
Candidate labels: {candidate_labels}
```

## 内容采样策略（控制 token 成本）

| 文件类型 | 采样 |
|---|---|
| 文本（md/txt/code/log） | 头 2KB |
| PDF | 提取前 1 页文字 |
| 图片（jpg/png/heic） | EXIF + 文件名（不发图） |
| Office（docx/xlsx/pptx） | 提取文本头 1KB |
| 二进制 / 其他 | 只看文件名 + 扩展名 |

## 后端选择策略

每次 classify 时：
1. 文件所在 folder 是否有 override（per-folder）
2. 命令行 `--backend` 是否指定
3. vault config 的 `backend.default`
4. 全局 `~/.filehelper/config.yaml`

## 依赖关系

- **依赖**：[09-rule.md](09-rule.md)（规则匹配）、[02-scan.md](02-scan.md)（file_meta）
- **被依赖**：输出 actions 到 uncommitted

## 待定问题

- AI rate limit / retry 策略（指数退避？）
- 多标签分类（一个文件可能同时是 `ai/ml` 和 `paper/2024`）
- 失败成本控制（一次 classify 跑爆 API）？
- 本地 Ollama 连接失败的 fallback？
