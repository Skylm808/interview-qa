# Agent 开发：先按“层”理解，再记名词与方案

> 学 Agent 最容易乱，是因为把模型、RAG、MCP、Workflow、Sandbox、Eval 都放在同一平面记。正确方式是先有一张分层地图：**每个概念属于哪一层、解决什么问题、不能越过什么边界。**

## 0. 先记住系统边界

**Agent = 模型的动态决策能力 + 工程系统的受控执行能力。**

模型并不直接读数据库、改文件或拥有权限。它只依据本轮 Context 选择“回答、追问、调用工具或结束”；Agent Runtime/Harness 负责真正执行、记录、限制和恢复任务。

```text
用户目标
  │
  ▼
┌──────────── Agent Runtime / Harness ────────────┐
│  组装 Context → Model 决策 → 校验/执行 Tool       │
│       ▲                           │              │
│       └──── Observation（新事实）──┘              │
│          更新 State；达到终止条件才结束            │
└─────────────────────────────────────────────────┘
  │                 │                    │
  ▼                 ▼                    ▼
用户回答       真实系统中的改动         Trace / Audit / Eval
```

OpenAI 将最小 Agent 概括为 Model、Tools、Instructions；工程上还必须补齐 Context、State、权限、执行环境和可观测性。[OpenAI：A practical guide to building agents](https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/)

---

## 1. Agent 的六层架构地图

不要先背名词，先把系统按责任拆成六层。下面层与层之间是接口，不是必须拆成六个微服务。

```text
┌──────────────────────────────────────────────────────────┐
│ L0 接入与产品层：Web / IDE / API / Chat / 身份 / 会话      │
├──────────────────────────────────────────────────────────┤
│ L1 决策与编排层：Model、Instructions、Planner、Loop         │
├──────────────────────────────────────────────────────────┤
│ L2 Context 与知识层：Context Engine、State、Memory、RAG    │
├──────────────────────────────────────────────────────────┤
│ L3 工具与执行层：Function Calling、MCP、API、Shell、Sandbox │
├──────────────────────────────────────────────────────────┤
│ L4 治理与安全层：Policy、ACL、Guardrail、审批、预算、幂等   │
├──────────────────────────────────────────────────────────┤
│ L5 运营与学习层：Trace、Metrics、Eval、Bad case、灰度      │
└──────────────────────────────────────────────────────────┘
```

### L0：接入与产品层——“谁发起、谁能看、怎样交互”

| 组件 | 职责 | 边界 |
| --- | --- | --- |
| Client / IDE / CLI / API | 接收任务、展示过程与结果 | 不承担 Agent 的业务决策 |
| Auth / Tenant / Session | 确认身份、租户、会话归属 | 身份信息必须传给 L2/L4，不能只在 UI 判断 |
| Streaming / Cancel | 流式显示进度、用户中断长任务 | 取消必须通知 Runtime，不能只隐藏前端 loading |

### L1：决策与编排层——“下一步做什么”

| 组件 | 职责 | 边界 |
| --- | --- | --- |
| **Model** | 理解目标、比较选择、决定回答/工具/追问/结束 | 只产生建议或结构化 Tool Call，不直接执行 |
| **Instructions** | 角色、输出格式、常规步骤、软性约束 | 不是权限系统，也不是业务事务引擎 |
| **Planner** | 将目标临时拆为下一步或计划 | 计划可以被新证据推翻，不能当成真相 |
| **Orchestrator / Agent Loop** | 调模型、工具或子 Agent，并判断何时结束 | 不应把不确定性全部硬编码成巨大流程图 |
| **Workflow** | 开发者预定义的步骤、分支和关卡 | 适合确定流程，不等于 Agent 的动态决策 |

### L2：Context 与知识层——“模型这一次知道什么”

| 组件 | 职责 | 边界 |
| --- | --- | --- |
| **Context Engine** | 从各种信息中挑出本轮最相关、最短的 Context | 不把所有历史/全文档塞给模型 |
| **Context Window** | 单次推理可处理的 token 容量 | 它是有限注意力，不是无限记忆 |
| **State / Checkpoint** | 一个 Run 的任务账本：步骤、调用 ID、重试、审批、产物 | 不等于对话全文 |
| **Memory** | 跨轮/跨会话保留的已确认偏好或经验 | 要有 owner、来源、权限与过期策略 |
| **RAG Knowledge** | 通过检索提供有来源、版本、ACL 的知识证据 | 不用于“当前订单状态”“实时监控数据” |
| **Compaction** | 把已完成过程压为摘要，续接长任务 | 摘要应保留事实、待办和风险，不能只留结论 |

