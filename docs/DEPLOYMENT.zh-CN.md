# OpenCodeReview（ocr）本地部署指南

> 适用仓库：`https://github.com/Erickw87/open-code-review`（fork 自
> `alibaba/open-code-review`，本地分支 `dev`）。
> 本文面向人工或 AI 助手，给出在本机把 OpenCodeReview 从源码跑起来、并完成
> LLM 配置与首次代码评审的完整步骤。

---

## 0. 命令名说明：`ocr` 与 `opencodereview.exe` 的关系

编译产物叫 `opencodereview.exe`（项目名），而本文中写的 `ocr` 只是它的**别名**。
本项目 FAQ 明确说明：release 二进制以项目命名（`opencodereview`），NPM wrapper
为便于使用安装为 `ocr`；从源码构建得到 `dist/opencodereview`——复制为 `$PATH`
上的 `ocr` 即可。两者子命令完全一致（`config` / `review` / `scan` / `llm` /
`viewer` / `delegate` ...），只是可执行文件名不同。

本机（Windows + Git Bash）想直接用 `ocr` 命令，任选其一：

- **复制/重命名到 PATH**（推荐）：把 `dist\opencodereview.exe` 复制为 `ocr.exe`
  放入 PATH 目录，例如：
  ```bash
  mkdir -p ~/bin && cp dist/opencodereview.exe ~/bin/ocr.exe
  echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc
  ```
- **别名（仅 Git Bash 交互式 shell 生效）**：在 `~/.bashrc` 追加
  `alias ocr='/c/Users/Admin/.codex/projects/feishu-ai-assistant/3rd/open-code-review/dist/opencodereview.exe'`，
  然后 `source ~/.bashrc`。

不想做任何安装动作时，文中所有 `ocr <子命令>` 都可以等价写成
`./dist/opencodereview.exe <子命令>`（已在 `dist/` 目录内则为
`./opencodereview.exe <子命令>`）。

---

## 1. 项目概览："本地运行"到底指什么

OpenCodeReview（CLI 命令为 `ocr`）是一个 **Go 编写的命令行工具**，不是 Web 服务。

- 它**读取本地 Git 仓库的 diff**（`git diff` / `git show`），把变更内容发送给你
  配置的 **LLM 端点**（云端 API 或本地模型服务），由 LLM 以"工具调用（function
  calling）"的方式执行评审，最后产出结构化、定位到行号的评审意见。
- 评审过程中的会话记录（JSONL）、规则文件、配置文件全部存放在本机
  `~/.opencodereview/`，**不依赖任何数据库或常驻服务**。
- 唯一的"外部依赖"是 **LLM 端点**：要么是你自己的 API Key（Anthropic / OpenAI /
  DeepSeek / 通义 / Kimi / 智谱 / SiliconFlow 等），要么是本地自建的
  OpenAI 兼容服务（Ollama、vLLM、LiteLLM 等）。另外还有**委托模式**（`ocr delegate`）
  可以完全不配 LLM，把推理交给宿主的 AI 编码 Agent。

结论：**工程可以本地运行**。下面按"安装 CLI → 配置 LLM → 运行评审"三步展开。

## 2. 本机验证情况（2026-06 实测）

在本机（Windows 11 / amd64，Go 1.25.5、Git 2.49）已实际完成验证：

| 步骤 | 结果 |
|---|---|
| `go build` 编译 CLI | ✅ 成功，产出 `dist\opencodereview.exe` |
| `ocr version` / `ocr --help` | ✅ 正常输出 |
| `ocr llm test`（未配置时） | ✅ 按预期报 `no valid LLM endpoint configured`，提示需要配置 |
| `ocr review --commit HEAD --preview` | ✅ 正确列出上次提交的候选文件与排除原因 |

> 说明：`review --preview` 只跑文件筛选、不调用 LLM，零成本，是部署完成后
> 快速自检的最好方式。

## 3. 前置条件

