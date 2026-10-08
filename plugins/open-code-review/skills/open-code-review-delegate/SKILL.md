---
name: open-code-review-delegate
description: >
  OpenCodeReview（OCR）的委托模式。与由 OCR 调用 LLM 端点不同，本技能指示宿主
  Agent 自行执行代码评审，OCR 只负责确定性工程：文件筛选与规则解析。当宿主 Agent
  应使用自身的 LLM 能力驱动评审时使用。
license: Apache-2.0
compatibility: >
  需要安装 `ocr` CLI（通过 `npm install -g @alibaba-group/open-code-review` 或
  GitHub release 二进制）。**不需要**配置 LLM 端点——委托模式下 OCR 侧零 LLM。
metadata:
  author: alibaba
  homepage: https://github.com/alibaba/open-code-review
  version: "1.0.0"
---

# Open Code Review —— 委托模式

本 Codex 插件技能刻意镜像 canonical 技能
`skills/open-code-review-delegate/SKILL.md`。更新 OCR 委托说明时请保持两个文件
同步；不采用符号链接，是因为插件安装可能只物化插件子树。

本技能用于执行 AI 代码评审：OCR 提供确定性工程（文件筛选、规则解析），宿主 Agent
使用自身的智能与工具完成实际评审。

## 工作流

### 第 1 步：Preview —— 确定评审范围

```bash
ocr delegate preview --format json [--from <ref> --to <ref>] [--commit <hash>] [--exclude <patterns>]
```

输出内容：

- **mode**（workspace / range / commit）
- **from / to / commit / merge_base** —— 构造 git 命令所需的 ref 元数据
- **可评审文件列表** —— 路径、状态、增删行数
- **被排除文件** —— 及排除原因

**常见调用：**

| 场景 | 命令 |
|----------|---------|
| 工作区变更 | `ocr delegate preview` |
| 分支对比 | `ocr delegate preview --from main --to feature` |
| 单个 commit | `ocr delegate preview -c abc123` |

### 第 2 步：获取文件规则

```bash
ocr delegate rule --format json <path1> <path2> ...
```

传入第 1 步的可评审文件路径。输出按规则内容分组——共享相同规则的文件归为一组，
避免重复。

### 第 3 步：获取 diff

根据第 1 步的 mode/ref 信息直接使用 git：

**Range 模式**（preview 输出中提供了 merge_base）：
```bash
git diff <merge_base>..<to> -- <path>
```

**Commit 模式**：
```bash
git show <commit> -- <path>
```

**Workspace 模式**：
```bash
# 已跟踪文件
git diff HEAD -- <path>
# 新增的未跟踪文件 —— 直接读取（整个文件都是新代码）
cat <path>
```

### 第 4 步：逐文件评审

创建包含每个 `reviewable_files` 条目的清单。对每个可评审文件：

使用 `(path, status)` 作为清单标识。workspace 模式下，当 staged 删除后紧跟
untracked 重建时，同一路径可能出现两次。

1. 获取其 diff（第 3 步）
2. 参照其规则组（第 2 步）作为评审清单
3. 进行深入评审，按需使用适当的上下文工具
4. 将文件标记为 `reviewed`，或 `skipped`（附具体原因）

对于大改动，按"共享规则 + diff 大小"分批进行有界的评审，不要找到第一个高严重度
问题就停止。

### 第 5 步：格式化输出

每条评论必须遵循以下结构：

| 字段 | 类型 | 必填 | 说明 |
|-------|------|----------|-------------|
| path | string | 是 | 相对文件路径 |
| content | string | 是 | 描述问题的评审意见 |
| start_line | integer | 否 | 新文件中的起始行 |
| end_line | integer | 否 | 新文件中的结束行 |
| category | enum | 否 | bug, security, performance, maintainability, test, style, documentation, other |
| severity | enum | 否 | critical, high, medium, low |

### 第 6 步：分类与汇报

汇报前核验每个 previewed 文件都有归属。汇总需包含 `total_files`、
`reviewed_files`、`skipped_files` 与 `coverage_rate`。被跳过的文件必须给出原因。

