# AI 工具链 / OpenClaw / Ironclaw 整理

> 目标：把 `function call`、`MCP`、`skills`、`Codex / Claude Code 常用命令`、`OpenClaw / Ironclaw 框架`、`Rust 安全性`、`WASM 沙箱` 用一份文档讲清楚。
> 风格：尽量详细，但不绕，适合自己复习，也适合面试时讲明白。

---

## 1. function call、MCP、skills，到底有什么区别？

这三个词经常一起出现，但它们其实不在同一个抽象层次上。

### 1.1 一句话先分层

- `function call`：**模型调用一个具体函数/工具**
- `MCP`：**把外部工具、数据源、工作流接进模型的标准协议**
- `skills`：**把经验、步骤、约束打包成可复用的“能力说明书”**

也就是说：

- `function call` 更像“执行动作”
- `MCP` 更像“接外设的标准接口”
- `skills` 更像“做事方法论 + 操作手册”

### 1.2 最容易记住的类比

- `function call` 像你按了一个具体按钮：`get_weather(city="Beijing")`
- `MCP` 像一排标准化插口：只要设备按协议接进来，AI 就能统一使用
- `skills` 像 SOP / 作战手册：告诉 agent 遇到什么问题该怎么做、按什么步骤做

### 1.3 三者对比表

| 维度    | function call      | MCP                                            | skills              |
| ----- | ------------------ | ---------------------------------------------- | ------------------- |
| 本质    | 一次具体工具调用           | 连接 AI 与外部系统的标准协议                               | 一组可复用的提示、规则、脚本、模板   |
| 抽象层级  | 最低                 | 中间层                                            | 最高                  |
| 解决的问题 | “现在调用哪个函数？”        | “外部工具怎么统一接进来？”                                 | “这类任务应该怎么做更稳？”      |
| 典型内容  | 函数名、参数 Schema、返回结果 | tools / resources / prompts / server-client 交互 | `SKILL.md`、模板、脚本、示例 |
| 适合场景  | 查天气、查订单、执行 SQL、发请求 | 接 GitHub、数据库、Figma、Notion、浏览器等外部系统             | 代码评审、方案设计、调试流程、发布流程 |
| 关注点   | 参数是否正确             | 协议是否统一、工具是否可发现                                 | 流程是否稳定、经验是否可复用      |

### 1.4 function call 是什么？

`function call` 最核心的点是：**模型不直接“执行代码”，而是先按结构化参数提出“我要调用这个函数”**，然后由宿主程序真正去执行。

比如定义一个函数：

```json
{
  "type": "function",
  "name": "get_weather",
  "parameters": {
    "type": "object",
    "properties": {
      "location": { "type": "string" }
    },
    "required": ["location"]
  }
}
```

模型如果判断需要查天气，就会产出类似：

```json
{
  "name": "get_weather",
  "arguments": {
    "location": "Beijing"
  }
}
```

然后真正执行的是宿主程序，不是模型本身。

你可以把它理解成：

- 模型负责“决定调用什么、传什么参数”
- 你的程序负责“真正去执行、拿结果回来”

所以 function call 的优点是：

- 结构化，容易校验
- 权限边界清楚
- 比让模型直接拼命令更稳

但它的局限也很明显：

- 一次 function call 只描述一个具体动作
- 如果工具一多，函数定义会越来越多
- 不同平台之间不通用，常常是“这家一套、那家一套”

### 1.5 MCP 是什么？

MCP 全称是 `Model Context Protocol`。

它解决的是一个更大的问题：

> 不是“模型怎么调一个函数”，而是“AI 应用怎么统一接入外部世界”。

官方介绍里，MCP 是一个开源标准，让 AI 应用连接：

- 数据源：本地文件、数据库
- 工具：搜索、计算器、浏览器
- 工作流：特殊 prompt、业务操作流

所以 MCP 比 function call 高一层。

#### 你可以这样理解 MCP

如果说 function call 是：

- “这里有一个函数，你来调”

那 MCP 更像是：

- “这里有一个标准化工具服务器，你来发现它有哪些能力，再决定怎么用”

也就是说，MCP 往往不只暴露一个函数，而是可能同时暴露：

- tools
- resources
- prompts
- 甚至一整套外部系统能力

#### MCP 和 function call 的关系

- function call 是“单次工具调用”
- MCP 是“工具接入标准”

在很多 agent 体系里，MCP 最终仍然会落到某种“工具调用”上，但它解决的是：

- 发现工具
- 标准化接入
- 权限边界
- 跨客户端复用

所以你可以把它记成：

> **function call 是“点按钮”，MCP 是“定义按钮面板怎么接进来”。**