| 依赖 | 最低版本 | 用途 | 本机状态 |
|---|---|---|---|
| Go | ≥ 1.25（`go.mod` 声明 `go 1.25.5`） | 编译源码 | ✅ 1.25.5 |
| Git | ≥ 2.41 | diff 生成、代码搜索、仓库操作 | ✅ 2.49 |
| Make | 任意 GNU make | 可选，`make build` 需要；也可直接 `go build` | ✅（需配合 POSIX shell） |
| Node.js | ≥ 18 | 可选，仅 NPM 安装方式需要 | ✅ 24.12 |

> - 编译器本身不挑剔平台：Windows / macOS / Linux 均受支持（`cmd/opencodereview`
>   是纯 Go，`CGO_ENABLED=0` 静态编译）。
> - 如果本机没有 Go，从 <https://go.dev/dl/> 下载安装，并确保 `go` 在 `PATH` 中。

## 4. 方式一：从源码构建（推荐，用于运行 fork 代码）

从源码构建会编译**当前 checkout 的代码**——要运行本 fork（`dev` 分支）的改动，
这是唯一准确的方式（NPM 安装的是上游发布版二进制，见第 5 节）。

### 4.1 Windows（PowerShell）

```powershell
cd C:\Users\Admin\.codex\projects\feishu-ai-assistant\3rd\open-code-review

# 直接使用 go build（不依赖 make / sh，推荐）
go build -o dist\opencodereview.exe .\cmd\opencodereview

# 或者：使用 Makefile 的 Windows 目标（会同时带版本号 ldflags）
make build-windows-amd64        # 产出 dist\opencodereview-windows-amd64.exe
```

> 若用 `make build`（产出 `dist\opencodereview`），需要注意 Windows 上 GNU make
> 会调用 PATH 中的 `sh.exe`（通常来自 Git Bash），因此需要在 Git Bash 终端里执行，
> 而不是 PowerShell 或 cmd。PowerShell 下直接用 `go build` 更省事。

### 4.2 Linux / macOS

```bash
cd open-code-review
make build                       # 产出 dist/opencodereview
sudo cp dist/opencodereview /usr/local/bin/ocr   # 可选：放入 PATH 方便全局调用
```

### 4.3 构建中可能遇到的问题

| 现象 | 原因与解决 |
|---|---|
| `Access is denied`（`C:\Users\...\go\pkg\mod\cache\...`） | Go 模块缓存目录不可写（例如受限环境/沙箱）。把缓存指到可写目录：`$env:GOMODCACHE="<可写目录>\mod"; $env:GOCACHE="<可写目录>\build"` 后再 `go build`。普通终端环境一般不会遇到。 |
| `make: *** Error 256` + `sh.exe ... fatal error - CreateFileMapping` | Windows 下 make 调用的 msys/cygwin `sh` 崩溃。改用 `go build`，或在 Git Bash 中执行 `make`。 |
| 首次构建较慢 | 需要下载全部 Go 依赖（约 90 个模块），属正常现象；之后有缓存会很快。 |

## 5. 方式二：NPM 全局安装（运行上游发布版）

```bash
npm install -g @alibaba-group/open-code-review
ocr version
```

说明：

- NPM 包只是一个 wrapper：它按平台下载 **GitHub Release 上的预编译二进制**。
- 因此这种方式装到的是**上游发布版**，而不是本 fork 的 `dev` 代码。要跑 fork
  代码请用第 4 节的源码构建。
- 可选安装方式：Homebrew（macOS/Linux）、`curl -fsSL https://open-codereview.ai/install.sh | sh`、
  直接下载 release 二进制。详见仓库内文档 `pages/src/content/docs/zh/installation.md`。

## 6. 安装验证

```bash
ocr version          # 打印版本 + 平台
ocr --help           # 顶层命令列表：review / scan / config / llm / delegate / session / viewer / rules ...
ocr review --help    # review 子命令的全部参数
```

验证通过后进入配置环节。**未配置 LLM 时**任何评审命令都会报：

```
Error: resolve LLM endpoint: no valid LLM endpoint configured; one of
OCR_LLM_URL/OCR_LLM_TOKEN/OCR_LLM_MODEL, ~/.opencodereview/config.json, or
ANTHROPIC_BASE_URL/ANTHROPIC_AUTH_TOKEN/ANTHROPIC_MODEL must be set
```

## 7. 配置 LLM（本地运行的必需配置）