按严重度分组：

- **Critical/High**：Bug、安全问题、数据丢失风险 —— 始终报告
- **Medium**：性能问题、错误处理缺失、可维护性问题 —— 附上下文报告
- **Low**：风格瑕疵、微小建议 —— 仅在确有价值时报告

静默丢弃疑似误报。

### 第 7 步：修复（可选）

若用户要求 "review and fix"：
- High/Critical 直接修复
- 需要人工介入的 Medium 修复，说明方案
- Low 除非琐碎否则跳过

## 子命令参考

| 命令 | 用途 |
|---------|---------|
| `ocr delegate preview` | 评审哪些文件 + mode/ref 元数据 |
| `ocr delegate rule <path...>` | 按内容分组的评审规则 |

## 共享参数

| 参数 | 说明 |
|------|-------------|
| `--from <ref>` | Range 模式的源 ref |
| `--to <ref>` | Range 模式的目标 ref |
| `-c, --commit <hash>` | 单 commit 模式 |
| `--repo <path>` | 仓库根目录（默认：当前目录） |
| `--rule <path>` | 自定义 rule.json 路径 |
| `--exclude <patterns>` | 逗号分隔的排除模式 |
| `-b, --background <text>` | 业务上下文 |
| `-B, --background-file <path>` | 从 Markdown 文件读取业务上下文（优先于 `-b`） |
| `-f, --format <text\|json>` | 输出格式；agent 集成请用 `json` |

## 注意事项

- **OCR 侧不需要 LLM** —— 委托模式从不调用 LLM，所有智能来自宿主 Agent。
- **规则按内容分组** —— 共享相同规则的文件在输出中归为一组。每次调用可传入任意
  数量路径；大改动按评审进度分批次取规则。
- **工作目录很重要** —— `ocr delegate` 作用于当前目录下的 Git 仓库。用
  `--repo /path` 覆盖。
- **workspace 模式下的未跟踪文件** —— `preview` 包含未跟踪文件。对这类文件直接
  读取文件内容，而不是用 `git diff`。
- **背景上下文** —— 有需求上下文时向 `preview` 传 `--background`；它会出现在输出
  中供评审时参考。
- **覆盖率是硬性要求** —— 每个 `reviewable_files` 条目必须以 reviewed 或显式
  skipped 结束；不得静默遗漏文件。

### 恢复超限的背景上下文

`--background-file` 有两个独立限制：原始文件不得超过 1 MiB，净化后的内容不得超过
8000 字符。任一条件触发都会中止命令。当命令报告任一限制时：

1. 不要静默截断源文件。
2. 对原始材料做摘要，同时保留其需求、约束、验收标准及其他评审关键细节。
3. 以单条 shell 安全参数重试受影响的命令（例如使用宿主 shell 生成带引号/转义的
   参数，或写入新的、大小受限的文件再传入）。不要把不受信任的摘要文本直接放进
   双引号 shell 模板里；`$()`、反引号、引号与变量引用仍可能被求值。省略原
   `--background-file`，以免 CLI 重新加载同一个超大文件再次失败。
4. 若无法做忠实摘要，则完全省略 OCR 背景，评审时直接阅读原始材料。

### CLI 版本兼容性排查

`--format` 参数自 `ocr` v1.9.0 起可用。技能与已安装 CLI 可各自独立更新。若请求的
`preview` 或 `rule` 命令带 `--format json` 时恰好报 `unknown flag: --format`，去掉
该参数重跑，用文本输出完成后续委托流程。保留该输出中的显式 mode、ref、文件与规则
信息；不要把文本输出当 JSON 解析，也不要凭空编造缺失的 schema 字段。其他任何错误
都不要去掉参数重试；直接报告并停止受影响的流程。

宿主 Agent 技能也可以消费等价的文本输出来完成评审清单。需要 `schema_version` 或
其他 JSON 字段的程序化集成，必须改用支持 JSON 的 CLI：用 `ocr --version` 验证，
必要时升级：

```bash
npm install -g @alibaba-group/open-code-review
```
