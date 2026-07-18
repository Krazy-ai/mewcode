# Mewcode

Mewcode 是一个使用 Python 开发的终端 AI 编程助手。它以大语言模型为决策核心，通过文件读写、代码搜索、Shell、Skill、子 Agent、Git Worktree、MCP 等能力完成代码分析与修改，并在执行前经过权限规则和安全检查。

项目同时提供三种使用入口：

- 基于 Textual 的交互式终端界面（TUI）
- 适合脚本和 CI 的非交互模式
- 通过 WebSocket 连接浏览器页面的远程模式

> 当前包版本由 `pyproject.toml` 声明为 `0.2.0`，要求 Python 3.11 或更高版本。

## 核心能力

- **流式 Agent 循环**：流式接收模型的思考、文本和 Tool Call，并将执行结果继续反馈给模型，直到任务结束。
- **多模型协议**：支持 Anthropic、OpenAI 和 OpenAI-compatible 接口，可配置模型、上下文窗口、思考模式及最大输出长度。
- **内置编码工具**：支持文件读取、写入、局部编辑、Glob、Grep 和 Bash；参数由 Pydantic 模型生成 JSON Schema 并在执行前校验。
- **按需工具发现**：延迟工具默认不把完整 Schema 放入上下文，模型通过 `ToolSearch` 检索并激活，降低工具较多时的 Token 开销。
- **Skill 扩展**：从项目级或用户级目录加载 Markdown Skill，支持内联执行、Fork 执行、参数替换和热重载。
- **子 Agent 与团队协作**：支持前台、后台、Fork、Worktree 隔离等执行方式，并提供共享任务、消息邮箱和 Agent Team 管理能力。
- **分层上下文治理**：先压缩或外置大型工具结果，接近模型上下文上限时再对早期对话做摘要压缩。
- **持久化记忆和会话**：保存会话记录、项目记忆、用户偏好与文件变更快照，支持恢复会话和回退检查点。
- **权限与安全控制**：组合权限模式、路径边界、规则文件、危险命令检测、生命周期 Hook 和可选 OS 沙箱。
- **MCP 集成**：可连接 stdio 或 HTTP MCP Server，将外部工具注册到统一工具表。
- **项目指令注入**：自动发现 `MEWCODE.md`、`AGENTS.md` 等项目规则，并支持 `@./path` 形式的文件引用。

## 工作原理

```text
用户输入
   │
   ├─ 加载项目指令、Skill/Agent 目录、长期记忆和环境信息
   │
   ▼
Agent 构建 System Prompt 与当前会话
   │
   ├─ 工具结果预算 / 上下文自动压缩
   ├─ 生命周期 Hook
   └─ 按协议转换 Tool Schema
   │
   ▼
LLM 流式返回文本或 Tool Call
   │
   ├─ 无 Tool Call ───────────────► 返回最终结果
   │
   └─ 有 Tool Call
         │
         ├─ 权限规则、危险命令、路径和沙箱检查
         ├─ 执行内置工具、MCP 工具、Skill 或子 Agent
         └─ Tool Result 写回会话 ─► 进入下一轮
```

Agent 的核心循环位于 `mewcode/agent.py`。它负责环境注入、自动 compact、模型流式调用、事件分发、工具执行、权限交互、失败重试和会话结束处理。UI、非交互入口和远程服务只消费同一套 Agent 事件，因此核心执行逻辑不依赖具体界面。

## 环境要求

- Python `>= 3.11`
- 一个可用的 Anthropic、OpenAI 或 OpenAI-compatible 模型服务
- Git（使用 Worktree 或部分多 Agent 隔离功能时需要）
- 可选：`uv`，用于更快地创建环境和安装依赖
- 可选：Node.js/npm，使用依赖 `npx` 启动的 MCP Server 时需要
- 可选：Linux 的 `bubblewrap` 或 macOS 的 `sandbox-exec`，用于 OS 级命令沙箱

## 安装

推荐使用 `uv`：

```bash
git clone <repository-url>
cd mewcode
uv sync
```

也可以使用标准虚拟环境：

```bash
python -m venv .venv
```

Windows PowerShell：

```powershell
.\.venv\Scripts\Activate.ps1
pip install -e .
```

Linux/macOS：

```bash
source .venv/bin/activate
pip install -e .
```