配置文件位置：`~/.opencodereview/config.json`，**与用户主目录绑定，与二进制或
`dist/` 目录无关**——OCR 不会把配置或结果写到二进制旁边。本机（用户 `Admin`）
的具体路径：

- Windows（PowerShell）：`C:\Users\Admin\.opencodereview\config.json`
- Git Bash：`/c/Users/Admin/.opencodereview/config.json`（即 `~/.opencodereview/config.json`）

评审产生的**会话记录**同样放在主目录：
`~/.opencodereview/sessions/<编码后仓库路径>/<session-id>.jsonl`，也不会出现在
`dist/` 或仓库目录下。删除 `~/.opencodereview` 即可清除全部配置与历史。

该文件在首次运行配置命令（`ocr config provider` 或 `ocr config set ...`）时自动
创建；也可以手动新建目录并放置文件（见 7.8 的手写示例）。

有三种配置方式，任选其一；推荐交互式。

### 7.1 交互式配置（推荐）

```bash
ocr config provider   # 选择内置 provider 或添加自定义 provider、填 API Key、自动测试连通性
ocr config model      # 为当前 provider 挑选模型
```

TUI 会引导你完成"选 provider → 填 key → 选 model"，保存后自动运行一次
`ocr llm test` 验证端点。

### 7.2 非交互式配置（适合脚本 / AI / CI）

```bash
# 以 DeepSeek 为例（内置 provider）
ocr config set provider deepseek
ocr config set model deepseek-v4-pro
ocr config set providers.deepseek.api_key sk-xxxx

# 以 Anthropic 为例
ocr config set provider anthropic
ocr config set model claude-sonnet-4-6
ocr config set providers.anthropic.api_key sk-ant-xxxx
```

> 本 fork（`dev` 分支）内置 provider 的模型清单以 `internal/llm/providers.go`
> 为准（如 DeepSeek 预置 `deepseek-v4-pro` / `deepseek-flash`，通义预置
> `qwen3.8-max` 等）。模型名可随时用 `ocr config model` 或 `ocr config set model <名>`
> 更换，不必受预置清单限制。

### 7.3 内置 Provider 速查

选中内置 provider 后只需填 API Key；未配置 `providers.<name>.api_key` 时会
自动回退到对应环境变量。常用内置 provider（完整列表见
`internal/llm/providers.go` 与 `pages/src/content/docs/zh/configuration.md`）：

| 名称 | 协议 | Base URL | 环境变量 |
|---|---|---|---|
| `anthropic` | anthropic | `https://api.anthropic.com` | `ANTHROPIC_API_KEY` |
| `openai` | openai | `https://api.openai.com/v1` | `OPENAI_API_KEY` |
| `deepseek` | openai | `https://api.deepseek.com` | `DEEPSEEK_API_KEY` |
| `dashscope` | openai | `https://dashscope.aliyuncs.com/compatible-mode/v1` | `DASHSCOPE_API_KEY` |
| `kimi` | openai | `https://api.moonshot.cn/v1` | `MOONSHOT_API_KEY` |
| `z-ai` | openai | `https://open.bigmodel.cn/api/paas/v4` | `Z_AI_API_KEY` |
| `siliconflow-cn` | openai | `https://api.siliconflow.cn/v1` | `SILICONFLOW_API_KEY` |
| `volcengine` | openai | `https://ark.cn-beijing.volces.com/api/v3` | `ARK_API_KEY` |
| `bedrock` | anthropic-bedrock | 由 `aws_region` 决定（AWS 凭证链，无 Key） | — |

### 7.4 自定义 Provider（本地模型 / 企业网关）

不在内置列表中的 provider 名即为自定义，**必须**提供 `url` 和 `protocol`
（`anthropic` / `openai` / `openai-responses` / `anthropic-bedrock`）。

本地 Ollama 示例（OpenAI 兼容端点）：

```bash
ocr config set provider                          ollama
ocr config set custom_providers.ollama.url       http://127.0.0.1:11434/v1
ocr config set custom_providers.ollama.protocol  openai
ocr config set custom_providers.ollama.model     qwen3:32b
ocr config set custom_providers.ollama.api_key   ollama
```