### L3：工具与执行层——“如何读世界、改变世界”

| 组件 | 职责 | 边界 |
| --- | --- | --- |
| **Tool** | 获取外部事实或执行动作 | 如查订单、搜索代码、跑测试、发消息 |
| **Function / Tool Calling** | Model 按 Schema 说“我要调用哪个工具、参数是什么” | 调用提议不等于调用已执行 |
| **Tool Schema** | 参数、返回、错误类型、范围限制的机器契约 | 描述应具体，避免工具职责重叠 |
| **MCP** | Agent 客户端连接 Tools、Resources、Prompts 的协议 | MCP 不是 Tool，也不是 Agent |
| **Skill** | 可复用任务操作手册：触发条件、步骤、工具、边界、验证 | 各产品格式不同，不要把它等同于模型能力 |
| **Sandbox** | 约束文件、网络、命令、CPU、内存、时长的环境 | 不能替代密钥管理、鉴权与审批 |

### L4：治理与安全层——“允许做什么，最多做到什么程度”

| 组件 | 职责 | 边界 |
| --- | --- | --- |
| **Policy / ACL** | 数据与工具的最小权限、租户隔离 | 必须在服务/数据层强制，而非相信 Prompt |
| **Guardrail** | 输入/输出/工具参数的程序化校验与拦截 | 不只是“请安全操作”一句话 |
| **Human-in-the-loop** | 高影响动作前展示影响、请人批准或接管 | 不把高风险审批交给同一个模型 |
| **Budget / Stop condition** | 轮数、并发、token、费用、时长、无进展限制 | 每个 Run 都要有，不是线上出问题再补 |
| **Idempotency / Verification** | 防重复写入；写后再查/对账确认结果 | API 200 不等于业务已正确完成 |

### L5：运营与学习层——“为什么成功/失败，怎样持续变好”

| 组件 | 职责 | 关键记录 |
| --- | --- | --- |
| **Trace** | 回放一条任务的全链路 | Context 来源/版本、模型、工具、结果、错误、耗时 |
| **Metrics** | 线上质量、性能、成本、安全趋势 | 完成率、延迟、token、工具成功率、拦截率 |
| **Eval** | 用固定样本和评分规则比较方案 | 任务完成、引用正确、SQL 正确、越权、成本 |
| **Bad case loop** | 失败分类、加入回归集、灰度验证修复 | 不要只改 Prompt 而不留回归样本 |

---

## 2. 信息怎样在六层之间流动：Agent Loop

以“排查订单退款失败”为例：

```text
L0 用户 + 身份
   → L2 Context：任务、当前订单号、相关政策片段、任务 State、工具定义
   → L1 Model：决定先调 get_order
   → L4：校验当前用户能否查询该订单、工具参数和预算
   → L3：执行 get_order
   → L2 Observation：订单状态与时间写入 State，生成下一轮 Context
   → L1 Model：发现政策版本不明，决定检索政策
   → …
   → L1 Final 或 L4 人审
   → L5 写 Trace / 指标 / 评测样本
```

```text
Context → Model decision → Action / Tool Call → Observation → State update
   ▲                                                                  │
   └────────────────── 未到终止条件时继续循环 ────────────────────────┘
```

这条循环叫 **Agent Loop**。OpenAI 对 Codex 的描述也正是：模型给最终消息或请求工具；外壳执行工具，将结果加入下一轮输入，直到模型停止调用工具。[OpenAI：Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop/)

| 词 | 属于哪层 | 含义 |
| --- | --- | --- |
| Inference | L1 | 一次向模型请求输出。 |
| Agent loop | L1 + L2 + L3 + L4 | 决策、行动、观察、更新的反馈循环。 |
| Turn | L0 / 产品语义 | 用户一条消息到一次可见回复；可含多轮 loop。 |
| Run | L1 / L2 | 后端一次任务执行记录，有 run_id、状态和预算。 |
| Session / Thread | L0 + L2 | 多个 Turn 共享的会话容器。 |
| Observation | L2 | 工具、环境或子 Agent 返回的新事实/错误/元数据。 |
| Harness / Runtime | L1-L4 的跨层实现 | 包住模型的运行外壳：loop、工具、状态、权限、环境、交互。 |