## 快速开始

### 1. 创建配置

Mewcode 会依次读取以下文件，后面的配置层覆盖或补充前面的配置层：

1. `~/.mewcode/config.yaml`
2. `<项目目录>/.mewcode/config.yaml`
3. `<项目目录>/.mewcode/config.local.yaml`

可以从示例文件开始：

```powershell
New-Item -ItemType Directory -Force .mewcode
Copy-Item mewcode\config.yaml.example .mewcode\config.yaml
```

最小配置示例：

```yaml
providers:
  - name: anthropic
    protocol: anthropic
    base_url: https://api.anthropic.com
    api_key: ""
    model: your-model-name
    thinking: true

permission_mode: default
```

`api_key` 为空时，Anthropic 协议读取 `ANTHROPIC_API_KEY`，OpenAI 和 OpenAI-compatible 协议读取 `OPENAI_API_KEY`。建议通过环境变量提供密钥，不要把真实密钥提交到仓库。

Windows PowerShell：

```powershell
$env:ANTHROPIC_API_KEY = "your-api-key"
```

Linux/macOS：

```bash
export ANTHROPIC_API_KEY="your-api-key"
```

### 2. 启动交互界面

使用 `uv`：

```bash
uv run mewcode
```

安装为可编辑包后：

```bash
mewcode
```

Mewcode 默认以当前目录作为工作目录，所以应在需要分析或修改的项目根目录启动。

## 命令行用法

```text
mewcode [--mode MODE] [-p PROMPT] [--output-format FORMAT] [--remote]
```

### 交互模式

```bash
mewcode
mewcode --mode plan
mewcode --mode acceptEdits
```

### 非交互模式

直接输出最终文本：

```bash
mewcode -p "分析当前项目的入口和核心模块"
```

以 NDJSON 持续输出流式事件，适合脚本消费：

```bash
mewcode -p "运行测试并解释失败原因" --output-format stream-json
```

### 远程模式

```bash
mewcode --remote
```

启动后访问 `http://localhost:18888`。服务端通过 WebSocket 将 Agent 事件、工具结果和权限请求发送给浏览器。

## Provider 配置

每个 Provider 支持以下主要字段：

| 字段 | 说明 |
| --- | --- |
| `name` | Provider 名称 |
| `protocol` | `anthropic`、`openai` 或 `openai-compat` |
| `base_url` | API 基础地址 |
| `api_key` | API Key；为空时读取对应环境变量 |
| `model` | 模型名称 |
| `thinking` | 是否启用思考模式 |
| `context_window` | 可选，显式指定上下文窗口 |
| `max_output_tokens` | 可选，显式指定最大输出 Token |

上下文窗口按“显式配置 → Provider 返回值 → 内置模型映射 → 默认值”的顺序解析。对未知模型，Claude 名称默认按 200K 处理，其他模型默认按 128K 处理。

OpenAI-compatible 示例：

```yaml
providers:
  - name: local-openai-compatible
    protocol: openai-compat
    base_url: http://localhost:8000/v1
    api_key: ""
    model: your-model-name
```

## 权限模式

| 模式 | 读取 | 写入 | 命令 | 适用场景 |
| --- | --- | --- | --- | --- |
| `default` | 自动允许 | 询问 | 询问 | 日常交互，默认推荐 |
| `acceptEdits` | 自动允许 | 自动允许 | 询问 | 已确认允许修改代码 |
| `plan` | 自动允许 | 询问 | 询问 | 以分析和计划为主 |
| `bypassPermissions` | 自动允许 | 自动允许 | 自动允许 | 完全受信任的隔离环境 |

权限判断不只依赖模式。工具执行还会经过只读命令识别、危险命令检测、路径边界、YAML 规则、会话授权、Hook 以及可选 OS 沙箱。

可以在以下位置配置权限规则：

- `~/.mewcode/permissions.yaml`
- `<项目目录>/.mewcode/permissions.yaml`
- `<项目目录>/.mewcode/permissions.local.yaml`

`bypassPermissions` 会绕过正常询问，只有在容器、临时 Worktree 等明确隔离且可信的环境中才应使用。

## 内置工具与工具发现

默认工具表包含：