### 1.6 skills 是什么？

`skills` 不是协议，也不是底层调用机制，而是 **把一类任务的最佳实践打包成可复用能力**。

在 Claude Code 官方文档里，skill 本质上是：

- 一个 `SKILL.md`
- 可选的模板、脚本、示例、参考文档
- 前面还能加 frontmatter，控制什么时候触发、谁能触发、能用哪些工具

在你现在这个 Codex 环境里，其实也是类似思路：

- skill 是本地能力包
- 核心文件也是 `SKILL.md`
- 可能带脚本、模板、参考资料
- 它告诉 agent：遇到这类问题，应该按什么步骤做

所以 skills 更像：

- 经验封装
- 工作流封装
- 行为约束封装

#### skills 和 function call / MCP 的关系

可以把三者串起来看：

```text
skills
  -> 告诉 agent 该怎么做
  -> agent 决定调用哪些工具
  -> 这些工具可能是 function call
  -> 也可能来自 MCP server
```

所以：

- `skills` 决定“方法”
- `function call` 决定“动作”
- `MCP` 决定“外部能力怎么接进来”

### 1.7 面试里怎么一句话答这三个的区别？

你可以这样说：

> function call 是模型调用某个具体工具的结构化方式；MCP 是把外部工具、数据源和工作流统一接入 AI 的标准协议；skills 则是更高一层的能力封装，本质上是一套可复用的做事方法和约束。简单说，function call 是“执行动作”，MCP 是“接外部能力”，skills 是“教 agent 怎么更稳地完成一类任务”。

---

## 2. Codex 和 Claude Code 的一些常用命令

说明：

- 下面这些命令，`Codex` 和 `Claude Code` 部分主要基于我在本机 `2026-03-17` 运行 `--help` 看到的结果整理
- 所以这部分是“当前本机常用命令总结”，不是官方完整手册

### 2.1 Codex 常用命令

#### 启动与基本使用

```bash
codex
```

- 直接进入交互式会话

```bash
codex "帮我分析这个仓库"
```

- 直接带一句 prompt 启动

#### 非交互执行

```bash
codex exec "帮我生成本周工作总结"
```

- 非交互执行任务

```bash
codex exec --json "分析这个目录"
```

- 用 JSONL 方式输出结果，适合脚本接入

#### 代码相关

```bash
codex review
```

- 非交互代码审查

```bash
codex apply
```

- 把最近一次 agent 产出的 diff 应用到当前工作区

#### 会话恢复

```bash
codex resume
codex resume --last
codex fork --last
```

- `resume`：恢复会话
- `fork`：从旧会话分叉一个新会话继续做

#### MCP 相关

```bash
codex mcp list
codex mcp add <name> <command-or-url>
codex mcp get <name>
codex mcp remove <name>
```

- 管理外部 MCP server

#### 运行模式与安全

```bash
codex --full-auto
codex -s workspace-write
codex -a on-request
codex --dangerously-bypass-approvals-and-sandbox
```

- `--full-auto`：低摩擦自动执行
- `-s`：设置 sandbox
- `-a`：设置审批策略
- 最后一个参数非常危险，只有你明确知道风险时才该用

### 2.2 Claude Code 常用命令

#### 启动与基本使用

```bash
claude
```

- 进入交互式会话

```bash
claude "帮我分析这个项目"
```

- 直接带 prompt 启动

#### 非交互输出

```bash
claude -p "解释这个函数"
```

- `-p / --print`：打印结果后退出，适合管道和脚本

#### 恢复会话

```bash
claude -c
claude -r
claude -r <session-id>
```

- `-c / --continue`：继续当前目录最近一次会话
- `-r / --resume`：恢复某个历史会话

#### MCP 相关

```bash
claude mcp list
claude mcp add <name> <commandOrUrl>
claude mcp get <name>
claude mcp remove <name>
claude mcp serve
```

- 管理和启动 MCP 相关能力

#### Agents / 工作树

```bash
claude agents
claude --worktree
claude --tmux
```

- `agents`：查看已配置 agent
- `--worktree`：在新 git worktree 里工作
- `--tmux`：给工作树创建 tmux 会话

#### 权限与模式

```bash
claude --permission-mode plan
claude --permission-mode auto
claude --dangerously-skip-permissions
```

- 控制权限模式和执行风格

#### 其他常见管理命令

```bash
claude auth
claude update
claude plugins
```

- `auth`：认证
- `update`：更新
- `plugins`：插件管理

### 2.3 Codex 和 Claude Code 命令风格差异