---

## 3. Workflow、ReAct、Plan-and-Execute 放在 L1：都是控制流模式

| 模式 | L1 中谁决定下一步 | 适用场景 | 风险/治理点 |
| --- | --- | --- | --- |
| 单次 LLM | 开发者 | 抽取、分类、摘要 | 结构化输出校验即可 |
| **Workflow / Prompt Chain** | 开发者预定义 | 退款、发布、审批、固定报表 | 分支/状态机、审计、人审 |
| Routing | 分类器或 LLM | 客服、领域/模型分流 | 低置信度兜底到通用流程/人工 |
| 并行 + 汇总 | 编排器 | 独立检索、独立检查 | 汇总器解决冲突、控制成本 |
| **ReAct** | Model 在循环中选择 | 需边查边判断、路径未知 | 最大轮数、重复调用检测、工具白名单 |
| **Plan-and-Execute** | Model 先计划，执行中可重规划 | 长而可分解的任务 | 计划须是临时工件，不可绕过关卡 |
| Generator–Critic | 生成者和验证者 | 代码、报告、结构化产物 | 验收要独立且可运行 |
| Multi-Agent | Manager 或 handoff | 权限/工具/专业性不同，或子任务真能并行 | 明确输入输出契约、隔离 Context、控制交接成本 |

### Workflow 与 Agent 的区别

```text
Workflow（代码定路径）：查订单 → 硬规则校验 → 人确认 → 退款 → 对账

Agent（模型定下一步）：订单号缺失则追问；政策不明则检索；证据不足则转人工
```

生产常用组合：**外层 Workflow 固定高风险骨架，内层 ReAct 处理检索、诊断和解释。**Anthropic 将前者定义为预定义代码路径，后者定义为 LLM 动态管理过程与工具使用。[Anthropic：Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)

---

## 4. Context Engineering 放在 L2：不是只会写 Prompt

**Context** 是本次 inference 实际送入模型的所有 token：

```text
Instructions + 用户目标/附件 + 工具定义 + 历史消息/工具结果
+ 当前 State + RAG 证据 + 项目规则 + 相关代码/文档摘要
```

| 概念 | 属于哪层 | 关注点 |
| --- | --- | --- |
| Prompt Engineering | L1 | 指令、示例、输出格式怎么写。 |
| Context Engineering | L2 | 每轮从不断增长的信息中挑出最有用的 token。 |
| Just-in-time retrieval | L2 + L3 | 先存文件路径/文档 ID，真正需要时才读内容。 |
| Compaction | L2 | 近窗口上限时压缩已完成过程，续接新 Context。 |

