# AI Agent / AI Coding / RAG 面经合集

> 阅读顺序：**先看架构，再看一次任务链路，再做高频题。**

## 一、完整 Agent 架构

```text
用户 / Client
  -> Gateway：鉴权、限流、租户隔离、SSE / WebSocket
  -> Agent Runtime / Orchestrator
       -> State：任务、计划、步骤、预算、checkpoint
       -> Context：Session、Memory、RAG 证据
       -> Policy / Workflow：路由、审批、终止条件
       -> Planner / Model：回答、检索、追问、调工具
       -> Tool Executor：Function Call、MCP、Skill、代码 / 浏览器 / 业务 API
       -> Guardrail：Schema、权限、幂等、风险、审计
       -> Model Router：模型选择、fallback、流式输出
  -> DB / Cache / MQ / Vector DB / 外部服务
  -> Trace / Eval / Metrics / Bad Case 回放
```

| 层 | 做什么 |
| --- | --- |
| Gateway | 接入、鉴权、限流、流式返回，绑定 `user_id`、`tenant_id`、`request_id`。 |
| Runtime | 管一次任务的超时、取消、状态持久化、恢复和成本预算。 |
| Planner / Model | 决定下一步是直接答、RAG、追问还是调用工具。 |
| Context | 只提供当前需要的对话、摘要、长期记忆和检索证据。 |
| Tool Executor | 受控执行外部能力，负责参数、权限、超时、重试与审计。 |
| Guardrail | 控制越权、危险操作、循环和 Prompt Injection。 |
| Eval / Trace | 观察质量、延迟、成本和失败原因。 |

**一句话：**模型负责不确定的理解与决策；Runtime、Workflow 和工具层负责把决策变成可控执行。

---

## 二、基础概念：Agent、RAG、Workflow

### 1. Agent 和普通后端有什么不同？

普通后端的调用路径主要由开发者写死；Agent 会围绕目标在回答、检索、工具调用和观察结果之间调整下一步。Agent 不会替代微服务，而是在微服务之上增加任务理解和智能编排。

Agent Runtime 还要治理任务状态、总时长、轮数、token / 成本、工具副作用和不可信执行；这也是 Coding Agent 常需要 Sandbox 的原因。

### 2. RAG 是什么？

RAG 是“先检索证据，再生成回答”：

```text
文档 -> 解析 / 切块 -> metadata + ACL -> 向量 / 倒排索引
问题 -> 权限过滤 -> 混合召回 -> rerank -> 证据进 Prompt -> 回答
```

RAG 解决私有知识、过期知识和引用证据；它是 Agent 的知识组件，不等于 Agent。当前订单、指标、日志等实时事实必须调用工具，不能只查 RAG。

### 3. Workflow 与 Agent 怎么选？

| 场景 | 选择 |
| --- | --- |
| 支付、退款、审批、发布 | Workflow / 状态机：步骤与权限固定。 |
| 开放问答、资料检索、诊断、工具选择 | Agent / ReAct：根据中间结果调整。 |
| 生产默认 | **Workflow 管住确定主流程，局部节点交给 Agent 决策。** |

---

## 三、一次请求怎样运行？

以“查订单，符合条件则申请退款”为例：

```text
1. Gateway 鉴权，创建 request / session 上下文
2. Runtime 建立 State：deadline、轮数、预算、checkpoint
3. Planner：缺订单号则追问；否则调用查询工具
4. Executor：校验参数和当前用户权限，带超时执行
5. 结果压缩为 Observation，写回 State
6. Workflow 校验退款资格；需要时向用户申请确认
7. 确认后带幂等键调用退款工具，保存产物与审计
8. 输出流式进度 / 最终结果，记录 Trace 和指标
```

State 不能只保存聊天记录；至少要有任务 ID、已确认参数、计划、工具调用 ID、结果摘要、审批状态、重试次数和预算，才能取消、恢复和回放。

---

## 四、ReAct：最常见的执行循环

ReAct 是 **Reasoning + Acting**：模型根据 State 选择 Action，工具返回 Observation，再决定下一步。

```text
State -> Plan（回答 / 检索 / 追问 / 工具）
      -> 校验并执行 Action
      -> Observation 写回 State
      -> 已完成？Final : 下一轮
```

必须设最大轮数、最大工具数、总时长、token / 成本预算、用户取消、连续失败和无进展检测；高风险写操作转审批。工程里记录计划摘要、工具参数摘要和结果即可，不必保存模型完整思维链。

---

## 五、常用架构

### 1. 单 Agent + ReAct

适合查订单、查知识库、生成总结等短任务。实现快，但工具和轮数必须受限。

### 2. Workflow / Graph + 局部 Agent

```text
固定：鉴权 -> 资格校验 -> 审批 -> 执行 -> 对账
开放：意图识别、RAG、候选工具选择、结果解释
```

适合退款、发版、审批、On-call；也是生产系统最稳的默认方案。

### 3. Plan-and-Execute 与 Multi-Agent

Plan-and-Execute 先生成可检查计划，再逐步执行和更新。Multi-Agent 只在角色的权限、工具或评审职责确实不同，或任务能独立并行时再拆；否则单 Agent + Workflow 更易维护和控成本。

### 4. Coding Agent / On-call Agent

```text
Coding：任务 -> 代码检索 / 规划 -> Sandbox -> 测试 -> Diff / 证据 -> 人工确认
On-call：告警 -> 实时指标 / 日志 / Trace + RAG Runbook -> 建议 -> 审批执行 -> 验证
```

Sandbox 限制文件、网络、CPU、内存和执行时间；RAG 只提供 Runbook、历史事故和文档，当前线上状态必须用实时工具查。