> - 自定义 provider **没有**环境变量回退，因此 `api_key` 必须非空；Ollama 忽略
>   key，填任意占位值即可。
> - 本地模型必须**支持原生工具调用（function calling）**，否则评审会陷入
>   "No tool calls parsed" 循环——见第 12 节排查。推荐从 Ollama 带 tools 标签的
>   模型里选：<https://ollama.com/search?c=tools>。
> - 本地模型响应慢时可调大超时：`ocr config set custom_providers.ollama.timeout_sec 900`
>   （默认 300 秒）。

vLLM / LiteLLM / 企业网关同理，换 URL 与模型名即可。也可以用
`providers.<name>.url` 覆盖内置 provider 的 Base URL，把内置 provider 指向自建网关。

### 7.5 通过环境变量配置（不写配置文件）

以下三件套齐备即视为已配置（优先级：配置文件 > 环境变量 > Claude Code 环境变量）：

```bash
export OCR_LLM_URL=...      # 如 https://api.deepseek.com
export OCR_LLM_TOKEN=...    # API Key
export OCR_LLM_MODEL=...    # 模型名
export OCR_LLM_PROTOCOL=openai    # anthropic | openai | openai-responses
```

> 已配置过 `~/.opencodereview/config.json` 时，环境变量会被忽略（取第一个完整的
> 三元组）。要让环境变量生效需先清掉配置文件里的对应项。

### 7.6 设置评审输出语言（可选）

```bash
ocr config set language 中文     # 评审意见用中文输出（默认英文）
```

### 7.7 验证连通性

```bash
ocr llm test
```

成功时输出类似：

```
Source: provider:deepseek
URL:    https://api.deepseek.com
Model:  deepseek-v4-pro
✓ Connection test successful
```

`401 / 403` 表示 token 错误或已过期；`no valid LLM endpoint configured`
表示配置不完整，回到 7.2 补齐 `(url, token, model)`。

### 7.8 手动创建 `config.json`（示例）

如不便运行交互命令，可手动创建目录与文件。以 DeepSeek 为例：

```powershell
# PowerShell：新建目录
New-Item -ItemType Directory -Force "$env:USERPROFILE\.opencodereview" | Out-Null
```

`C:\Users\Admin\.opencodereview\config.json`：

```json
{
  "provider": "deepseek",
  "model": "deepseek-v4-pro",
  "providers": {
    "deepseek": {
      "api_key": "sk-你的真实Key"
    }
  },
  "language": "中文"
}
```

要点：

- `provider` + `providers.<name>` 是**推荐结构**：内置 provider 的 URL/协议已预置，
  只需填 `api_key`；自定义 provider 则必须给 `url` 和 `protocol`。
- 也可用旧版 `llm` 段（`llm.url` / `llm.auth_token` / `llm.model` / `llm.protocol`），
  二者取其一即可，`provider` 段优先。
- 手动写文件后运行 `ocr llm test` 验证；下次执行 `ocr config set` 时文件会被
  重新格式化（属正常现象）。
- `api_key` 可留空并改用环境变量回退（内置 provider 会读 `DEEPSEEK_API_KEY` 等）。

## 8. 运行第一次评审

进入**任意 Git 仓库**（`ocr review` 依赖 git）：

```bash
cd /path/to/your/git-repo

# 工作区模式（默认）：评审 staged + unstaged + untracked 变更
ocr review

# 分支区间：评审 feature 相对 main 分叉后的变更
ocr review --from main --to feature-branch

# 单个 commit
ocr review --commit abc123

# 全文件扫描（不需要 diff / git 历史）
ocr scan
ocr scan --path internal/agent
```

先零成本预览会评审什么（不调用 LLM）：

```bash
ocr review --preview                    # 文本预览：文件 + 排除原因
ocr review --commit HEAD --preview --format json --audience agent   # 机器可读
```

机器可读输出（推荐给 AI / CI 使用）：

```bash
ocr review --format json --audience agent --output result.json
# --audience agent 屏蔽进度行，让 stdout 只含 JSON
```

常用参数（详见仓库内文档 `pages/src/content/docs/zh/cli-reference.md`）：