| 工具 | 用途 |
| --- | --- |
| `ReadFile` | 按范围读取文本文件 |
| `WriteFile` | 创建或覆盖文件 |
| `EditFile` | 对文件做精确局部替换 |
| `Bash` | 执行 Shell 命令 |
| `Glob` | 按文件名模式查找文件 |
| `Grep` | 按内容搜索代码 |

运行入口还会按功能配置注册 `LoadSkill`、`InstallSkill`、`ToolSearch`、`Agent`、Worktree、Team 和交互询问等高层工具。标记为 deferred 的工具只在目录中暴露名称，不立即发送完整 Schema；模型调用 `ToolSearch` 后才会将选中的 Schema 加入后续请求。

所有工具继承统一的 `Tool` 抽象，并通过 Pydantic 参数模型完成两件事：

1. 使用 `model_json_schema()` 生成发送给模型的 JSON Schema。
2. 使用 `model_validate()` 校验模型返回的参数并执行类型转换。

新增工具时通常只需要定义参数模型、工具元信息和异步 `execute()` 方法，再注册到 `ToolRegistry`。

## Skill 扩展

Skill 的搜索优先级为：

1. `<项目目录>/.mewcode/skills`（最高）
2. `~/.mewcode/skills`
3. 内置 Skill（当前实现未提供内置 Skill）

支持三种组织形式：

- `.mewcode/skills/review.md`
- `.mewcode/skills/review/SKILL.md`
- `.mewcode/skills/review/skill.yaml` + `prompt.md`

单文件 Skill 示例：

```markdown
---
name: explain-code
description: 解释指定代码的职责、流程和边界条件
mode: inline
context: full
---

请分析下面的代码或文件：

$ARGUMENTS
```

主要字段：

| 字段 | 说明 |
| --- | --- |
| `name` | 必填，小写字母开头，可包含数字和连字符 |
| `description` | 必填，用于目录展示和能力匹配 |
| `mode` | `inline` 或 `fork` |
| `model` | 可选，为 Fork Skill 指定模型 |
| `context` | `full`、`recent` 或 `none` |

`$ARGUMENTS` 会被替换为调用 Skill 时的用户参数。如果正文中没有该占位符，参数会作为 `User Request` 追加到正文末尾。

## 自定义 Agent 与多 Agent

自定义 Agent 的优先级同样是“项目级 → 用户级 → 内置”：

- `<项目目录>/.mewcode/agents/*.md`
- `~/.mewcode/agents/*.md`
- `mewcode/agents/builtins/*.md`

示例：

```markdown
---
name: dependency-reviewer
description: 分析依赖升级带来的兼容性和安全风险
tools: [ReadFile, Glob, Grep, Bash]
disallowedTools: [WriteFile, EditFile]
model: inherit
maxTurns: 30
permissionMode: default
background: false
isolation: ""
---

你只负责分析依赖变化并给出证据，不修改文件。
```

内置 Agent 包括探索、通用执行和规划等角色；Verification Agent 需要通过配置开启。主 Agent 可以用 `Agent` 工具启动子 Agent，并选择前台、后台、Fork 或 Worktree 隔离方式。团队模式在此基础上增加共享任务列表、消息邮箱、追踪树和团队生命周期管理。

相关开关：

```yaml
enable_fork: true
enable_verification_agent: false
enable_coordinator_mode: false
teammate_mode: ""
```

Worktree 默认会复用 `node_modules`、`.venv` 和 `vendor` 等大型目录，并定期清理陈旧 Worktree。配置项如下：

```yaml
worktree:
  symlink_directories: [node_modules, .venv, vendor]
  stale_cleanup_interval: 3600
  stale_cutoff_hours: 24
```

## MCP 集成

stdio MCP Server：

```yaml
mcp_servers:
  - name: context7
    command: npx
    args: ["-y", "@upstash/context7-mcp"]
```

HTTP MCP Server：

```yaml
mcp_servers:
  - name: internal-tools
    url: https://example.com/mcp
    headers:
      Authorization: "Bearer ${MCP_TOKEN}"
```

MCP 的 `headers` 和 `env` 支持 `${ENV_NAME}` 环境变量替换。stdio 子进程只继承 `PATH` 和配置中显式声明的环境变量，避免无意中向第三方进程暴露全部宿主环境。

## 项目指令与自定义命令

Mewcode 会把以下指令文件合并进 Agent 上下文：

