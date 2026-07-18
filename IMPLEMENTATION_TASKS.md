# Mewcode 能力增强任务清单

> 状态：仅规划，尚未开始实现。
>
> 目标：保留 Mewcode 的异步事件流、Pydantic Tool、Context Compact、Session、Permission、OS Sandbox、Worktree 和 Agent Team，在此基础上吸收 Minicode 的分层 Skill 路由、知识库 RAG、内容寻址 Artifact、集中安全审计，并预留 SWE-bench-live 评测能力。

## 0. 实施原则与基线

- [ ] 记录当前 Python、依赖、操作系统和 Git commit。
- [ ] 在不修改代码的情况下运行完整测试，记录通过、跳过、失败和错误数量，作为回归基线。
- [ ] 记录关键性能基线：启动耗时、基础 Prompt Token、一次 ReadFile 的上下文增长量。
- [ ] 新能力默认关闭或保持向后兼容，不能改变现有配置的行为。
- [ ] 新工具继续使用异步 `Tool.execute()`、Pydantic 参数模型和 `ToolRegistry` 注册机制。
- [ ] 不替换现有 PermissionChecker、OS Sandbox、Compact、Session 和 TeamManager，只在现有链路上扩展。
- [ ] 所有持久化文件采用 UTF-8、原子写入，并限制在项目 `.mewcode/` 或用户配置目录中。
- [ ] 为新增模块补齐单元测试、集成测试、恢复测试和异常路径测试。

## 1. 内容寻址 Artifact 与大型工具结果治理

### 1.1 数据模型与存储

- [ ] 新增 `mewcode/context/artifacts.py`。
- [ ] 定义 `ArtifactMetadata`：ID、工具名、内容类型、字符数、创建时间、来源 Tool Call、预览和存储路径。
- [ ] 使用 `sha256(tool_name + "\\0" + content)` 生成稳定 Artifact ID。
- [ ] 相同工具和内容只存储一次，避免重复占用磁盘。
- [ ] Artifact 写入项目 `.mewcode/artifacts/`，支持按 Session 引用但不依赖 Session ID 寻址。
- [ ] 使用临时文件加原子重命名，避免进程中断产生半文件。
- [ ] 增加单文件大小、总目录大小和保留时间限制。
- [ ] 增加安全路径检查，禁止通过 Artifact ID 读取存储目录之外的文件。

### 1.2 接入上下文压缩

- [ ] 改造 `context.manager.apply_tool_result_budget()`，超过阈值时写入 Artifact。
- [ ] 用稳定占位符替换正文，包含 Artifact ID、预览、大小和恢复提示。
- [ ] 保证相同工具结果产生相同占位符，提高 Prompt Cache 稳定性。
- [ ] Compact 后保留仍被引用的 Artifact ID。
- [ ] Session Resume 时恢复 Artifact 引用，不依赖旧进程内存状态。
- [ ] 防止 Artifact 读取结果再次被外置，形成无限引用循环。

### 1.3 Agent 工具

- [ ] 新增 `ReadArtifact` 工具，支持按 ID 读取、offset/limit 分页和关键词片段检索。
- [ ] 新增 `ListArtifacts` 或调试命令，默认不暴露大量正文给 LLM。
- [ ] 为 Anthropic、OpenAI Responses、OpenAI Compatible 验证工具 Schema。

### 1.4 验收标准

- [ ] 超长 Tool Result 不再完整进入对话历史。
- [ ] 相同内容重复出现时复用同一 Artifact。
- [ ] Agent 能按需恢复完整内容或指定片段。
- [ ] Compact 和 Session Resume 后仍可读取。
- [ ] 非法 ID、目录穿越、文件缺失和损坏均返回结构化错误。
- [ ] 现有 Context 测试全部保持通过。

## 2. 集中安全审计链

### 2.1 审查模型

- [ ] 新增 `mewcode/security/` 包。
- [ ] 定义 `RiskLevel`、`RiskCategory`、`ReviewDecision` 和 `ReviewFinding`。
- [ ] 风险分类至少覆盖：破坏性命令、越权路径、敏感文件、凭据泄露、网络外传、Prompt 注入和间接 Prompt 注入。
- [ ] 定义统一决策：allow、deny、ask、sanitize。
- [ ] 明确 Tool 自检接口，使工具能声明额外风险和确认要求。

### 2.2 审查链路

- [ ] 在 Tool 参数完成 Pydantic 校验后、PermissionChecker 之前运行 SecurityReviewChain。
- [ ] 对 Bash 命令进行分段和规范化，不能只做原始字符串包含判断。
- [ ] 对文件工具解析真实路径，并检查符号链接后的最终路径。
- [ ] 对网络、MCP、Hook 和 Agent Tool 统一应用数据外传检查。
- [ ] 保留原有项目/用户规则、人工确认和 OS Sandbox 作为最终执行边界。
- [ ] 非交互 `-p` 模式不得无条件绕过高风险审查；定义明确的拒绝或配置策略。