- `Codex` 的命令风格更偏“agent runtime / sandbox / MCP / diff 应用”
- `Claude Code` 的命令风格更偏“会话恢复 / agents / plugins / slash commands / MCP”
- 两者都支持：
  - 交互式会话
  - 非交互执行
  - MCP 扩展
  - 权限/沙箱控制

如果面试官问二者差异，你可以说：

> 它们本质上都是 coding agent，但 Codex 更强调工具执行、sandbox 和会话应用链路，Claude Code 则更强调 agent 组织、skills / slash commands、plugin 和交互式工程体验。

---

## 3. OpenClaw / Ironclaw 框架怎么理解？

先说结论：

> 你可以把 OpenClaw / Ironclaw 理解成一个“本地优先、可扩展、带安全隔离的 agent 操作系统”。

公开资料里，`Ironclaw` 的 README 明确写了：

- 它是 **受 OpenClaw 启发的 Rust 重写版**
- 它强调 `security-first`
- 它把 `WASM sandbox`、`MCP`、`plugin architecture` 放在核心位置

### 3.1 用一张图先理解

根据 Ironclaw README 的架构图，可以把它拆成这几层：

```text
Channels / 入口层
  -> Agent Loop / Router
  -> Scheduler / Routines
  -> Worker / Orchestrator
  -> Tool Registry
  -> Safety Layer + Workspace Memory
```

### 3.2 每一层分别在干什么？

#### 1）入口层：Channels

入口层就是“用户从哪里接进来”。

README 里提到的入口包括：

- REPL
- HTTP webhook
- Web Gateway
- WASM channels

也就是说，这个框架不只服务一个 CLI，会把不同交互入口统一汇总到同一个 agent 核心。

#### 2）Agent Loop / Router

这一层相当于“大脑中枢”。

它负责：

- 收到用户输入
- 识别这是问答、命令、自动化任务，还是复杂工作流
- 决定下一步走哪个执行路径

你可以把它理解成：

- `Router` 负责“分流”
- `Agent Loop` 负责“主循环”

#### 3）Scheduler / Routines

这一层负责“任务编排”。

README 里提到：

- 并行 jobs
- cron
- event trigger
- webhook handler

这说明它不只是一个聊天机器人，而是一个能长期跑后台任务的 agent 框架。

也就是说，它支持：

- 即时响应
- 定时任务
- 事件驱动
- 并发执行

#### 4）Worker / Orchestrator

这一层是“真正干活的人”。

它负责：

- 跑 LLM 推理
- 发起工具调用
- 调度本地 worker
- 如果需要，还能把任务丢进隔离容器/隔离执行环境

README 里甚至提到了：

- Local workers
- Docker sandbox
- Orchestrator

所以它不是单线程问答，而是更像“任务执行平台”。

#### 5）Tool Registry

这一层是工具总线。

它把不同来源的能力统一起来，比如：

- built-in tools
- MCP tools
- WASM tools

所以从框架设计角度看，OpenClaw / Ironclaw 的一个核心思想是：

> 模型不是直接碰外部世界，而是统一通过工具层去做。

这样做的好处是：

- 权限可控
- 能做审计
- 好扩展
- 不同能力可以热插拔

#### 6）Safety Layer + Workspace

这一层负责两个关键词：

- 安全
- 持久化记忆

README 提到的安全能力包括：

- prompt injection defense
- leak detection
- endpoint allowlisting
- secrets injection at host boundary

同时它也有 workspace / hybrid search / memory 这些概念，说明它不是“问完就忘”的纯聊天系统，而是更像有长期上下文的 agent 工作台。

### 3.3 所以 OpenClaw / Ironclaw 本质上是什么框架？

如果你非要用一句工程化的话去概括：

> 它本质上是一个“多入口 + 任务编排 + 工具总线 + 安全隔离 + 持久化记忆”的 agent 框架。

如果再口语一点：

> 它不是单纯把 LLM 套个壳，而是把“会话、任务、工具、安全、记忆、执行环境”都做成了一个完整系统。

### 3.4 面试里怎么讲 OpenClaw / Ironclaw 的架构？

你可以这样说：

> 我理解 OpenClaw / Ironclaw 这类框架，不是单一的聊天应用，而是一个 agent runtime。它把入口层、任务路由、任务调度、执行 worker、工具注册、安全隔离和持久化记忆拆开了。这样做的好处是，一方面能接多种交互入口，另一方面能把工具调用和安全控制统一收口，所以比“直接让大模型调外部 API”更工程化，也更适合长期运行和扩展。

---

## 4. Ironclaw 为什么用 Rust 重写？Rust 为什么常被认为更安全？

先说结论：