1. `~/.mewcode/MEWCODE.md`、`~/.mewcode/AGENTS.md`
2. 从 Git 根目录到当前工作目录链路上的 `MEWCODE.md`、`AGENTS.md`
3. `<项目目录>/.mewcode/INSTRUCTIONS.md`
4. `<项目目录>/MEWCODE.local.md`

指令文件支持 `@./relative/path`、`@../path`、`@~/path`、`@/absolute/path` 和兼容格式 `@include path`。实现包含循环检测、代码块跳过和最大五层递归限制。

自定义斜杠命令从以下目录加载，项目级覆盖用户级：

- `~/.mewcode/commands`
- `<项目目录>/.mewcode/commands`

例如 `.mewcode/commands/git/log.md` 会注册为 `/git:log`。Markdown frontmatter 可声明 `description`、`argument-hint` 和 `aliases`，正文也支持 `$ARGUMENTS`。

常用内置命令包括：

| 命令 | 用途 |
| --- | --- |
| `/help` | 查看命令帮助 |
| `/status` | 查看模型、权限和 Token 状态 |
| `/clear` | 清空当前对话 |
| `/compact` | 手动压缩当前上下文 |
| `/memory` | 查看和管理记忆 |
| `/session` | 列出、恢复、新建或删除会话 |
| `/permission` | 查看或切换权限模式 |
| `/plan` | 进入或退出计划模式 |
| `/review` | 发起代码变更审查 |
| `/rewind` | 回退到文件历史检查点 |
| `/skill` | 列出、查看或重载 Skill |
| `/mcp` | 查看 MCP 连接状态 |
| `/sandbox` | 查看或切换沙箱设置 |
| `/tasks` | 查看或取消后台任务 |
| `/trace` | 查看 Agent 父子追踪树 |
| `/worktree` | 创建、查看、进入或退出 Worktree |

## 上下文压缩

Mewcode 采用两层上下文治理：

### 第一层：工具结果预算

模型调用前会检查历史 Tool Result 的体积。超出预算的大结果会写入：

```text
<项目目录>/.mewcode/session/tool-results/
```

发送给模型的副本使用包含文件路径的占位内容替代原文，原始 `ConversationManager` 历史不会因此被直接改写。替换记录会持久化，便于后续恢复和审计。

### 第二层：对话 compact

当真实 Token 用量接近模型上下文上限时，Agent 调用模型总结较早的对话，只保留摘要和近期消息。压缩边界写入会话 JSONL，恢复会话时会从最近的边界重建上下文。用户也可以通过 `/compact` 手动触发。

这两层分别解决“单个工具结果过大”和“长期对话累计过长”的问题。

## 记忆、会话与文件历史

### 长期记忆

长期记忆按内容类型分为两层：

- 用户级：`~/.mewcode/memory/`，保存 `user`、`feedback` 类型，跨项目复用。
- 项目级：`<项目目录>/.mewcode/memory/`，保存 `project`、`reference` 类型。

每条记忆是带 YAML frontmatter 的独立 Markdown 文件，`MEMORY.md` 只保存简短索引。Agent 会把索引注入上下文，并根据任务按需读取具体记忆文件。项目还包含自动提炼和周期性 consolidation 逻辑，用于从会话中更新记忆资产。

### 会话

会话记录保存在：

```text
<项目目录>/.mewcode/sessions/
```

每个会话使用 `.jsonl` 保存消息和压缩边界，使用 `.meta` 保存标题、时间等元信息。`/session` 可恢复或管理历史会话。

### 文件历史

Agent 修改文件时可在以下目录保存版本快照：

```text
<项目目录>/.mewcode/file-history/<session-id>/
```

`/rewind` 使用这些快照将工作区回退到某个会话检查点，它不是 `git reset` 的替代品，也不会自动处理所有外部文件变化。

## Hook

Hook 可以在 Agent 生命周期节点执行命令或注入提示。支持的事件包括：

- `session_start`、`session_end`
- `turn_start`、`turn_end`
- `pre_send`、`post_receive`
- `pre_tool_use`、`post_tool_use`

`pre_tool_use` Hook 可以拒绝一次工具调用，因此适合放置项目级策略和额外安全检查。其他 Hook 可用于格式化、通知、日志记录或动态提示。Hook 配置位于 `config.yaml` 的 `hooks` 列表中。