### 2.3 审计日志

- [ ] 新增脱敏 JSONL 审计日志，记录时间、Session、Tool、风险、决策和原因。
- [ ] API Key、Token、Cookie、Authorization 和常见凭据模式必须脱敏。
- [ ] 默认不记录完整文件内容和完整 Prompt。
- [ ] 支持日志轮转、大小限制和关闭开关。
- [ ] 增加查看最近安全事件的命令或调试接口。

### 2.4 验收标准

- [ ] 普通只读工具不增加人工确认噪声。
- [ ] 明显破坏性操作和越权写入在执行前被阻断。
- [ ] 外部文档中的工具诱导指令能够被标记为间接 Prompt 注入风险。
- [ ] 高风险事件写入脱敏审计日志。
- [ ] Permission、Hook、Sandbox 原有测试保持通过，并增加组合链路测试。

## 3. 分层 Skill 路由

### 3.1 Skill 元信息

- [ ] 扩展 Skill Frontmatter，兼容现有字段并新增 `tags`、`examples`、`boundaries`、`priority`、`catalog` 和 `estimated_tokens`。
- [ ] 保持旧 Skill 不修改也能正常加载。
- [ ] 支持项目、用户、内置 Skill 的优先级和同名覆盖规则。
- [ ] 建立 Skill 目录树和扁平检索索引。
- [ ] 缓存解析结果，并在文件变更后增量刷新。

### 3.2 二阶路由与精排

- [ ] 第一阶段按目录、任务意图和标签粗召回候选 Skill。
- [ ] 第二阶段按名称、描述、示例、边界、优先级和上下文匹配精排。
- [ ] 对功能重叠 Skill 做去重或降权。
- [ ] 对明确不适用的 boundary 做硬过滤。
- [ ] 支持 `top_k`、最小分数和总 Token 预算。
- [ ] 路由结果包含可解释得分和命中原因，便于调试与评测。
- [ ] 将 ToolSearch 的延迟工具候选与 Skill 候选协调，避免两个检索系统重复注入。

### 3.3 Agent 接入

- [ ] 每轮发送模型前，根据最新用户任务执行路由。
- [ ] 只向 Prompt 注入候选 Skill 的短摘要，不直接注入全部正文。
- [ ] 保留现有 LoadSkill，以及 inline/fork、model、context 执行语义。
- [ ] Skill 激活后记录来源、版本和 Token 消耗。
- [ ] 提供 `/skills explain <query>` 或等价调试命令展示路由结果。

### 3.4 验收标准

- [ ] Skill 数量增加时，基础 Prompt Token 不线性增长。
- [ ] 旧 Skill 加载与执行行为不变。
- [ ] 典型任务的正确 Skill 位于 top-k。
- [ ] 不适用 Skill 能被 boundary 排除。
- [ ] 同一次输入产生稳定、可解释的路由结果。
- [ ] 补充召回率、MRR、Token 节省量的离线小型评测集。

## 4. 知识库 RAG

### 4.1 范围与依赖

- [ ] 将 RAG 设计为可选能力，避免增加 Mewcode 基础安装的强制依赖。
- [ ] 第一阶段支持 Markdown、TXT 和代码文件；PDF、Office 放到后续可选扩展。
- [ ] 定义知识库配置：存储目录、排除规则、Chunk 策略、召回数量和最大注入 Token。
- [ ] 区分项目知识库与用户知识库。

### 4.2 文档处理与存储

- [ ] 新增 `mewcode/knowledge/` 包。
- [ ] 实现文档发现、格式解析、清洗和元信息提取。
- [ ] 实现按标题、段落、代码结构和 Token 上限切片。
- [ ] 使用内容哈希支持增量索引、更新和删除。
- [ ] 使用 SQLite 保存文档、Chunk、来源、哈希和索引状态。
- [ ] 第一阶段实现 BM25；向量检索作为可选后端。

### 4.3 检索链路

- [ ] 实现查询规范化和可选 Query Rewrite。
- [ ] 实现 BM25 召回。
- [ ] 实现可选向量召回及 Embedding Provider 接口。
- [ ] 实现混合排序、去重、来源多样性和可选 Rerank。
- [ ] 按 Token 预算生成带来源引用的上下文片段。
- [ ] 将检索结果标记为不可信外部内容，并接入间接 Prompt 注入审查。

### 4.4 工具与命令

- [ ] 新增 KnowledgeIngest、KnowledgeSearch、KnowledgeStatus 和 KnowledgeRemove 工具。
- [ ] 增加 `/knowledge` 管理命令。
- [ ] 支持显式 Tool 调用检索，自动检索默认关闭或可配置。
- [ ] 大型检索结果复用 Artifact 存储，避免直接填满上下文。

### 4.5 验收标准