---

## 六、核心组件

### 1. Function Calling、MCP、Skill

| 概念 | 作用 |
| --- | --- |
| Function Calling | 模型输出“调用哪个函数、传什么结构化参数”。 |
| Tool | 应用真正执行的能力，如 DB、RPC、浏览器、搜索。 |
| MCP | 工具 / 资源提供方与 Agent 客户端的标准接入协议。 |
| Skill | 可复用工作流说明：触发条件、事实源、步骤、边界、验证与交付。 |

模型可通过 Function Calling 选择 MCP Tool；Skill 约束什么时候、按什么步骤使用工具。三者可组合，不互相替代。

### 2. Memory 与 Context

- 短期：当前会话、Observation、任务状态。
- 摘要：长会话中的已确认事实、未完成步骤与关键结论。
- 长期：跨会话偏好和经验，必须带 user / tenant、来源、时间、可信度和过期策略。

不要把所有历史或全文文档塞 Prompt；先压缩、过滤和检索。

### 3. RAG 工程要点

- 文档保留来源、版本、标题路径、chunk 和 ACL，支持删除 / 重建。
- 向量检索解决语义相似，BM25 / 倒排解决错误码和精确词，再做 rerank。
- 权限过滤要发生在召回阶段，不是生成后再删敏感句。
- 评测看召回、上下文相关性、引用正确性、答案忠实度、拒答正确率和延迟。

### 4. 工具可靠性与安全

调用前：Schema、权限、参数范围、风险分级。调用中：deadline、重试预算、幂等键。调用后：结果校验、审计和状态确认。生产读取限制时间窗和返回行数；高风险写操作默认审批。

---

## 七、工程化：可靠、可观测、可评估

### 1. 可靠性与成本

- 每个任务都有 deadline、并发 / 轮数 / 工具数上限和 token / 金额预算。
- 重试只用于可重试且幂等的操作；失败应降级、转人工或明确失败。
- 长任务使用 checkpoint；外部写操作使用幂等键。

### 2. 延迟优化

- 无依赖的检索 / 工具并行，依赖步骤串行。
- RAG 和工具结果做摘要，控制 Prompt 长度。
- 使用流式输出、模型路由；推理层再考虑 batching、KV Cache 和模型缓存。

### 3. AgentOps / Eval

| 类别 | 指标 |
| --- | --- |
| 质量 | 任务完成率、忠实度、引用正确率、人工接管率 |
| 工具 | 成功率、参数错误、重复调用、重试次数 |
| 性能 | 首 token、端到端耗时、token、模型 / 工具成本 |
| 安全 | 越权拦截、审批拒绝、注入命中 |

Trace 应能回放输入、证据、工具调用、结果、模型 / Prompt / Skill 版本。bad case 分类后回灌离线评测集，再灰度验证改动。

---

## 八、高频面试题

### Q1：什么是 ReAct？为什么不用纯 Workflow？

ReAct 是“决策 → 调工具 → Observation → 再决策”，适合路径不确定的任务；纯 Workflow 适合固定高风险流程。生产中常用 Workflow 固定骨架、ReAct 做局部开放决策。

### Q2：Agent 与 RAG 有什么区别？

RAG 找证据并辅助回答；Agent 决定是否检索、是否调工具、怎样完成多步任务。RAG 是 Agent 的组件。

### Q3：Agent Runtime 至少需要什么？

State、Context / Memory、模型与编排、受控工具、权限与审批、预算与终止条件、Trace / Eval。它比普通后端多管理任务不确定性、资源消耗和工具副作用。

### Q4：什么时候需要 Multi-Agent？

角色权限、工具集合或评审职责确实不同，或任务能独立并行时再拆。否则单 Agent + Workflow 更稳。

### Q5：Memory 如何避免串话与过期？

短期状态绑定 session，长期记忆绑定 user / tenant；保存来源、时间和过期策略，检索前权限过滤。模型猜测不能直接沉淀为长期事实。

### Q6：怎样防止工具调用失控？

工具白名单、Schema、最小权限、最大轮数 / 成本 / 时长、重复调用检测、超时和幂等键。高风险操作展示影响并审批。

### Q7：知识库没有答案时怎么办？

设置召回阈值；证据不足则拒答、转人工或调用实时工具，不要编造。

### Q8：如何做 Agent 评测？

离线评估任务完成、工具选择、检索证据和格式；线上看质量、延迟、成本和安全；bad case 回放并回灌评测集。

### Q9：MCP、Function Calling、Skill 怎么区分？

Function Calling 是模型输出调用的能力；MCP 是工具接入协议；Skill 是任务工作流约束。三者可组合。

### Q10：Coding Agent 为什么需要 Sandbox？

代码执行可能访问文件、网络或运行不可信脚本。Sandbox 限资源与权限，避免影响宿主机；仍要配合鉴权、密钥隔离和人工确认。

---

## 九、AI Coding 的 TDD 补充

TDD 适合确定逻辑：工具 Schema、权限策略、状态机、输出解析器、RAG 过滤和 Skill 工作流。模型生成结果更适合用 Eval：样本、评分规则、坏例回归和人工验收。**TDD 保证确定逻辑，Eval 衡量不确定输出。**

## 十、复习顺序

1. 先背第一章的架构图和职责。
2. 用第三、四章讲清一次任务链路和 ReAct。
3. 按岗位选择第五章：RAG、Coding、On-call、Multi-Agent。
4. 最后复习工程治理和高频题。

## 参考资料

- [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/)
- [Model Context Protocol](https://modelcontextprotocol.io/)
- [LangGraph 文档](https://langchain-ai.github.io/langgraph/)