> Rust 安全，不是因为它“不会出 bug”，而是因为它在语言层面提前拦掉了很多传统系统语言最常见、最危险的 bug。

### 4.1 Rust 最核心的安全来源：Ownership

Rust 官方书里讲得很清楚：

- Rust 用 `ownership` 管理内存
- 这些规则由编译器检查
- 违反规则，代码直接不过编译

这意味着：

- 很多内存错误不是“上线后炸”，而是“编译阶段就不让你过”

### 4.2 它主要避免哪些典型问题？

#### 1）double free

C/C++ 里一个经典问题是：

- 两个变量指向同一块堆内存
- 最后释放两次

Rust 通过 move / ownership 规则，避免同一块资源被多个所有者随便乱 free。

#### 2）dangling pointer / 野指针

Rust 的 reference / borrowing 机制保证：

- 引用必须在合法生命周期内使用
- 编译器会阻止悬垂引用

官方原话可以理解成：

- 引用像指针
- 但它保证在生命周期内指向的是有效值

#### 3）data race

Rust 还很强调并发安全。

在 safe Rust 里，想出现“多个线程同时读写同一块可变内存且没有同步保护”这种 data race，会非常难，很多情况编译器就直接拦掉了。

#### 4）空值和错误处理更显式

Rust 没有那种“默认到处都是 null 指针随便飞”的风格，常见做法是：

- `Option<T>` 表示值可能不存在
- `Result<T, E>` 表示成功或失败

这会逼着开发者更显式地处理边界情况。

### 4.3 Rust 为什么很适合 agent runtime 这类系统？

因为 agent runtime 一般有这几个特点：

- 长时间运行
- 要管理很多外部资源
- 并发和异步很多
- 要跟文件、网络、插件、沙箱打交道
- 一旦内存问题或权限边界出错，后果会很严重

所以 Rust 的优势会特别明显：

- 内存安全
- 并发安全
- 性能接近 C/C++
- 部署通常可以做成单二进制

这也是为什么 Ironclaw README 里把 `Rust vs TypeScript` 直接列成关键差异之一。

### 4.4 但要注意：Rust 安全 != 系统天然安全

这是面试里很加分的一点。

你可以主动补一句：

> Rust 主要解决的是内存安全和很多并发层面的低级错误，但它不能自动解决业务逻辑漏洞、权限设计错误、prompt injection、越权访问这些更高层的安全问题。所以工程上通常还是要再叠加沙箱、权限模型、审计和隔离。

---

## 5. 什么是 WASM 沙箱？为什么大家会说 “Rust + WASM” 很安全？

### 5.1 先说最直白的理解

WASM 沙箱就是：

> 把不完全可信的代码关进一个受限的小盒子里运行，让它只能做被明确允许的事。

这个“小盒子”不是虚拟机那种完整操作系统，而是更轻量的隔离运行环境。

### 5.2 WebAssembly 为什么天然适合做沙箱？

Wasmtime 安全文档里提到，WebAssembly 的一个核心目标就是：

- **在沙箱里安全执行不可信代码**

它的几个关键点是：

#### 1）内存隔离

WASM 实例有自己的线性内存（linear memory）。

它不是像原生程序那样随便拿真实地址乱跑，而是：

- 用 offset 访问自己的线性内存
- 越界会被检查

所以它天然更适合做隔离。

#### 2）没有默认系统调用权限

WASM 不能直接随便访问：

- 文件系统
- 网络
- 进程
- 主机内存

它只能调用宿主显式暴露给它的 imports。

这点特别重要：

> 没有被宿主开放的能力，它默认就拿不到。

这其实就是“默认拒绝”的思路。

#### 3）控制流更受约束

WASM 的函数跳转、调用目标都是受校验的，不像原生二进制那样更容易出现跳到非法地址、中间地址这种问题。

#### 4）更容易做能力最小化

比如宿主只给它：

- 访问某个白名单域名
- 读某个目录
- 调某几个工具

那它就只能做这些事。

这就是 capability-based security，能力按需授予。

### 5.3 Ironclaw 里的 WASM 沙箱可以怎么理解？

根据 Ironclaw README，它把不可信工具放进 WASM 容器里运行，并叠加了：

- capability-based permissions
- endpoint allowlisting
- credential injection at host boundary
- leak detection
- rate limiting
- resource limits

这说明它不是“只靠 WASM 本身”。

而是：

> WASM 提供基础隔离，框架再在外面叠加权限、配额、白名单、敏感信息保护这些防线。

这就是典型的 `defense in depth`，也就是纵深防御。

### 5.4 为什么说 Rust + WASM 是一个很自然的组合？

