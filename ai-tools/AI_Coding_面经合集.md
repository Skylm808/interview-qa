# AI Agent / AI Coding：先建立一张架构地图

> 先回答三个问题：Agent 由什么组成？一次任务怎样跑？企业里不同 Agent 应怎样落地？

## 0. 一句话心智模型

**Agent 不是模型，也不是一段 Prompt。**它是一个以模型为决策器、以工具为手脚、以 Context 为当下工作记忆、以 Harness（运行外壳）为控制器的系统。

```text
用户目标
  │
  ▼
┌────────── Agent Harness / Runtime ──────────┐
│  组装 Context → 调模型 → 校验/执行 Tool      │
│       ▲                         │           │
│       └──── Observation（新事实）┘           │
│              更新 State，直到结束             │
└─────────────────────────────────────────────┘
  │                │                 │
  ▼                ▼                 ▼
用户回答       文件/DB/业务系统改动    Trace / Eval / Audit
```

换句话说：**模型负责判断下一步；工程系统负责安全、可靠地把这一步做出来。**OpenAI 将最小 Agent 概括为 Model、Tools、Instructions；真正的生产系统还需补上 Context、状态、权限、执行环境和可观测性。[OpenAI：构建 Agent 实践指南](https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/)

---

## 1. 全景架构：脑、记忆、手脚和交通规则

```text
┌──────────────── 接入层 ────────────────┐
│ Web / CLI / IDE / API                   │
│ 鉴权、租户隔离、会话、限流、流式、取消   │
└──────────────────┬────────────────────┘
                   ▼
┌────────────── Agent Application / Harness ───────────────┐
│ ① Orchestrator / Agent Loop：调度模型、工具、子 Agent、终止 │
│ ② Context Engine：挑选并压缩本轮模型该看到的信息           │
│ ③ Model / Planner：理解目标，决定下一步 Action              │
│ ④ Policy / Guardrail：权限、预算、审批、Schema、风险控制    │
│ ⑤ Tool Executor：受控执行 API / MCP / Shell / Browser      │
│ ⑥ State / Checkpoint：保存任务进度，支持恢复和幂等          │
└──────────────┬────────────────────────────┬──────────────┘
               ▼                            ▼
┌──────────────────────────┐   ┌──────────────────────────┐
│ Knowledge & Memory        │   │ Action & Execution        │
│ RAG、文档、会话摘要、偏好 │   │ DB、RPC、SaaS、Git、Shell │
│ 向量/关键词索引、ACL      │   │ Browser、Sandbox、MCP     │
└──────────────────────────┘   └──────────────────────────┘
                         │
                         ▼
             Trace、Metrics、Eval、审计日志
```

| 部件 | 做什么 | 不能替代什么 |
| --- | --- | --- |
| **Model** | 理解自然语言、比较方案、选择下一步 | 不能直接拥有生产环境权限 |
| **Instructions** | 定义角色、边界、输出格式和常规步骤 | 不能代替程序化权限控制 |
| **Context Engine** | 从历史、State、RAG、工具结果中挑选高信号信息 | 不能把全部历史和全文档塞进 Prompt |
| **Harness / Runtime** | 运行循环、管理取消/超时/预算/恢复 | 不能假设模型计划永远正确 |
| **Tool + Executor** | 提取实时事实或改变外部系统；校验、超时、重试、审计 | 不能绕过身份、审批和幂等 |
| **State / Checkpoint** | 保存已确认参数、步骤、调用 ID、产物、审批状态 | 不等于完整聊天历史 |
| **RAG / Memory** | 提供私有知识，或保存跨轮仍有价值的已确认信息 | RAG 不适合查询实时订单或监控指标 |
| **Guardrail** | Schema、最小权限、影响范围、预算、人工审批 | 不应只是一句“请安全操作” |
| **Sandbox** | 限制代码的文件、网络、CPU、内存与时间 | 不代替鉴权和密钥管理 |
| **Trace / Eval** | 回放失败、衡量质量/成本/安全、形成回归集 | 不能只留最终回答 |

