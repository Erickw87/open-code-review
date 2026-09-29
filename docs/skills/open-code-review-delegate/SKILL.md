---
name: open-code-review-delegate
description: >
  OpenCodeReview（ocr）委托模式：OCR 只做确定性工程（文件筛选、规则解析），宿主
  Agent（DSH 会话）用自己的模型完成代码评审。当用户要求评审代码变更 / 分支对比 /
  单个 commit，且希望复用 DSH 自身模型、不消耗 ocr 配置的 LLM 端点时使用。不要求
  ocr 侧配置任何 LLM。
license: Apache-2.0
compatibility: >
  需要 ocr CLI 可用。本机：C:\Users\Admin\.codex\projects\feishu-ai-assistant\3rd\open-code-review\dist\opencodereview.exe
  （或按部署文档第 0 节配置 ocr 别名/重命名）。不需要配置 LLM 端点。
metadata:
  author: alibaba (DSH 本地化改编)
  homepage: https://github.com/Erickw87/open-code-review
  version: "1.0.0"
---

# OpenCodeReview 委托模式（ocr delegate）

本技能镜像官方 `plugins/open-code-review/skills/open-code-review-delegate/SKILL.md`
（为其 DSH/Windows 本地化改编）。OCR 提供确定性工程（文件筛选、规则解析），**评审
推理由宿主 Agent（即本会话）用自己的模型完成**。OCR 端零 LLM 调用。

## 调用约定

- 优先使用 `ocr`（用户已按部署文档第 0 节配置别名/复制到 PATH）；
- 若 `command not found`，改用完整路径：
  `/c/Users/Admin/.codex/projects/feishu-ai-assistant/3rd/open-code-review/dist/opencodereview.exe`
  （PowerShell 中为 `C:\Users\Admin\...\dist\opencodereview.exe`）。
- 目标仓库用 `--repo <路径>` 显式指定，Windows 路径形如 `D:\pro\src\micro\loadservice`。

## 工作流

### 第 1 步：Preview —— 确定评审范围

```bash
ocr delegate preview --repo <仓库路径> --format json [--from <ref> --to <ref>] [--commit <hash>] [--exclude <模式>]
```

输出（JSON，`schema_version: "1"`）：

- `mode`（workspace / range / commit）
- `from` / `to` / `commit` / `merge_base` —— 构造 git 命令所需的 ref 元数据
- `reviewable_files` —— 路径、状态、增删行数
- `excluded_files` —— 及排除原因

常见调用：

| 场景 | 命令 |
|---|---|
| 工作区变更 | `ocr delegate preview --repo <路径> --format json` |
| 分支对比 | `ocr delegate preview --repo <路径> --from dev --to feature/xxx --format json` |
| 单个 commit | `ocr delegate preview --repo <路径> -c abc123 --format json` |

### 第 2 步：取规则

```bash
ocr delegate rule --repo <仓库路径> <文件1> <文件2> ...
```

把第 1 步 `reviewable_files` 的路径传入；输出按规则内容分组（共享相同规则的文件
归为一组，避免重复）。大改动可分批次取规则。

### 第 3 步：取 diff

按第 1 步的 mode/ref 直接用 git 构造：

- **Range 模式**（用 preview 输出的 `merge_base`）：
  ```bash
  git -C <仓库路径> diff <merge_base>..<to> -- <文件>
  ```
- **Commit 模式**：
  ```bash
  git -C <仓库路径> show <commit> -- <文件>
  ```
- **Workspace 模式**：
  ```bash
  git -C <仓库路径> diff HEAD -- <文件>     # 已跟踪文件
  # 新增的未跟踪文件：直接读取全文（整文件都是新代码）
  ```

### 第 4 步：逐文件评审

以 `(path, status)` 为清单项（workspace 模式下同一路径可能以"staged 删除 +
untracked 重建"出现两次），对每个 `reviewable_files` 条目：

1. 取该文件 diff（第 3 步）
2. 对照其规则组（第 2 步）作为评审清单
3. 深入评审，需要时用 file_read / 搜索等工具核实调用点、同步、输入边界
4. 标记 `reviewed`，或 `skipped`（附具体原因）

大改动按"共享规则 + diff 大小"分批评审，不要找到第一个高严重度问题就停止。

### 第 5 步：输出格式（每条意见）

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| path | string | 是 | 相对文件路径 |
| content | string | 是 | 描述问题的评审意见 |
| start_line | integer | 否 | 新文件中的起始行 |
| end_line | integer | 否 | 新文件中的结束行 |
| category | enum | 否 | bug, security, performance, maintainability, test, style, documentation, other |
| severity | enum | 否 | critical, high, medium, low |

行号必须与第 3 步 diff 的 hunk 头（`@@ -a,b +c,d @@`）核对，只评论改动行（+ 行）。

### 第 6 步：分类与汇报

汇报前核验每个 previewed 文件都有归属；汇总需含 `total_files`、`reviewed_files`、
`skipped_files`、`coverage_rate`，被跳过的文件必须给原因。按严重度分组：

- **Critical/High**：bug、安全问题、数据丢失风险 —— 必须报告
- **Medium**：性能问题、错误处理缺失、可维护性问题 —— 附上下文报告
- **Low**：风格建议、微小改进 —— 仅在确有价值时报告

静默丢弃疑似误报，不追求数量。

### 第 7 步：修复（可选）

若用户要求 "review and fix"：High/Critical 直接修复；Medium 需要人工介入的说明
修复方案；Low 除非琐碎否则跳过。

## 子命令参考

| 命令 | 用途 |
|---|---|
| `ocr delegate preview` | 评审哪些文件 + mode/ref 元数据 |
| `ocr delegate rule <path...>` | 按内容分组的评审规则 |

## 常用参数

| 参数 | 说明 |
|---|---|
| `--repo <path>` | 仓库根目录（默认当前目录） |
| `--from <ref>` / `--to <ref>` | 区间模式 |
| `-c, --commit <hash>` | 单 commit 模式 |
| `--rule <path>` | 自定义 rule.json |
| `--exclude <模式>` | 逗号分隔的排除模式 |
| `-b, --background <text>` | 业务上下文（会出现在 preview 输出中供评审参考） |
| `-B, --background-file <path>` | 从 Markdown 读取上下文（优先于 -b；文件 ≤ 1MiB、净化后 ≤ 8000 字符） |
| `-f, --format <text\|json>` | agent 集成请用 `json` |

## 注意事项

- **OCR 侧不需要 LLM**：委托模式从不调用 LLM，所有智能来自宿主 Agent。
- **规则按内容分组**：可一次传任意数量路径；大改动按批次取规则。
- **工作目录**：`ocr delegate` 默认作用于当前目录的 git 仓库，用 `--repo` 覆盖。
- **untracked 文件**：workspace 模式 preview 会包含；这些文件直接读全文而非 git diff。
- **覆盖率是硬性要求**：每个 `reviewable_files` 条目必须最终为 reviewed 或显式
  skipped，不得静默遗漏。
- **`--format json` 版本兼容**：需 ocr v1.9.0+。若报 `unknown flag: --format`，
  去掉该 flag 用文本输出继续，不要伪造 JSON 字段。
- **`--background-file` 超限**：不静默截断源文件，先摘要（保留需求/约束/验收标准）
  再重试，必要时放弃 background 直接读原文。
- **Windows/DSH 细节**：PowerShell 与 Git Bash 命令略有差异（本技能示例用 bash 风格，
  PowerShell 下把 `/c/...` 换成 `C:\...`、单引号改为双引号时注意转义）；多行 JSON
  参数（如 `--rule`）建议先写入文件再传路径。