| 参数 | 作用 |
|---|---|
| `--repo <path>` | 指定仓库根目录（默认当前目录） |
| `--from <ref> --to <ref>` / `--commit <sha>` | 评审范围 |
| `--format text|json|sarif` | 输出格式（sarif 供 GitHub Code Scanning） |
| `--output <path>` | 写入文件 |
| `--concurrency <n>` | 并行子任务数（默认 8） |
| `--effort low|medium|high` | 评审轮数：1 / 2 / 3 轮（默认 medium） |
| `--max-tokens <n>` | 单子任务提示词上限（review 默认 200000） |
| `--max-tokens-budget <n>` | 本次运行总 token 预算（输入+输出） |
| `--background <text>` / `--background-file <path>` | 注入业务上下文，显著提升评审质量 |
| `--resume <session-id>` | 恢复中断的评审（先用 `ocr session list` 查 id） |

查看评审历史：

```bash
ocr session list        # 列出本仓库的评审会话
ocr viewer              # 启动浏览器端的会话查看器（本地 Web UI）
```

## 9. 无需 LLM 的部署：委托模式（delegate）

如果你使用 Claude Code / Codex / Cursor 等订阅制 AI 编码 Agent，可让 **宿主 Agent
用自己的模型执行评审**，OCR 只负责确定性工程（文件筛选、规则解析），**完全不需
配置 LLM**：

```bash
ocr delegate preview                   # 列出可评审文件 + mode/ref 元数据
ocr delegate rule src/main.go src/handler.go   # 解析这些文件匹配的评审规则
```

详细工作流见 `pages/src/content/docs/zh/integrations/delegate.md`。

### 9.1 委托模式 + DSH（DeepSeek Harness）本地部署与实测

**场景**：本机已有 DSH（DeepSeek Harness，即运行 AI 会话的本地环境）且其模型可用；
不想为 OCR 单独配置/消耗云端 LLM 端点时，让 **DSH 宿主 Agent 用自己的模型执行
评审**。OCR 端只做确定性工程（文件筛选、规则解析），**delegate 模式不读取也不
需要 `~/.opencodereview/config.json` 中的 LLM 配置**（已有配置也不影响）。

**部署步骤（在 DSH 会话中交给 Agent 执行）**：

```bash
# 0) 前置：按第 4 节编译好二进制（dist/opencodereview.exe），并准备目标 git 仓库
#    与要评审的分支（--from <base> --to <feature>，两分支都必须存在）

# 1) 文件筛选：列出可评审文件 + mode/ref 元数据（merge_base 从这里取）
ocr delegate preview --repo <仓库路径> --from <base> --to <feature> --format json

# 2) 规则解析：为可评审文件获取匹配的评审规则（按内容分组输出）
ocr delegate rule <文件1> <文件2> ... --repo <仓库路径>

# 3) 构造评审材料：按 preview 输出的 merge_base 取 diff
git -C <仓库路径> diff <merge_base>..<feature> -- <文件1> <文件2> ...

# 4) 宿主 Agent（DSH 模型）按第 2 步规则逐文件评审第 3 步 diff，
#    产出结构化意见：path / start_line / end_line / severity / category / message
```

**与 `ocr review` 的对比**：

| 维度 | `ocr review`（OCR 自管 LLM） | 委托模式 + DSH |
|---|---|---|
| 谁调用 LLM | OCR 调用 `config.json` 配置的端点 | DSH 宿主 Agent 调用自己的模型 |
| OCR 端 LLM 配置 | 必需（第 7 节） | 不需要 |
| OCR 端 token 消耗 | 有（评审轮 + 压缩轮） | 无（仅 preview/rule 本地计算） |
| 评审能力 | OCR 内置 agent 循环（工具调用、多轮） | 取决于宿主 Agent 的模型与工具能力 |
| 输出 | `--format json/sarif` 结构化评论 | 宿主 Agent 自行汇总（可用 JSON 规范约束） |

**实测记录（2026-06）**：对 `D:\pro\src\micro\loadservice`（Go 项目，分支
`dev` 与 `feature/redis_state`，merge-base `093b9ac`）执行上述流程：preview 选出
12 个可评审文件（3 个测试文件被默认规则排除），rule 给出内置 Go 评审原则，
Agent 依据 `git diff 093b9ac..feature/redis_state` 完成评审，共产出 3 条发现
（1 条安全 / 2 条正确性-可维护性，详见工作区报告 `loadservice-redis-state-delegation-review.md`）。