## OS 沙箱

沙箱默认关闭：

```yaml
sandbox:
  enabled: false
  auto_allow: false
  network_enabled: false
```

- macOS 使用 Seatbelt（`sandbox-exec`）。
- Linux 使用 bubblewrap（`bwrap`）。
- Windows 当前没有 OS 级沙箱实现，但路径权限、危险命令检测和人工确认仍然有效。

`auto_allow` 表示当命令受到可用 OS 沙箱约束时允许自动执行；`network_enabled` 控制沙箱内网络访问。沙箱不可用时不要把它当作权限系统的替代品。

## 项目结构

```text
mewcode/
├── __main__.py             # CLI、非交互和远程模式入口
├── app.py                  # Textual TUI 与事件展示
├── agent.py                # Agent 主循环、工具执行与流式事件
├── client.py               # Anthropic/OpenAI 客户端适配
├── config.py               # 分层配置加载与合并
├── conversation.py         # 对话消息和环境注入
├── context/                # 工具结果预算与自动 compact
├── tools/                  # 内置工具、高层工具与 ToolRegistry
├── skills/                 # Skill 解析、加载、安装与执行
├── agents/                 # 子 Agent 定义、Fork 与工具过滤
├── teams/                  # Agent Team、共享任务和消息邮箱
├── worktree/               # Git Worktree 生命周期
├── permissions/            # 权限模式、规则和路径边界
├── sandbox/                # Seatbelt/bubblewrap OS 沙箱
├── memory/                 # 指令、长期记忆、会话和 consolidation
├── hooks/                  # 生命周期 Hook
├── mcp/                    # MCP 连接与工具适配
├── commands/               # 内置及自定义斜杠命令
├── filehistory/            # 文件修改快照与 rewind
├── remote.py               # HTTP/WebSocket 远程服务
└── serialization.py        # 事件和数据序列化

tests/                      # 单元测试与集成测试
pyproject.toml              # 包元数据、依赖和 CLI 入口
uv.lock                     # uv 锁定依赖
IMPLEMENTATION_TASKS.md     # 后续能力建设任务清单
```

## 开发与测试

安装开发依赖：

```bash
uv sync --group dev
```

运行全部测试：

```bash
uv run pytest
```

运行单个测试文件：

```bash
uv run pytest tests/test_agent.py -q
```

项目使用 `pytest-asyncio` 测试异步 Agent、工具、Hook 和权限流程。新增功能时应优先补充边界测试，尤其是工具参数错误、权限拒绝、路径逃逸、超时、取消、上下文压缩和会话恢复。

## 安全注意事项与已知边界

1. **非交互模式会自动批准询问型权限请求。** 当前 `mewcode -p` 在收到 `PermissionRequest` 后会返回 `ALLOW`。在 CI 中使用前，应通过权限规则、受限账户、容器或只读工作区建立外部边界，不要仅依赖交互确认。
2. **远程服务当前没有身份认证。** `--remote` 默认监听 `0.0.0.0:18888`，能够访问该端口的客户端可能操作 Agent。只应在可信网络或通过防火墙、反向代理认证和端口转发进行隔离后使用。
3. **Windows 没有 OS 级命令沙箱。** Windows 上应更加依赖权限规则、路径边界和人工确认。
4. **会话和工具结果可能包含敏感信息。** `.mewcode/sessions`、`.mewcode/session/tool-results`、`.mewcode/memory` 和 debug 日志应按项目敏感级别决定是否加入 `.gitignore`、加密或定期清理。
5. **模型输出不等于可信指令。** MCP 返回值、网页内容、仓库文件和命令输出都可能包含 Prompt Injection。高风险写入、命令和外部操作仍应经过规则或人工审查。
6. **多 Agent 会放大资源消耗。** 后台 Agent、Team 和 Worktree 会增加模型调用、磁盘占用和并发进程，使用时应设置明确任务边界并及时清理。

## 当前实现边界

- Skill 插件源接口保留，但插件加载尚未实现；内置 Skill 列表当前为空。
- OS 沙箱仅覆盖 macOS 和 Linux，且依赖宿主机相应组件可用。
- 远程模式适合本地或受保护网络，不是开箱即用的多租户服务。
- 这是一个能够真实读写文件和执行命令的开发工具，运行前应先检查配置、权限模式和工作目录。