长任务里，更多上下文不必然更好：日志、旧工具输出和无关文档会稀释关键信息。Anthropic 的定义是，在每次推理中持续管理系统规则、工具、MCP、外部数据和历史，使有限 Context 保持高信号。[Anthropic：Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

---

## 5. RAG 放在 L2：先讲架构，再讲如何排错

RAG 是“检索有来源的知识，把证据放入 Context 后再生成”。它解决**模型该知道什么**，不是自主行动系统。

```text
离线：文档 → 解析/OCR → 清洗/切块 → metadata（source/version/ACL）
      → 向量索引 + 倒排/BM25 索引 → 增量更新/删除

在线：用户问题 + 身份 → ACL filter → query rewrite（可选）
      → hybrid retrieve → rerank → Context budget → LLM 回答 + 引用
```

| RAG 部件 | 所属层 | 作用 |
| --- | --- | --- |
| 文档解析、chunk、embedding、索引 | L2 离线知识管道 | 把知识变成可检索单元。 |
| metadata / source / version / ACL | L2 + L4 | 知道证据来自哪里、何时有效、谁能看到。 |
| 关键词/BM25 + 向量召回 | L2 | 前者擅长编号/错误码，后者擅长语义相似。 |
| reranker | L2 | 对候选块精排，决定谁进前面的 Context。 |
| Context budget / packing | L2 | 选择多少块、怎样去重和排序，避免证据被截断。 |
| 引用和忠实度校验 | L1 + L5 | 让答案断言可追溯到证据；评估是否编造。 |
| ACL filter | L4 | 必须在检索前，不是生成后删敏感句。 |

### 高频难题：正确块在候选集里，但排不到前面，如何排查？

先禁止“盲调 embedding”。将一次请求拆成可观测的漏斗：

```text
Ground truth chunk
  → 召回候选集 top-K
  → 重排序候选集 top-N
  → 真正被装进 Context 的片段
  → 模型最终引用/回答
```

对每条标注问题记录 `gold_chunk_id` 在每一站的 **rank、score、是否被过滤、是否被截断**。这一步能把“RAG 答错”定位到一个具体环节。

| 现象 | 优先检查 | 常见原因 | 对应修复 |
| --- | --- | --- | --- |
| gold 不在 top-K | 召回层 | chunk 切得不完整；query 与文档词不一致；embedding/索引版本错；ACL/metadata 过滤误伤 | 调整切块和标题继承；query rewrite/扩展；混合召回；检查索引和过滤日志 |
| gold 在 top-K，却排得很后 | **排序层** | 只用向量导致精确词丢失；近义噪声块太多；候选 K 太小；没有/用错 reranker；文档标题、版本、权限等 metadata 未参与排序 | BM25 + 向量融合；扩大召回 K 后精排；引入 reranker；加入 freshness/source/标题等特征 |
| rerank 后靠前，却没有进入 Prompt | **Context packing 层** | token 预算截断；去重误删；多个长 chunk 挤掉正确块；拼接顺序丢失 | 记录最终 Context；给高分块保留预算；压缩其他块；控制 chunk 大小与每文档配额 |
| gold 已在 Context，回答仍错 | **生成/grounding 层** | 指令没要求依据证据；证据相互矛盾；答案被上下文中无关内容干扰 | 强制引用和“证据不足则拒答”；按主题分组；用 verifier 检查断言-引用对应关系 |
| 某些用户总是检索不到 | L4 过滤层 | tenant/ACL、文档版本、语言、时间范围过滤异常 | trace 记录 filter 前后数量及拒绝原因；做权限回归测试 |

这里有一个面试加分点：**“候选集有正确块”只说明 Recall@K 没问题，不能说明 RAG 没问题。**至少要分别看：

- `Recall@K`：正确块有没有被召回；
- `MRR / nDCG`：它在候选队列的位置是否足够靠前；
- `Context inclusion rate`：高分块是否真正进入模型 Context；
- `Citation correctness / Faithfulness`：模型答案是否由已给证据支持；
- `ACL violation rate`：是否有不该出现的 chunk。

### 一套可执行的 RAG 排障顺序

1. 建一个带 `question + gold_doc/chunk + expected_answer` 的小型评测集，先覆盖高频 bad case。
2. 对每条请求打 trace：原 query、rewrite、过滤条件、各索引 top-K、rerank 分数、最终 Context、引用。
3. 先判定问题发生在**召回、排序、装包还是生成**，一次只改一个环节。
4. 改动后按问题类型比较指标；例如做 rerank，不应只看最终答案，也要看 MRR/nDCG 是否真的提升。
5. 线上灰度观察延迟、token、拒答率与越权率；把新 bad case 回灌离线集。

---

## 6. 三类企业 Agent：目标、主架构和关键点

### 6.1 Enterprise RAG / Knowledge Agent

**目标**：员工询问政策、产品或技术资料时，给出有版本、有引用、权限正确的回答；证据不足时澄清、拒答或转人工。

```text
用户/身份 → ACL 过滤 → Hybrid Retrieval → Rerank → Evidence Gate
  ├─ 证据充分：LLM（带引用约束）→ 回答
  └─ 证据不足：追问 / 拒答 / 人工 / 实时业务 Tool
```

**推荐架构**：L2 的 RAG Workflow 为主，局部 ReAct 只负责模糊问题、多跳检索或决定是否调用实时 Tool。权限、召回、引用格式和阈值必须是确定性关卡。

**亮点**：chunk 级 ACL 前置；来源/版本/删除同步；“证据充分性”独立于模型置信度；将检索、grounding、答案三段分别评测。

### 6.2 On-call / Incident Response Agent

**目标**：缩短定位时间，整合告警、日志、指标、trace、变更和 Runbook；先给证据化建议，再逐步开放低风险自动化。

```text
Alert → 去重/归一化/统一时间窗
      → Workflow 拉取 Metrics + Logs + Traces + 最近 Deploy + CMDB
      → RAG 召回 Runbook/历史事故
      → 只读 ReAct Diagnose Agent（假设与证据链）
      → Policy Gate → 建议 | 低风险预批准动作 | 人审后高风险动作
      → 执行后重新验证 SLI/SLO
```

**推荐架构**：L1 固定采集 Workflow + 只读 ReAct 诊断 + L4 审批式执行。不要把“API 返回 200”当事故恢复，必须用 SLI/SLO 二次验证。

**亮点**：输出严格分成“观测事实 → 待验证假设 → 建议动作 → 风险/批准条件”；所有查询带 incident time window；工具按只读/可逆/高风险分级；评测分诊准确率、根因候选命中、人工采纳率、危险操作拦截率和 MTTR。

### 6.3 Data / Analytics Agent

**目标**：把“为什么华东新用户转化下降”转化为**口径正确、权限合规、可复现**的数据结论，而不是只生成一段 SQL。

```text
问题 → 指标语义层（metric/dimension/lineage/owner/ACL）
     → Router → Plan → Safe SQL / 语义查询工具
     → 数仓/BI → 结果校验 → 分析解释 + 图表 + SQL/口径引用
     → 必要时 drill-down loop
```

**推荐架构**：语义层约束的 Plan-and-Execute。数据集选择、指标口径、行列权限和查询成本属于 L2/L4 的确定性门；SQL 生成与业务解释分开。

**亮点**：优先暴露语义工具而非裸 SQL；必须 SQL 时做 allowlist、只读账号、AST 校验、dry-run/成本上限、超时与审计；结果 validator 检查空值、重复 join、时区、总量和口径；输出保留 SQL、指标版本、过滤条件和数据快照。

| Agent | L1 主模式 | L2 真相来源 | L4 最重要防线 |
| --- | --- | --- | --- |
| Enterprise RAG | 检索 Workflow + 局部 ReAct | 有 ACL/版本的文档证据 | 检索前 ACL、证据充分性、引用忠实度 |
| On-call | 采集 Workflow + 只读 ReAct + 审批 | 实时 telemetry + Runbook | 最小权限、风险分级、写后验证 |
| Data Agent | Plan-and-Execute + 查询校验 | 语义层、数仓、BI | 行列权限、成本上限、口径/SQL 校验 |

共同原则：**不变的业务规则、权限与风险边界放在代码/数据层；理解、规划和解释交给模型。**

---

## 7. 术语速查：每个词先找层

| 名词 | 层 | 一句话 |
| --- | --- | --- |
| Agent | 跨层系统 | 使用模型动态选择步骤/工具，在边界内完成多步目标。 |
| Agentic system | 跨层系统 | 广义带 LLM 的行动系统，包含 Workflow 和 Agent。 |
| ReAct | L1 | Reason + Act：根据 Observation 继续决策。 |
| Context Engineering | L2 | 每轮挑选和维护高信号 Context 的工程。 |
| RAG | L2 | 检索外部证据后再生成。 |
| Memory | L2 | 跨会话的已确认信息。 |
| Function Calling | L1→L3 接口 | 模型输出符合 Schema 的工具调用请求。 |
| MCP | L3 接入协议 | 连接 Agent 与外部工具/资源的协议。 |
| Skill | L1/L3 | 可复用的工作说明与工具能力包。 |
| Guardrail | L4 | 程序化的行为、权限或格式防护。 |
| Eval | L5 | 固定数据集上的系统化质量比较。 |
| Human-in-the-loop | L4 | 人对高影响动作审批、拒绝或接管。 |

## 8. 复习顺序

1. 先画第 1 节六层图，任何新概念先问“它属于哪层”。
2. 用第 2 节讲一次 `Context → Model → Tool → Observation` 的循环。
3. 比较第 3 节：Workflow 是代码定路径，ReAct 是模型定下一步。
4. 用第 5 节讲 RAG 分层和“正确块靠后”的排障漏斗。
5. 从第 6 节任选企业场景，按目标、架构、风险、评测完整表达。

## 参考资料（官方）

- [OpenAI：A practical guide to building agents](https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/)
- [OpenAI：Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop/)
- [Anthropic：Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- [Anthropic：Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Kimi：智能体 AI 架构在实践中的运作方式](https://www.kimi.com/resources/agentic-ai-architectures)
