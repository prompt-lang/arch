# UMI (Unified Machine Interface)

> 本规范定义了 AGI 智能体之间、智能体与人类及外部系统之间的标准交互接口与行为模式，并以 PML 作为其声明式实现语言。任何符合 UMI 规范的智能体均可通过 PML 工作流进行描述、部署和互操作。


## 1. 引言

### 1.1 目标

UMI 旨在为异构智能体（AI、机器人、人类、服务）提供统一的交互抽象，包括：

- **通信协议**：标准化的消息格式与传输方式（请求-响应、发布-订阅、流式）。
- **交互模式**：人机协作、自我循环、多步任务、提示优先级等。
- **接口定义**：智能体的能力（Skills）以标准方式暴露和发现。
- **记忆与会话**：跨交互的上下文保持与长期知识存储。

PML 作为 UMI 的**规范语言**，提供声明式语法实现上述所有特性。

### 1.2 核心原则

- **接口与实现分离**：UMI 定义“契约”，PML 定义实现。
- **可发现**：智能体应能通过标准机制（如 MCP）获取对方的能力列表。
- **可组合**：交互模式可嵌套、串联，形成复杂会话。
- **可观测**：所有交互均有追踪、指标和日志。

---

## 2. 智能体模型与接口定义

### 2.1 智能体作为 PML 工作流

一个 UMI 智能体由 PML 工作流描述，其公共接口通过以下方式暴露：

| 暴露方式 | PML 实现 |
|----------|----------|
| **MCP Server** | 在 `config.mcp_server` 中声明，将工作流中的部分节点映射为 MCP 工具（tools）或资源（resources）。 |
| **gRPC/REST API** | 通过 `protocols` + `communication.request_reply` 暴露 RPC 方法。 |
| **事件总线** | 通过 `events` 和 `protocols` 中的消息队列发布事件。 |
| **人类代理** | 通过 `type: skill` 调用 Slack、WebSocket 等接口。 |

#### 示例：暴露 MCP 工具的智能体

```yaml
pml:
  version: "1.2"

meta:
  name: "weather_agent"

config:
  mcp_server:
    enabled: true
    name: "weather-service"
    tools:
      - name: "get_current_weather"
        description: "Get weather for a location"
        input_schema: { type: object, properties: { location: { type: string } } }
        handler: "pnode://get_weather"          # 指向工作流中的节点

workflow:
  input: { location: string }
  nodes: [get_weather]
  output: { temperature: "{{get_weather.output.temp}}" }

pnodes:
  - id: get_weather
    type: tool
    tool:
      name: "weather_api"
      endpoint: "https://api.weather.com/v1"
      params: { q: "{{workflow.input.location}}" }
```



### 2.2 智能体发现

- 静态发现：通过 PML 文件的 meta.name 和 meta.description 以及暴露的接口（如 config.mcp_server.tools）注册到服务目录（例如 Nacos、Consul）。

- 动态发现：智能体可发送 mcp_servers 中定义的 MCP 服务器的 tools/list 请求，获取对方能力。

## 3. 交互模式规范

下表列出 UMI 定义的交互模式及其在 PML 中的标准实现方式。

| 模式 | 描述 | PML 实现支持 |
|------|------|--------------|
| **A-A-Human** | 任务智能体与委托智能体协作，委托智能体负责人机交互 | `subworkflow` 分离逻辑，`type: skill` / `type: mcp` 对接人类界面 |
| **Self-Loop** | 智能体延迟自身提示，实现记忆回顾或周期性反思 | `type: timer` + `memory` 节点读历史状态 |
| **Multi-Step Task** | 多步骤分解与执行 | `type: planning` 生成计划 + `foreach` 依次执行 |
| **Seek-for-Prompt** | 信息不足时主动请求额外 Prompt | `type: tool` 查询外部信息，`control.on_error` 中的 fallback 策略 |
| **Prompt Rank** | 多个 Prompt 按优先级处理 | `control.priority` 设置节点优先级，调度器排序 |
| **Realtime Prompt** | 低延迟流式 Prompt 处理 | `communication.inbound` 订阅 WebSocket/SSE，`taskgraph: dynamic` 动态响应 |
| **Session** | 会话状态管理 | `workflow.vars` + `config.checkpoint` + `ttl_sec` |
| **Prompt Modeling** | 结构化 Prompt 表示 | `input`/`output` JSON Schema；`ippl` 机器表示 |
| **Request-Reply** | 同步双向通信 | `communication.request_reply` |
| **Memory** | 长期/短期记忆 | `type: memory` 节点 + `workflow.vars.ttl_sec` |

### 3.1 模式组合示例
以下工作流同时使用 Self-Loop、Memory 和 A-A-Human：
```
pml:
  version: "1.2"

config:
  memory:
    backends:
      - id: "local"
        type: key_value

workflow:
  vars: { last_intent: "", loop_count: 0 }
  nodes: [delegate, task, memory_recall, timer]

pnodes:
  - id: delegate
    type: skill
    skill: "human_interface"
    output: { user_input: "{{result}}" }

  - id: memory_recall
    type: memory
    operation: read
    key: "{{workflow.vars.last_intent}}"

  - id: task
    type: planning
    process:
      prompt: "当前目标：{{delegate.output.user_input}}，历史记忆：{{memory_recall.output.value}}"

  - id: timer
    type: timer
    control:
      delay: 3600
      repeat: true
    edges: [-> delegate]   # 每小时重新激活
```