**限制与注意**：

- `--from`/`--to` 引用的分支必须真实存在，否则报
  `--to value "xxx" is not a valid commit ref`。
- 委托模式下 OCR 不做多轮工具调用与行号重定位，意见的**行号准确性取决于宿主
  Agent 对照 diff 的仔细程度**；关键行号建议用 `git diff` 的 hunk 头核对。
- 默认规则会排除 `*_test.go`；如需评审测试文件，用 `--exclude`/`--rule` 调整。

### 9.2 把委托技能挂载到 DSH 预设（一键化）

目标：让 DSH 会话的**技能目录**出现 `open-code-review-delegate`，会话内读取该技能
即按 9.1 的 7 步流程执行——与 Codex/Cursor 加载同一份 SKILL.md 完全同构，只是把
技能文件放进了 DSH 预设。

**机制**：预设的 `agent.cordis.yml` 中有一行 `@deepseek-ai/dsh-skill-filesystem`
（`customSkillDirs` 指向预设目录下的 `skills/`），会话挂载时会把该目录下每个含
`SKILL.md` 的子目录注册为一个技能。因此挂载新技能 = 在预设的 `skills/` 下放一个
新子目录。

**步骤**：

```bash
# 1) 确认当前预设目录（本机 DSH 用户预设根：C:\Users\Admin\.dsh\.agent-presets\，
#    当前会话使用 book-study）
ls ~/.dsh/.agent-presets/

# 2) 技能源文件（本仓库内已备好，DSH/Windows 本地化改编版）：
#    docs/skills/open-code-review-delegate/SKILL.md
#    （上游官方原版：plugins/open-code-review/skills/open-code-review-delegate/SKILL.md）

# 3) 复制到预设的 skills/ 目录（Git Bash）
cp -r docs/skills/open-code-review-delegate ~/.dsh/.agent-presets/book-study/skills/
#    PowerShell 等价：
#    Copy-Item -Recurse docs\skills\open-code-review-delegate "$HOME\.dsh\.agent-presets\book-study\skills\"

# 4) 若目标预设的 agent.cordis.yml 没有 skill-filesystem 行，添加：
#    - id: skill-filesystem
#      name: '@deepseek-ai/dsh-skill-filesystem'
#      config:
#        customSkillDirs:
#          - !!js "process.getBuiltinModule('node:url').fileURLToPath(new URL('skills/', baseUrl))"
#    （book-study 预设已含该行，通常无需修改。）

# 5) 开一个新会话 —— 技能目录在会话挂载时构建，当前已打开的会话看不到新增技能。
#    新会话技能列表应出现 open-code-review-delegate；让 Agent 读取该技能即按 7 步执行。

# 6) 冒烟验证（无需 LLM）
ocr delegate preview --repo D:\pro\src\micro\loadservice --from dev --to feature/redis_state --format json
```

**注意**：

- 前置条件仍是 `ocr` 二进制可用（第 0 节：复制为 `~/bin/ocr.exe` 或别名）。
- 本改编版与官方 SKILL.md 的差异仅为：增加 DSH/Windows 调用约定（`ocr` 完整路径
  回退、PowerShell / Git Bash 差异、`--repo` Windows 路径）、中文工作流说明；7 步
  流程、输出格式契约（`path/content/start_line/end_line/category/severity`）、
  覆盖率硬性要求均与官方一致。
- 若想多个预设共用该技能，把同一子目录复制到各预设的 `skills/` 下即可；技能归属
  预设本身，天然随预设复制/分发。

## 10. 进阶部署：CI 集成（GitHub Actions）

仓库根目录自带 `action.yml`（可复用 composite action）与
`.github/workflows/ocr-review.yml`（示例工作流）。在自己仓库接入时：

