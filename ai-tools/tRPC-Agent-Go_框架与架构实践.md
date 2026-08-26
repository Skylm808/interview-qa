# tRPC-Agent-Go：Go Agent 框架与架构实践

> 官方仓库：[trpc-group/trpc-agent-go](https://github.com/trpc-group/trpc-agent-go/)
> 适合：Go 后端、AI 应用、Agent 平台、RAG、AI Coding 方向。
> 这篇笔记不把它当成“又一个聊天机器人 SDK”，而是看它怎样把 Agent 的运行、状态、工具、图编排、评测和观测接进 Go 服务。

---

## 1. tRPC-Agent-Go 是什么，不是什么？

tRPC-Agent-Go 是一个面向生产 Agent 系统的 Go 框架。官方仓库提供了 LLM Agent、Graph 工作流、工具调用、Session 与 Memory、知识检索、MCP、A2A、AG-UI、评测和 OpenTelemetry 观测等能力。

它不是模型，也不会替你决定业务规则。模型调用、工具权限、检索质量、审批逻辑和数据存储仍然由应用负责。它解决的是：这些能力怎样以 Go 原生的并发、`context` 取消、事件流和服务化接口组织起来。

可以先记一句：

> **tRPC-Agent-Go 把“模型思考、工具执行、状态保存、事件输出”组织成可组合的 Go 运行时。**

---

## 2. 核心组件：每个包负责什么？

| 组件 | 主要职责 | 可以类比为 |
| --- | --- | --- |
| `agent` | Agent 的核心抽象，接收输入并产出事件/响应 | 一个可执行的任务单元 |
| `runner` | 管理一次运行，连接 Agent、Session、Memory、事件和取消 | 请求级编排器 |
| `model` | 屏蔽 OpenAI、DeepSeek 等模型提供方差异 | 大模型适配层 |
| `planner` | 规划下一步策略和工具选择 | 任务决策器 |
| `tool` | Function、MCP、搜索、代码执行等外部能力 | Agent 的受控手脚 |
| `session` | 保存当前会话的消息和事件状态 | 当前对话的工作台 |
| `memory` | 保存跨会话的长期记忆、偏好和可检索信息 | 用户长期档案 |
| `knowledge` | 文档加载、向量化、检索等 RAG 能力 | 知识库模块 |
| `graph` | 条件分支、并行和状态图编排 | 可控 Workflow / 状态机 |
| `skill` | 加载和执行以 `SKILL.md` 定义的可复用流程 | 可复用 SOP |
| `artifact` | 保存工具产出的版本化文件，如报告、图片 | 任务产物仓库 |
| `evaluation` | 用评测集和指标评估 Agent | 自动化验收 |
| `telemetry` | OpenTelemetry Trace 和 Metrics | 运行轨迹与指标 |
| `server` | 对外提供 Gateway、AG-UI、A2A 等服务接口 | 接入层 |

最容易混淆的是 `session`、`memory`、`knowledge`：

- `session` 是“这次对话刚说过什么、正在做第几步”；
- `memory` 是“这个用户长期偏好什么、过去确认过什么”；
- `knowledge` 是“系统文档里有什么证据可以用来回答当前问题”。

三者都可能被召回进 Prompt，但生命周期、权限和写入规则不应混在一起。

---

## 3. 常用的生产架构

```text
Web / App / CLI
  -> HTTP、SSE 或 AG-UI Server
  -> Runner（user_id、session_id、request_id、取消与事件流）
  -> Agent
       -> Model / Planner：理解目标，决定回答、检索或调工具
       -> Tool：业务 API、MCP、搜索、代码执行
       -> Knowledge：RAG 检索
       -> Memory：长期偏好和历史事实
       -> Graph / Team：固定流程、分支、并行或多 Agent 协作
  -> Event Stream：文本增量、工具状态、错误、完成事件
  -> Session / Artifact / Trace / Eval 结果
```

一次请求通常由 `Runner` 开始。它拿到用户、会话和消息后，驱动 Agent 执行，并把流式 `event.Event` 交给服务端。前端既可以只展示文本 token，也可以展示“正在检索”“正在调用工具”“等待审批”等中间事件。

这个边界很实用：HTTP 层不应该自己维护 Agent 的循环状态；具体工具也不应该直接操纵 SSE 连接。`Runner` 负责运行，Agent 负责决策，Tool 负责执行，Server 只负责协议转换。

---

## 4. 在框架里理解 Agent Loop

Agent Loop 可以简化为：

```text
Runner 载入 Session / Memory
  -> Agent 调模型，决定 Final Answer 或 Tool Call
  -> Tool 执行并产生 Observation
  -> Observation 写回 Session / State
  -> Agent 根据新状态继续，直到满足结束条件
```

框架里的 `CycleAgent` 就适合“Planner + Executor 反复迭代，直到收到停止信号”的模式。对于有明确步骤、审批和分支的业务，优先使用 Graph：把鉴权、参数校验、支付、写库等确定性节点固定下来，只把需要开放判断的局部交给 LLM。

生产环境要把循环限制写在代码或图里：

- 最大轮数、最大工具调用数、总超时和 token / 成本预算；
- 工具白名单、参数 Schema、权限与幂等键；
- 连续失败或状态无进展时的 fallback；
- 用户取消时调用 Go `context` 取消，并继续消费事件直到 channel 关闭；
- 长任务在关键节点写入 checkpoint，避免进程重启后从头执行。

不要把模型完整思维链写入日志。记录计划摘要、工具名、脱敏参数摘要、耗时、结果摘要和状态转移，已经足够排障和评测。

---

## 5. 内置 Agent 与常见编排方式

| 类型 | 适合什么任务 | 取舍 |
| --- | --- | --- |
| `LLMAgent` | 单 Agent 对话、简单工具调用、快速原型 | 上手快，但要自己限制工具和循环 |
| `ChainAgent` | 翻译、摘要、格式化等线性流水线 | 最直观；后一步依赖前一步，延迟会累加 |
| `ParallelAgent` | 多路检索、代码与测试并行生成 | 降低端到端延迟；要处理结果合并和资源上限 |
| `CycleAgent` | 计划—执行—观察的多轮任务 | 灵活；必须有停止条件和失败退出 |
| GraphAgent / `graph` | 审批、状态机、条件分支、并行扇出 | 可控、可测试；前期需要先把状态和节点设计清楚 |

官方示例中，`ChainAgent` 顺序运行多个子 Agent，`ParallelAgent` 并发运行后再合并结果，`CycleAgent` 迭代直到结束。Graph 工作流支持条件路由和多分支并行。

选型原则很简单：

- 路径固定，用 Chain 或 Graph；
- 彼此独立的子任务，用 Parallel；
- 需要依据工具结果临场调整，用有限轮次的 Cycle；
- 涉及写操作、资金、权限、发布，采用 Graph + 人工确认，不让自由循环直接执行。

---

## 6. 最小 Go 示例：带一个 Function Tool 的 Agent

下面是根据官方基本用法简化的示例，省略了模型提供方的 API Key 与部署配置。重点不是计算器，而是四个连接关系：Model → Tool → `LLMAgent` → `Runner`。

```go
package main

import (
    "context"
    "fmt"
    "log"

    "trpc.group/trpc-go/trpc-agent-go/agent/llmagent"
    "trpc.group/trpc-go/trpc-agent-go/model"
    "trpc.group/trpc-go/trpc-agent-go/model/openai"
    "trpc.group/trpc-go/trpc-agent-go/runner"
    "trpc.group/trpc-go/trpc-agent-go/tool"
    "trpc.group/trpc-go/trpc-agent-go/tool/function"
)

type addRequest struct {
    A float64 `json:"a" jsonschema:"description=first operand,required"`
    B float64 `json:"b" jsonschema:"description=second operand,required"`
}

type addResponse struct {
    Sum float64 `json:"sum"`
}

func add(_ context.Context, in addRequest) (addResponse, error) {
    return addResponse{Sum: in.A + in.B}, nil
}

func main() {
    m := openai.New("gpt-4o-mini")
    addTool := function.NewFunctionTool(
        add,
        function.WithName("add"),
        function.WithDescription("Add two numbers"),
    )

    a := llmagent.New("assistant",
        llmagent.WithModel(m),
        llmagent.WithTools([]tool.Tool{addTool}),
    )
    r := runner.NewRunner("demo-app", a)

    events, err := r.Run(
        context.Background(),
        "user-001",
        "session-001",
        model.NewUserMessage("请计算 12 加 30"),
    )
    if err != nil {
        log.Fatal(err)
    }
    for event := range events {
        if event.Object == "chat.completion.chunk" {
            fmt.Print(event.Response.Choices[0].Delta.Content)
        }
    }
}
```

模型决定是否调用 `add`；真正执行计算的是 Go 函数。线上工具不应直接暴露数据库写入、Shell 或退款等高风险能力，而应在 Function Tool 前增加身份校验、参数校验、审批与审计。

---

## 7. 一个更接近生产的例子：故障诊断助手

假设用户问：“订单接口今天下午为什么变慢了？” 一个稳妥的架构不是把所有权限交给模型，而是分层处理：

```text
用户问题
  -> Graph：鉴权、租户/服务范围校验、时间范围补全
  -> Parallel：指标查询 + Trace 查询 + 发布记录查询
  -> CycleAgent：根据三类证据生成假设，决定是否追加一次只读查询
  -> Graph：规则校验“结论必须有证据”“不能执行写操作”
  -> Artifact：生成诊断报告
  -> AG-UI / SSE：持续展示检索进度和最终建议
```

这里可以把 Prometheus、日志平台、Trace 平台封成只读 Tool；把“回滚发布”“扩容实例”封成需要审批的高风险 Tool。`ParallelAgent` 先并行取证，减少等待；`CycleAgent` 只允许有限轮补查；Graph 保证任何自动操作不会越过审批节点。

这个例子也说明了 Agent 和传统后端的分工：传统服务提供稳定的查指标、查 Trace、执行变更 API；Agent 负责理解自然语言、组织取证顺序、汇总证据和解释结论。

---

## 8. 不只会调模型：框架里的其他能力怎么用

### 8.1 Session、Memory 与 Knowledge

- Session 保存本轮对话和运行事件，适合恢复当前任务。
- Memory 服务可以使用内存或 Redis 等后端，适合跨会话保存偏好和长期事实。写入前要确认用户隔离与过期策略。
- Knowledge 是 RAG 层，负责加载数据源、生成向量并检索。它不是 Memory 的替代品：知识库更偏公共或业务文档，Memory 更偏用户和任务历史。

### 8.2 Tool、MCP、Skill 与 A2A

- Function Tool 最适合把明确的 Go 函数或业务 API 暴露给 Agent。
- MCP Tool 用于接入按 MCP 协议发布的工具、资源和 Prompt。
- Skill 将一类任务的步骤、文档和可选脚本定义为 `SKILL.md`；Agent 可按需加载和运行。启用本地执行时要特别限制工作目录、命令白名单和网络权限。
- A2A 用于 Agent 与其他 Agent 运行时互操作；它不是 Multi-Agent 的唯一实现方式，进程内的 Chain、Parallel、Graph 已能覆盖很多场景。

### 8.3 AG-UI、Event 与 Telemetry

AG-UI 解决 Agent 与前端之间的事件协议问题，仓库示例提供 SSE 服务和客户端示例。无论用不用 AG-UI，前端都应接收结构化事件，而不是只等最后一段文本：例如 `text_delta`、`tool_call_started`、`tool_call_finished`、`approval_required`、`error`、`done`。

Telemetry 基于 OpenTelemetry 覆盖 Runner、Model、Tool 等层。生产排障时至少关联 `request_id`、`user_id`（脱敏）、`session_id`、模型名、token、工具耗时、重试次数和最终状态；不要把原始 Prompt、密钥和个人数据直接送到 Trace 平台。

---

## 9. 上线前检查清单

- 每个 Tool 是否有 Schema、超时、鉴权、幂等和审计？
- `Runner.Run` 的 `context` 是否能被用户取消，并正确收尾事件流？
- 每个循环是否有轮数、时间、成本和重复失败的停止条件？
- Session、Memory、RAG 文档是否按租户和用户隔离？
- 高风险工具是否经过人工确认或独立审批服务？
- 是否有离线评测集、线上 Trace、工具成功率和 bad case 回放？
- 是否区分了“固定流程应该由 Graph 管”和“开放判断才交给 LLM”？

---

## 10. 面试里怎么回答

**问：tRPC-Agent-Go 的核心架构是什么？**

> 我把它理解为一个 Go 原生的 Agent 运行时。Runner 负责一次运行以及 Session、Memory、事件和取消；Agent 调用模型做决策；Planner、Tool、Knowledge 分别处理规划、外部执行和 RAG；Graph 和 Chain/Parallel/Cycle 用于不同复杂度的编排；最后通过 Server、AG-UI/A2A/MCP 对接外部系统，通过 OpenTelemetry 和 Evaluation 做可观测和评测。真正上线时，我会把确定性高风险流程固定在 Graph 和审批节点里，不让 LLM 自由循环直接执行。

**问：什么时候用 Graph，什么时候用 CycleAgent？**

> 审批、权限、状态迁移、写库等路径确定的逻辑用 Graph，因为节点、分支和状态都能测试；需要根据工具返回动态调整下一步的分析类任务用 CycleAgent，但要限制轮数、工具数、超时和预算。生产里更常见的是 Graph 包住主流程，CycleAgent 只负责其中一小段开放探索。

**问：它和只用 OpenAI SDK 有什么差别？**

> 只用模型 SDK 可以完成单次对话和 Function Calling；tRPC-Agent-Go 在其上提供了运行管理、Session/Memory、图编排、多 Agent、事件流、协议接入、评测和观测等工程能力。简单 Demo 直接调 SDK 足够；长会话、多工具、多步骤和服务化场景更需要这些运行时能力。

---

## 11. 官方资料

- [tRPC-Agent-Go 官方仓库与 README](https://github.com/trpc-group/trpc-agent-go/)
- [官方文档站](https://trpc-group.github.io/trpc-agent-go/)
- [Graph 工作流示例](https://github.com/trpc-group/trpc-agent-go/tree/main/examples/graph)
- [Runner 示例](https://github.com/trpc-group/trpc-agent-go/tree/main/examples/runner)