## 4. 通信协议（UMC）

UMC (Unified Machine Connector) 是 UMI 的通信层实现，支持多种传输协议。在 PML 中通过 protocols 和 communication 声明。

### 4.1 支持的传输类型

| 传输类型 | PML 配置 | 典型场景 |
|----------|----------|----------|
| **stdio** | `transport: stdio` | 本地子进程通信 |
| **HTTP/REST** | `transport: http` | 对外 API |
| **gRPC** | `transport: grpc` | 高性能 RPC |
| **WebSocket** | `transport: websocket` | 双向实时流 |
| **Message Queue** | `type: mq` | 异步任务分发 |

### 4.2 请求-响应模式

用于同步调用。客户端节点必须等待响应，服务端节点自动处理请求。

服务端定义：

```
protocols:
  - id: "calc"
    type: rpc
    transport: grpc
    endpoint: "0.0.0.0:50051"

pnodes:
  - id: calculator
    type: function
    communication:
      request_reply:
        - id: "add"
          protocol: "calc"
          request_channel: "add_request"
          reply_channel: "add_response"
          handler: "do_add"   # 函数名
```


客户端调用：

```
pnodes:
  - id: client
    type: script
    script: |
      async function run(input, ctx) {
          const result = await ctx.requestReply("add", { a: 1, b: 2 });
          return { sum: result.sum };
    }
```

### 发布-订阅模式

通过 events 字段发布事件；订阅节点通过 communication.inbound 接收。

```
pnodes:
  - id: publisher
    type: script
    events: { emit: "data_ready" }

  - id: subscriber
    type: script
    communication:
      inbound:
        - protocol: "event_bus"
          event: "data_ready"
          handler: "onData"
```
## 5. 记忆支持

### 5.1 短期记忆（会话）
使用 workflow.vars 定义会话级变量，并设置 ttl_sec 自动过期。

工作流执行期间可读写，执行结束可选择持久化（通过 config.checkpoint）。

### 5.2 长期记忆

通过 type: memory 节点操作外部记忆后端（向量或键值）。

常用操作：写入 (write)、检索 (search)、读取 (read)、更新 (update)、删除 (delete)。

### 5.3 记忆与交互的结合

在 Self-Loop 模式中，智能体可以将当前状态写入记忆，下次激活时读取记忆，实现“回忆”。

## 6. 可观测性与治理
UMI 要求所有交互满足以下可观测性标准，PML 通过以下方式满足：

| 要求 | PML 支持 |
|------|----------|
| 分布式追踪 | `config.telemetry.otlp_endpoint`，每个节点自动生成 span |
| 指标 | `metrics` 字段，导出到 Prometheus |
| 结构化日志 | `logger` 自动注入 trace_id |
| 审计 | `hooks.on_node_end` 将节点执行记录到 audit 日志 |
| 健康检查 | `pml serve` 提供 `/health` 端点 |

## 7. 示例：完整 UMI 智能体

以下智能体PML集成了会话管理、长期记忆、优先级调度和实时交互。

```
pml:
  version: "1.2"

meta:
  name: "customer_support_bot"
  taskgraph: "dynamic"

config:
  memory:
    default_backend: "qdrant"
    backends:
      - id: "qdrant"
        type: vector
        endpoint: "http://qdrant:6333"

workflow:
  input: { user_id: string, message: string }
  vars:
    history: { type: array, ttl_sec: 1800 }
  nodes: [priority_router, retrieve_context, generate, store_memory]

pnodes:
  - id: priority_router
    type: script
    control: { priority: 1 }   # 高优先级
    script: |
      function run(input, ctx) {
          const msg = input.message;
          if (msg.includes("urgent")) return { route: "agent" };
          return { route: "bot" };
      }

  - id: retrieve_context
    type: memory
    operation: search
    vector: "{{embed(input.message)}}"
    limit: 3
    filter: { user_id: "{{workflow.input.user_id}}" }

  - id: generate
    type: llm
    process:
      messages:
        - role: system
          content: "历史记忆：{{retrieve_context.output.memories}}；会话历史：{{workflow.vars.history}}"
        - role: user
          content: "{{workflow.input.message}}"
    output: { reply: "{{result.content}}" }

  - id: store_memory
    type: memory
    operation: write
    value:
      query: "{{workflow.input.message}}"
      reply: "{{generate.output.reply}}"
    metadata: { user_id: "{{workflow.input.user_id}}" }
```



## Appendix

3.1 Reference Description Formats for simulation

[MuJoCo Modeling](https://mujoco.readthedocs.io/en/latest/modeling.html)  

[USD](https://developer.nvidia.com/usd)  

[URDF](http://wiki.ros.org/urdf)  



Reference 

[Prompt Chain](https://www.promptingguide.ai/zh/techniques/prompt_chaining)  

[RAG](https://ai.meta.com/blog/retrieval-augmented-generation-streamlining-the-creation-of-intelligent-natural-language-processing-models/)  

[Anthropic MCP](https://www.anthropic.com/news/model-context-protocol)  

[Google A2A](https://github.com/google/A2A)  

[OpenAI Swarm](https://github.com/openai/swarm)   

[APPL](https://github.com/appl-team/appl)    

[PromptWizard](https://arxiv.org/abs/2405.18369)  

[DSPy](https://github.com/stanfordnlp/dspy)

[RAGFlow](https://github.com/infiniflow/ragflow)  