```yaml
# .github/workflows/ocr-review.yml（要点示例）
name: OpenCodeReview PR Review
on:
  pull_request_target:
    types: [opened]
permissions:
  contents: read
  pull-requests: write
jobs:
  code-review:
    runs-on: ubuntu-latest
    steps:
      - uses: Erickw87/open-code-review@main   # 或 alibaba/open-code-review@main
        with:
          llm_url: ${{ secrets.OCR_LLM_URL }}
          llm_auth_token: ${{ secrets.OCR_LLM_AUTH_TOKEN }}
          llm_model: ${{ secrets.OCR_LLM_MODEL }}
          llm_use_anthropic: 'false'
          github_token: ${{ secrets.GITHUB_TOKEN }}
```

需要的仓库 Secrets（对应环境变量）：

| Secret | 对应环境变量 | 含义 |
|---|---|---|
| `OCR_LLM_URL` | `OCR_LLM_URL` | LLM 端点，如 `https://api.deepseek.com` |
| `OCR_LLM_AUTH_TOKEN` | `OCR_LLM_TOKEN` | API Key |
| `OCR_LLM_MODEL` | `OCR_LLM_MODEL` | 模型名 |
| `OCR_LLM_USE_ANTHROPIC` | `OCR_USE_ANTHROPIC` | `true`（Anthropic）/ `false`（OpenAI 兼容） |

GitLab CI / 其他平台的示例在仓库 `examples/` 目录。

## 11. 状态与数据存放位置

| 路径 | 内容 |
|---|---|
| `~/.opencodereview/config.json` | LLM 端点、语言、遥测等配置 |
| `~/.opencodereview/rule.json` | 全局评审规则（可选） |
| `~/.opencodereview/sessions/<编码后仓库路径>/<session-id>.jsonl` | 每次评审的会话转录，供 `ocr viewer` / `ocr session` 使用 |
| `<repo>/.opencodereview/rule.json` | 项目级评审规则，可提交进仓库 |

> - OCR 只在 `~/.opencodereview/` 内写入状态；删除该目录即可干净卸载。
> - 隐私：OCR 只把你的 **diff**（及评审时可选读取的代码片段）发给配置的 LLM
>   端点；会话记录与规则文件均保存在本机。

## 12. 常见问题排查

| 现象 | 原因与解决 |
|---|---|
| `no valid LLM endpoint configured` | 尚未配置 LLM。按第 7 节补全 `(url, token, model)` 后 `ocr llm test` 验证。 |
| `ocr llm test` 返回 401 / 403 | token 错误/过期/厂商不匹配。检查 URL 协议是否与 provider 匹配：Anthropic 端点以 `/v1/messages` 结尾、`openai` 兼容端点以 `/v1` 结尾。 |
| `not a git repository` | `ocr review` 必须在 Git 工作树内运行；或传 `--repo /path/to/repo`。 |
| `[ocr] No tool calls parsed ... retrying` 直至 `Max tool requests reached` | 模型**不支持原生工具调用**（本地模型最常见，如 `deepseek-r1` 只会把工具调用写在文本里）。换支持 function calling 的模型（如 `qwen3`）；Ollama 选带 tools 标签的模型。 |
| `Max tool requests reached` | 模型用尽默认 100 轮工具调用仍未 `task_done`。可 `--max-tools <n>` 上调（只能上调）。 |
| 某些文件没被评审 | `ocr review --preview` 会给出每个文件的排除原因（binary / unsupported_ext / user_exclude / too_large 等）。 |
| 评审没有中文 | 执行 `ocr config set language 中文`。 |
| 本地模型评审很慢 | 调大超时：`ocr config set custom_providers.<name>.timeout_sec 900`；或调低 `--concurrency`。 |
| CI 运行比本地慢 | 模型限流（调低 `--concurrency`）或提示词缓存冷启动（同窗口内后续运行更快）。 |

## 13. 参考文档（仓库内）

- 快速开始：`pages/src/content/docs/zh/quickstart.md`
- 安装：`pages/src/content/docs/zh/installation.md`
- 配置：`pages/src/content/docs/zh/configuration.md`
- CLI 参考：`pages/src/content/docs/zh/cli-reference.md`
- FAQ：`pages/src/content/docs/zh/faq.md`
- 委托模式：`pages/src/content/docs/zh/integrations/delegate.md`
- CI 集成：`pages/src/content/docs/zh/integrations/ci.md`、`examples/` 目录