Kimi 的拆分也非常接近：推理与规划、工具与行动、记忆/上下文/知识、编排协调，以及防护/可观测/人工监督。[Kimi：智能体 AI 架构](https://www.kimi.com/resources/agentic-ai-architectures)

---

## 2. Agent Loop：一次任务究竟怎样运行？

以 Coding Agent 的需求“给订单服务增加退款资格校验，并跑测试”为例：

```text
1. Build Context：任务、项目规则、仓库摘要、工具定义、权限、预算
2. Inference：模型决定“先搜索退款入口”
3. Tool Call：search_code("refund")
4. Policy + Executor：检查只读权限，执行搜索
5. Observation：命中文件、片段、退出码；写入 State
6. Refresh Context：保留摘要，必要时补充相关文件
7. 回到第 2 步：阅读 → 计划 → 编辑 → 测试 → 读错误 → 修正
8. 结束：交付 Diff、测试证据、风险/待确认项
```

```text
Context → Model decision → Action / Tool Call → Observation → State update
   ▲                                                                  │
   └────────────────────── 未满足终止条件时循环 ─────────────────────┘
```

这就是 **Agent Loop**。它不是让模型一次“想完”，而是每拿到一个外部结果就可以调整行动。OpenAI 对 Codex 的描述也是：模型要么输出最终回复，要么请求工具；外壳执行工具并将输出加入下一轮输入，直到模型停止调用工具。[OpenAI：Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop/)

| 词 | 含义 |
| --- | --- |
| **Inference** | 一次向模型请求输出。 |
| **Agent loop** | `Context → 推理 → 工具 → Observation → 更新` 的反馈循环。 |
| **Turn** | 用户一条消息到一次可见回复；一个 Turn 可含很多 loop。 |
| **Run** | 后端一次任务执行的记录单位，有 run_id、状态、预算、trace。 |
| **Session / Thread** | 多个 Turn 共享的会话或工作空间。 |

Harness 必须有明确终止条件：Final 输出、合规的结构化结果、用户取消、最大轮数/时长/token/费用、连续失败、无进展或进入人工审批。不能只期待模型“自己会停”。

---

## 3. Workflow、ReAct、Plan-and-Execute：它们都是控制流模式

| 模式 | 谁决定下一步 | 图示 | 适用场景 | 代价 |
| --- | --- | --- | --- | --- |
| **单次 LLM** | 开发者 | `输入 → 模型 → 输出` | 摘要、抽取、分类 | 不会完成多步行动 |
| **Workflow / Prompt Chain** | 开发者预定义 | `A → 校验 → B → C` | 固定、高风险、需审计的流程 | 遇意外分支不灵活 |
| **Routing** | 分类器/LLM | `请求 → 路由 → 专用流程` | 客服、不同模型或领域分流 | 分错类会走错路径 |
| **并行 + 汇总** | 编排器 | `A,B,C 并行 → 汇总` | 互不依赖的检索/检查 | 冲突、重复、成本 |
| **ReAct** | 模型在循环中决定 | `Reason → Act → Observe → …` | 边查边判断、路径未知 | 可能循环、乱选工具 |
| **Plan-and-Execute** | 模型先计划，执行中可重规划 | `Plan → Step → Replan` | 长而可拆的任务 | 计划会过期，也耗 token |
| **Generator–Critic** | 生成者 + 验证者 | `生成 → 检查 → 修订` | 代码、报告、结构化交付 | 需独立验收标准 |
| **Multi-Agent** | Manager 或 handoff | `Manager → 专家 → 汇总` | 权限/工具/专业性不同，或子任务可并行 | 交接和共享状态更复杂 |
| **Human-in-the-loop** | 人握关键控制权 | `提议 → 审批 → 执行` | 付款、发版、删数据、对外发送 | 自动化速度下降 |

### Workflow 和 Agent 的根本差异

```text
Workflow：代码预先决定路径
  查订单 → 硬规则校验 → 用户确认 → 退款 API → 对账

Agent：模型依据当前状态选择路径
  缺订单号则追问；政策版本不明则检索；证据不足则转人工
```

Anthropic 的定义是：Workflow 中 LLM 和工具沿预定义代码路径运行；Agent 则由 LLM 动态决定过程和工具使用。[Anthropic：Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)

**生产中最常用的答案**：外层 Workflow 守住鉴权、硬规则、审批和写操作；内层 ReAct 处理检索、诊断、解释和候选方案。

```text
鉴权 → 资格硬校验 → [ReAct：检索/诊断/解释] → 人审 → 写操作 → 对账
```

### ReAct 是什么

ReAct = **Reason + Act**。关键不是暴露模型的“思维链”，而是让它能根据 Observation 修正下一步。

```text
Goal + State → 决定（回答/追问/调用工具/结束）
             → Act（结构化调用）
             → Observe（数据、错误、环境变化）
             → Update（State 与下一轮 Context）
```

工程上记录计划摘要、工具名、参数摘要、结果、耗时和错误码即可；不要把完整模型推理过程当作业务日志或安全依据。

### Multi-Agent 何时值得用

先用一个 Agent 跑通。仅当有明确瓶颈才拆：工具/规则太多且混淆；子任务真能并行；需要独立的研究、实现、审查角色；或权限必须隔离。OpenAI 同样建议先最大化单 Agent 能力，再考虑 Manager 或 handoff 式多 Agent。[OpenAI 指南](https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/)

Kimi Code 是典型实现：主 Agent 负责理解、规划与调工具；`explore`、`plan`、`coder` 子 Agent 在隔离 Context 工作，只把结论返回。这样保护主 Context，但会额外消耗 token。[Kimi Code：Agents and Sub-Agents](https://www.kimi.com/code/docs/en/kimi-code-cli/customization/agents)

---

## 4. Context Engineering：它不只是“写好 Prompt”

**Context 是本次模型推理实际看到的全部 token**：

```text
System / Developer Instructions
+ 用户目标和附件
+ 工具定义（名称、参数 Schema、描述）
+ 历史消息与工具结果
+ State / TODO / checkpoint
+ RAG 证据、会话摘要、项目规则、相关代码
```

| 概念 | 关心什么 | 例子 |
| --- | --- | --- |
| **Prompt Engineering** | 指令和输出约束怎么写 | “先出计划；改代码前跑测试；按 JSON 输出。” |
| **Context Engineering** | 每一轮什么信息该进入有限 Context，怎样保持高信号低噪声 | 只取相关文件和最新错误；旧日志压成摘要；按需读文档 |

上下文长不代表效果好。长任务会积累工具输出和历史，噪声会稀释关键事实。Anthropic 将 Context Engineering 定义为每轮从不断增长的信息中挑选、维护最有效 token 的策略，范围包括系统规则、工具、MCP、外部数据和消息历史。[Anthropic：Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

四个手法：

1. **Just-in-time retrieval**：只保留文件路径、文档 ID、查询条件等指针，真正需要才读取内容。
2. **Compaction**：接近 Context Window 时，压缩为目标、确认事实、已做变更、验证结果、待办与风险，再续接任务。
3. **结构化笔记**：计划、假设、实验和下一步写入 State，不靠模型从长历史里重新猜。
4. **子 Agent 隔离**：探索/评审在独立 Context 完成，主 Agent 仅接收结论和证据；代价是调用成本和信息损失。

---

## 5. Context、State、Memory、RAG：四种“记忆”不是一回事

| 名词 | 生命周期 | 保存什么 | 示例 |
| --- | --- | --- | --- |
| **Context** | 一次 inference | 此刻模型所见的精选 token | 当前需求、相关代码、工具结果摘要 |
| **State** | 一个 Run | 可恢复、可审计的任务进度 | `step=run_tests`、调用 ID、重试、审批 |
| **Memory** | 多轮/跨会话 | 已确认、未来仍有价值的信息 | 用户偏好、项目构建命令 |
| **RAG Knowledge** | 知识库生命周期 | 有来源、版本、ACL 的文档事实 | 政策、Runbook、产品文档 |

它们的关系：**State、Memory 与 RAG 都可能挑一小部分放入 Context；模型只能看 Context。**

```text
文档 → 解析/切块 → metadata（来源、版本、ACL）→ 向量/关键词索引
问题 → 权限过滤 → 召回（向量 + BM25）→ rerank → 证据进 Context → 回答/引用
```

RAG 擅长“政策怎么规定”；工具擅长“该订单现在的状态”。RAG 是 Agent 的知识组件，不是 Agent 本身。权限过滤要发生在召回前，不能先将敏感内容喂给模型。

---

## 6. Tool、Function Calling、MCP、Skill、Sandbox

| 名词 | 一句话 | 边界 |
| --- | --- | --- |
| **Tool** | 读取外部事实或执行行动的受控能力 | 如 `get_order`、`search_code`、`run_tests` |
| **Function/Tool Calling** | 模型按 Schema 生成“要调用什么、参数是什么” | 模型提议；应用执行 |
| **Tool Schema** | Tool 的机器契约 | 参数、返回、错误、数据范围与限制 |
| **MCP** | Agent 客户端连接外部 Tools、Resources、Prompts 的协议 | 不是 Tool，也不是 Agent |
| **Skill** | 针对任务可复用的操作说明/能力包 | 常含触发条件、步骤、允许工具、验收；具体格式因产品而异 |
| **Sandbox** | 受限执行环境 | 限文件、网络、命令、CPU、内存、时间 |

```text
Model：建议 call_tool({"order_id":"123"})
  ↓
Policy / Executor：Schema、身份、授权、风险、幂等键、预算校验
  ↓
Tool：真实调用订单服务
  ↓
Observation：{data / error, source, time} 回给模型
```

模型不应直接持有万能数据库账号、生产 Shell 或长期密钥。写操作需要影响范围展示、幂等键、执行后核验和必要的人审。

---

## 7. Coding Agent：同一架构换成软件工程工具

```text
任务 / Issue
  → Context：项目规则、相关代码、Git 状态、测试命令
  → Loop：搜索 → 阅读 → 计划 → 编辑 → 构建/测试 → 读错误 → 修复
  → 交付：Diff + 测试/构建证据 + 风险或待确认项
```

| 能力 | 需要的工程组件 |
| --- | --- |
| 认识仓库 | 文件/符号搜索、读取文件、Git diff、项目规则 |
| 做出改动 | 受权限控制的编辑、分支/工作区隔离 |
| 自我验证 | Sandbox 中运行测试、编译、格式化、静态检查 |
| 安全交付 | Diff、命令记录、checkpoint、CI/人工 review 关卡 |

Claude Code 的实践与此对应：先 explore、再 plan、再 code/commit；可验证的改动可先写失败测试再实现；将项目规则与命令写入 `CLAUDE.md`；通过工具允许列表控制影响范围。[Anthropic：Claude Code best practices](https://www.anthropic.com/engineering/claude-code-best-practices)

**Harness** 在 Coding Agent 语境尤其重要：它就是模型外的运行脚手架，负责循环、工具、Context 管理、权限、Sandbox、状态和用户交互。OpenAI 对 Codex 也将 agent loop 与执行逻辑称为 harness。[OpenAI：Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop/)

---

## 8. 企业 Agent 一：Enterprise RAG / Knowledge Agent

### 要解决的并不是“聊天”，而是可信知识问答

目标例子：员工问“上海办公室报销住宿上限是多少？请给出政策版本和原文依据。”系统必须回答、引用、遵守权限，并在证据不够时拒答或转人工。

```text
                 离线知识管道
文档/Confluence/Drive/工单
  → 解析/OCR → 清洗、切块 → metadata + ACL + version
  → 向量索引 + BM25 倒排索引 → 定时增量更新/删除

                 在线问答管道
用户 + user/tenant/role
  → Query rewrite（可选）
  → ACL filter → hybrid retrieve → rerank
  → evidence sufficiency gate
  ├─ 充分：证据 + 回答约束 → LLM → 带引用回答
  └─ 不足：澄清 / 拒答 / 转人工 / 调实时业务 Tool
```

### 推荐架构：**Workflow 为主，局部 Agent 为辅**

- 检索、ACL、rerank、引用格式与置信度阈值应该是确定性 Workflow；这保证合规、可复现、易评测。
- 仅在“用户问题模糊、需多跳查找、需决定是否调用实时系统”时，用小范围 ReAct。
- 不要让 Agent 自己决定能否绕过权限，也不要把“没有召回到”伪装成“政策不存在”。

### 面试官会在意的关键点

| 维度 | 亮点回答 |
| --- | --- |
| 权限 | 文档、chunk、索引三层继承 ACL；**检索前过滤**，而非生成后脱敏。 |
| 新鲜度 | metadata 有来源、版本、更新时间、失效/删除事件；回答展示引用版本。 |
| 正确性 | 先判断 evidence sufficiency；答案每个关键断言能映射到 chunk；证据不足必须拒答。 |
| 检索 | 混合召回：向量解决语义，BM25/关键词解决错误码、制度编号；再 rerank。 |
| 实时性 | 政策走 RAG，余额/订单/审批状态走权限受控的实时 Tool，绝不只查向量库。 |
| 评测 | Recall@k、MRR/nDCG、上下文相关性、引用正确性、答案忠实度、拒答正确率、ACL 越权率。 |

一个能加分的细节：**答案正确率不是唯一指标。**“检索到了正确 chunk 但模型编造了结论”和“模型说对了但引用错了”是两种不同故障，应分别监控 retrieval、grounding 与 generation。

---

## 9. 企业 Agent 二：On-call / Incident Response Agent

### 目标：缩短定位时间，不是让模型直接改生产

适合的任务是告警分诊、关联变更、汇总日志/指标/trace、推荐 Runbook、生成事件时间线和处置建议。高风险动作必须有硬规则和人审。

```text
Alert / Pager / Ticket
  → Event normalizer（服务、环境、时间窗、severity、dedup）
  → 固定 Workflow：拉 Metrics + Logs + Traces + 最近 Deploy/Config + CMDB
  → RAG：召回相关 Runbook、SLO、历史事故
  → ReAct Diagnose Agent：提出假设，选择只读查询，形成证据链
  → Policy Gate
       ├─ 建议/摘要：直接输出
       ├─ 低风险预批准动作：受限工具执行 + 验证
       └─ 回滚/扩缩容/改配置：人审批准 + 幂等执行 + 事后验证
  → Incident timeline、工单更新、复盘材料
```

### 推荐架构：**确定性采集 Workflow + 只读 ReAct 诊断 + 审批式执行**

不要以为 On-call 就应该“全自动”。第一阶段让 Agent 只读、做证据整合和建议，往往已能显著缩短 MTTR；第二阶段才对少数可逆、幂等、低风险操作开放自动化。

### 关键设计

| 难点 | 设计答案 |
| --- | --- |
| 告警噪声 | 先做去重、聚类、拓扑关联和事件归一化，再让模型看；不要把上千条原始告警直接塞 Context。 |
| 时序证据 | 所有查询带统一 incident time window；将 deploy、指标突变、错误 trace 放入同一 timeline。 |
| 幻觉 | 结论必须附证据链接、查询时间和来源；区分“观测事实”“推测假设”“建议动作”。 |
| 工具安全 | 查询工具默认只读、限制时间窗和返回量；执行工具按风险分级、最小权限、幂等键、审批。 |
| 处置闭环 | 执行后重新拉 SLI/SLO、错误率、延迟验证；不能以 API 200 当作事故已恢复。 |
| 评测 | MTTD/MTTR、分诊准确率、根因候选命中率、Runbook 推荐命中、人工采纳率、危险动作拦截率。 |

**加分表达**：把 Agent 的输出设计成 `事实 → 假设 → 建议 → 风险/批准条件` 四段，而不是一段自然语言。“CPU 在 10:03 上升”是事实；“可能内存泄漏”是待验证假设；二者不能混写。

---

## 10. 企业 Agent 三：Data / Analytics Agent

### 目标：让业务问题变成可验证的数据结论

“上周华东新用户转化为什么下降？”不是生成 SQL 这么简单：要理解指标口径、选择正确数据域、做权限控制、验证 SQL、解释异常，并保存可复现证据。

```text
自然语言问题
  → 语义层 / 指标目录（Metric definitions、维度、血缘、Owner、ACL）
  → Router：问答 / 指标查询 / 探索分析 / 报表生成
  → Plan：需要哪些指标、切分维度、比较基线、假设
  → Safe SQL Agent（只读、allowlist、语法/成本/权限校验）
  → Warehouse / BI semantic layer
  → Result validator（空值、异常、口径、总量、时间范围）
  → Analyst Agent：解释 + 图表 + SQL/口径/数据版本引用
  → 必要时 Drill-down loop；产出可复现报告
```

### 推荐架构：**语义层约束的 Plan-and-Execute，SQL 生成与解释分离**

- 指标定义、数据集选择、权限和查询成本是确定性关卡，不能只写在 Prompt。
- 模型应优先调用“查询 GMV、按地区分组”这类**语义工具**，而非任意生成裸 SQL。
- 如果必须 SQL：只读凭证、表/列 allowlist、参数化、行数/扫描量上限、`EXPLAIN` 或 dry-run、查询超时与审计。
- 执行结果要由独立 validator 检查，避免把空结果、重复 join、时区错位和指标口径错配解释成业务洞察。

### 关键设计

| 难点 | 设计答案 |
| --- | --- |
| 指标口径 | 建立 metrics semantic layer：名称、公式、grain、时间语义、owner、版本、允许维度；LLM 从中选，不自由猜。 |
| Text-to-SQL | 生成前先 schema linking，生成后 parser/AST 校验、成本预估、权限检查，再执行。 |
| 数据泄露 | 行/列级权限、PII 掩码、租户过滤由数据层强制，不能信任 LLM 自己附 `WHERE tenant_id`。 |
| 正确性 | 将 SQL、指标版本、过滤条件、时间范围、数据快照/执行时间连同图表输出，保证可复现。 |
| 洞察质量 | 用“事实、比较基线、可能原因、待验证实验”结构输出，避免把相关性说成因果。 |
| 评测 | SQL execution accuracy、指标口径正确率、权限违规率、成本、可复现率、人工分析师采纳率。 |

**加分表达**：Data Agent 的核心资产不是模型，而是可靠的 **semantic layer + 数据治理 + 查询沙箱**。没有这三样，Text-to-SQL 越会写，企业风险越大。

---

## 11. 三类企业 Agent 的选型对照

| Agent | 主控制流 | Agent 自主性 | 真相来源 | 最重要的 Guardrail |
| --- | --- | --- | --- | --- |
| Enterprise RAG | 检索 Workflow，局部 ReAct | 低到中 | 带 ACL/版本的文档证据 | 召回前 ACL、证据充分性、引用忠实度 |
| On-call | 采集 Workflow + 只读 ReAct + 审批执行 | 中 | 实时 metrics/logs/traces + Runbook | 最小权限、风险分级、人审、执行后验证 |
| Data Agent | Plan-and-Execute + 查询校验 Workflow | 中 | 语义层、数仓、BI 数据集 | 行列权限、成本上限、口径和 SQL 校验 |

共同原则：**把不可变的业务规则、权限和风险边界放在代码/数据层；把需要理解、规划、解释的部分交给模型。**这比“用一个大 Prompt 让模型什么都做”更可靠。

---

## 12. 术语速查表

| 术语 | 面试时的解释 |
| --- | --- |
| **Agent** | 模型动态决定步骤和工具、在边界内完成多步目标的系统。 |
| **Agentic system** | 广义的带 LLM 的行动系统，既可包含 Workflow，也可包含 Agent。 |
| **Harness / Runtime / Orchestrator** | 名称略有区别：Harness 偏 Coding Agent 外壳，Runtime 偏任务生命周期，Orchestrator 偏控制流调度；都在模型之外。 |
| **Planning** | 把目标暂时拆为步骤，且必须能随新证据重规划。 |
| **Observation** | Tool、环境或子 Agent 返回的新事实，包括成功、错误与元数据。 |
| **Context Window** | 单次推理可处理的 token 上限，不是无限记忆。 |
| **RAG** | 检索证据后再生成；解决“知道什么”，不等于“如何行动”。 |
| **Memory** | 跨轮/跨会话的已确认信息，需要来源、权限和过期策略。 |
| **State / Checkpoint** | 一个任务的可恢复账本：计划、完成步骤、调用 ID、审批与重试。 |
| **Guardrail** | 输入/输出/工具/权限/预算的程序化防护。 |
| **Human-in-the-loop** | 高影响动作前由人审查、批准、拒绝或接管。 |
| **Eval** | 用固定样本和判分标准评估质量、工具选择、安全与成本，并回归 bad case。 |

## 推荐复习顺序

1. 复述第 0、1 节：模型是脑，工具是手脚，Context 是工作记忆，Harness 是受控外壳。
2. 用第 2 节讲清 `Context → Model → Tool → Observation → Context`。
3. 说明 Workflow 是代码定路径，ReAct 是模型定下一步，生产中通常混用。
4. 分清 Context、State、Memory、RAG 的边界。
5. 任选第 8～10 节的一个企业 Agent，按“目标、架构、风险、评测”讲完整方案。

## 参考资料（官方）

- [OpenAI：A practical guide to building agents](https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/)
- [OpenAI：Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop/)
- [Anthropic：Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- [Anthropic：Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Anthropic：Claude Code best practices](https://www.anthropic.com/engineering/claude-code-best-practices)
- [Kimi：智能体 AI 架构在实践中的运作方式](https://www.kimi.com/resources/agentic-ai-architectures)
- [Kimi Code：Agents and Sub-Agents](https://www.kimi.com/code/docs/en/kimi-code-cli/customization/agents)