因为它们分别解决不同层面的问题：

#### Rust 负责什么？

- 让宿主程序自己更不容易出内存安全问题
- 让运行时、调度器、工具宿主管理层更稳

#### WASM 负责什么？

- 让“被执行的第三方代码/不可信工具”被隔离
- 限制它能访问哪些资源

所以：

- Rust 主要保护“宿主”
- WASM 主要隔离“客体”

把这两个组合起来，系统的整体安全性就会更强。

### 5.5 面试里可以怎么讲 WASM 沙箱？

你可以这样说：

> WASM 沙箱的核心，不是把代码简单编译成另一种格式，而是把不可信代码放进一个默认受限的执行环境里。它的内存是隔离的，对外部世界的访问必须通过宿主显式暴露的接口，而且宿主还可以继续叠加白名单、配额、超时和密钥注入边界。所以它特别适合做插件执行、工具执行和 agent 里的不可信代码隔离。

---

## 6. 如果面试官追问：为什么 Ironclaw 说自己比原版 OpenClaw 更“安全”？

根据 README 里给出的关键信息，可以从四层讲：

### 6.1 语言层

- Rust 相比 TypeScript/Node，更强调内存安全、并发安全、单二进制部署

### 6.2 执行层

- 用 WASM sandbox 代替更重的隔离方式，安全边界更清晰，也更轻量

### 6.3 权限层

- capability-based permissions
- endpoint allowlisting
- host boundary credential injection

### 6.4 防御层

- prompt injection defense
- leak detection
- audit log
- resource limits

所以比较稳的说法不是：

- “Rust 天生绝对安全”

而是：

> Ironclaw 把语言安全、执行隔离、权限控制和防御机制叠在一起了，所以整体安全设计更完整。

---

## 7. 面试可直接回答的精简版

### 7.1 function call、MCP、skills 的区别

> function call 是模型调用一个具体函数的结构化方式；MCP 是把外部工具、数据源和工作流统一接进 AI 的标准协议；skills 则是更高一层的能力封装，本质上是一套可复用的做事方法和约束。简单说，function call 管“调用动作”，MCP 管“能力接入”，skills 管“做事方法”。

### 7.2 OpenClaw / Ironclaw 是什么框架

> 我理解它本质上是一个 agent runtime，不只是聊天壳子。它把入口层、任务路由、调度、执行 worker、工具注册、安全层和持久化记忆都拆开了，所以更像一个能长期运行、可扩展、可控权限的 agent 操作系统。

### 7.3 为什么 Rust 安全

> Rust 的核心价值在于 ownership、borrowing 和编译期检查。很多 double free、悬垂引用、数据竞争这类底层问题，会在编译阶段直接被拦住，所以它特别适合做长期运行、并发多、资源边界复杂的系统。

### 7.4 什么是 WASM 沙箱

> WASM 沙箱就是把不可信代码关进一个默认受限的执行环境里。它的内存和宿主隔离，对文件、网络、密钥这些资源的访问都要宿主显式授权，所以很适合做插件执行和 agent 工具执行隔离。

---

## 8. 资料来源

### 官方 / 仓库

- OpenAI Tools / Function calling / Remote MCP: [https://developers.openai.com/api/docs/guides/tools](https://developers.openai.com/api/docs/guides/tools)
- MCP 官方介绍: [https://modelcontextprotocol.io/docs/getting-started/intro](https://modelcontextprotocol.io/docs/getting-started/intro)
- Claude Code Skills 文档: [https://code.claude.com/docs/en/slash-commands](https://code.claude.com/docs/en/slash-commands)
- Ironclaw README: [https://raw.githubusercontent.com/nearai/ironclaw/main/README.md](https://raw.githubusercontent.com/nearai/ironclaw/main/README.md)
- Ironclaw 仓库: [https://github.com/nearai/ironclaw](https://github.com/nearai/ironclaw)
- Rust 官方书 Ownership: [https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html)
- Rust 官方书 References and Borrowing: [https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html)
- Wasmtime Security: [https://docs.wasmtime.dev/security.html](https://docs.wasmtime.dev/security.html)

### 本机命令帮助（2026-03-17）

- `codex --help`
- `codex exec --help`
- `codex mcp --help`
- `claude --help`
- `claude mcp --help`
- `claude agents --help`

### 说明

- 关于 `OpenClaw` 原版的公开架构材料，相比 `Ironclaw` 没那么集中；所以上面“OpenClaw / Ironclaw 框架”的讲解，主要以 `Ironclaw README` 明确写出的架构图、核心组件和 `OpenClaw Heritage` 描述为准，再映射回原始框架思路。

---