- [ ] 文档新增、修改、删除后索引状态正确。
- [ ] 检索结果包含文件、位置、分数和可验证引用。
- [ ] 单个损坏文档不会导致整个索引失败。
- [ ] 不配置向量模型时，BM25 模式可独立工作。
- [ ] 检索注入严格受 Token 预算限制。
- [ ] 建立小型问答集，记录 Recall@K、MRR、延迟和 Token 成本。

## 5. SWE-bench-live 评测框架（可选后续阶段）

### 5.1 评测目录与任务模型

- [ ] 新增独立 `evaluation/swebench_live/`，不把评测依赖混入运行时核心包。
- [ ] 定义任务实例模型：instance_id、仓库、base commit、问题描述、测试信息和元数据。
- [ ] 实现数据集适配器，避免业务代码绑定单一数据源格式。
- [ ] 明确数据集许可、凭据和缓存策略，不把受限数据提交到仓库。

### 5.2 环境准备

- [ ] 实现仓库获取、指定 commit 检出和干净状态校验。
- [ ] 为每个任务创建隔离 Worktree 或容器。
- [ ] 支持任务级安装命令、环境变量、网络策略和超时。
- [ ] 默认禁止不同任务共享可变环境。
- [ ] 记录环境镜像、依赖版本和 commit，保证可复现。

### 5.3 Agent Runner

- [ ] 使用非交互 Mewcode 运行任务，固定系统提示和权限策略。
- [ ] 配置最大轮数、Token、费用、时间和工具调用预算。
- [ ] 保存 stream-json 事件、Trace、最终状态和错误原因。
- [ ] 支持单任务重试，但区分 Agent 失败和基础设施失败。
- [ ] 并发执行时限制 CPU、内存、网络和 Provider 请求速率。

### 5.4 补丁与评分

- [ ] 运行结束后提取标准化 `git diff`。
- [ ] 检查补丁为空、超限、越界文件和二进制文件。
- [ ] 在独立干净环境应用补丁并运行官方评测命令。
- [ ] 解析测试结果，输出 resolved、unresolved、infra_error 等状态。
- [ ] 保存 predictions JSONL 和逐任务报告。
- [ ] 汇总解决率、平均 Token、费用、耗时、工具调用数和失败类型。

### 5.5 验收标准

- [ ] 支持先用 1～5 个本地样例完成端到端 dry run。
- [ ] 同一任务和配置可重复运行并获得可解释结果。
- [ ] Agent 运行环境与评分环境相互隔离。
- [ ] 基础设施错误不会被错误计算为模型未解决。
- [ ] 输出格式可供后续排行榜或分析脚本消费。

## 6. 跨模块集成与质量保障

- [ ] 统一配置模型、默认值、环境变量和配置校验错误。
- [ ] Artifact、RAG、Skill Router 和安全审计均输出可观测指标。
- [ ] 增加端到端场景：Skill 路由 → RAG 检索 → Artifact 外置 → Tool 执行 → 安全审计。
- [ ] 验证 Anthropic、OpenAI Responses、OpenAI Compatible 三种协议。
- [ ] 验证交互 TUI、`-p`、stream-json 和 remote 模式。
- [ ] 验证普通 Agent、fork Subagent、Worktree Agent 和 Agent Team。
- [ ] 运行完整测试并与第 0 阶段基线比较，不允许新增非预期失败。
- [ ] 增加性能回归测试，避免基础 Prompt、启动时间和磁盘使用明显恶化。
- [ ] 更新架构文档、配置示例、迁移说明、安全边界和故障排查文档。

## 7. 推荐实施顺序

1. [ ] 建立测试与性能基线。
2. [ ] 实现 Artifact，先解决大结果和上下文成本问题。
3. [ ] 实现集中安全审计，为后续新工具建立统一边界。
4. [ ] 实现分层 Skill 路由，减少 Skill 和 Tool 增长带来的 Prompt 噪声。
5. [ ] 实现 BM25 RAG MVP，并接入 Artifact 和安全审查。
6. [ ] 增加向量检索、混合排序和 Rerank。
7. [ ] 完成跨模式回归、性能评测和文档。
8. [ ] 最后独立实现 SWE-bench-live Harness，避免评测需求反向污染核心 Agent。

## 8. 阶段性交付定义

### Milestone A：上下文与安全基础

- [ ] Artifact 完成。
- [ ] 集中安全审计完成。
- [ ] 原有测试无新增回归。

### Milestone B：能力检索

- [ ] 分层 Skill 路由完成。
- [ ] 路由离线评测和 Token 对比完成。

### Milestone C：知识库

- [ ] BM25 RAG MVP 完成。
- [ ] 可选向量与混合检索完成。
- [ ] RAG 安全与引用验证完成。

### Milestone D：Agent Benchmark

- [ ] SWE-bench-live 本地小样本端到端跑通。
- [ ] 资源、费用、结果与失败分类报告完成。

