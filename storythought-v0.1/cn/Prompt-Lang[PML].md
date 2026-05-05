# PML (Prompt-Lang Markup Language)

## 总览 (Overview)

### 定义

PML 是一种**声明式语言**，用于定义 AI 执行流程，其核心能力包括：

- 描述**工作流 (Workflow)**
- 定义**执行节点 (PNode)**
- 编排 **LLM 与非 LLM 系统**
- 支持**可控、可验证、可恢复**的执行过程
- 集成 **Skills** 与 **MCP (Model Context Protocol)** 等外部能力
- 提供**双重视图**（机器优化 ippl / 人类可读 ppl）
- 支持**显式节点通信协议**

### 设计目标

| 目标 | 说明 |
|------|------|
| **声明式** | 用户描述“做什么”，而非“如何做” |
| **确定性执行** | 相同输入 + 相同 DAG → 可预测输出（LLM 部分受模型随机性影响，但流程确定） |
| **可观测** | 日志、指标、追踪、评估、检查点 |
| **可组合** | PNode 可嵌套组合为更复杂的 PNode，Workflow 可复用 |
| **LLM-Native** | 原生支持 Prompt 模板、模型配置、动态规划、工具调用、多轮对话 |
| **安全可靠** | 补偿事务、熔断、权限约束、资源配额、沙箱执行 |
| **协议无关** | 通过适配层支持 MCP、OpenAPI、gRPC、Skills 等 |

### 执行流水线
```
PML (文本) → AST → IR → Execution Graph → Runtime (NNOS)
               ↑         ↑
         validate    graph export
         compile     serve editor
```

### 核心概念

| 概念 | 说明 |
|------|------|
| **Workflow** | 有向无环图 (DAG)，定义节点执行顺序与数据流 |
| **PNode** | 最小执行单元，支持 `llm`, `tool`, `function`, `planning`, `subworkflow`, `foreach`, `skill`, `mcp`, `openapi`, `grpc`, `script` 等多种类型 |
| **State** | 全局执行上下文，所有值均为不可变快照 |
| **TaskGraph** | 由 Workflow + PNode 实例化得到的可执行图，分为静态图、动态图、混合图 |
| **PMLScript** | 嵌入在 PML 中的脚本（JS/Python/Shell），用于自定义逻辑、生成 CLI 命令 |
| **Skill** | 可复用的能力单元，可来源于 MCP Tool、Subworkflow、OpenAPI 等 |
| **IPP / PPL** | 双重视图：`ippl` 面向 LLM/机器优化，`ppl` 面向人类阅读 |
| **NNOS** | 运行时系统，支持调度、状态管理、重试、验证、资源配额、追踪、补偿、脚本执行等 |

---

## 语法规范 (Syntax Specification)

### 根结构

```yaml
pml:
  version: "1.0"

meta:
  name: string
  dialect: string
  description: string
  taskgraph: static | dynamic | hybrid
  compatible_with: [string]
  dynamic_policy: {...}
  script_engine: {...}

config:
  default_model: string
  retry: number
  timeout: number
  resources: {}
  hooks: {}
  telemetry: {}
  checkpoint: {}
  metrics: {}
  script_policy: {}
  mcp_server: {}       # 可选，将工作流暴露为 MCP Server

workflow:
  input: {}
  vars: {}
  output: {}
  nodes: []
  edges: []
  metrics: []
  evaluation: []

pnodes: []

# 新增顶层模块
skills: []
mcp_servers: []
scripts: []
protocols: []
command_generators: []
evaluation_experiment: {}
evaluation_templates: []
secrets: {}
permissions: {}
test: {}
```
## 分层设计  

### 1. Meta Layer   

```
meta:
  name: "example_workflow"
  dialect: "agentic"
  description: "An adaptive data analysis workflow"
  taskgraph: "dynamic"
  compatible_with: ["0.6", "0.7", "1.0"]
  dynamic_policy:
    allowed_node_types: [llm, tool, subworkflow]
    max_total_nodes: 20
    max_depth: 3
    disallowed_tools: ["delete_database"]
  script_engine:
    default_language: "javascript"
    allowed_languages: ["javascript", "python"]
    env: { NODE_ENV: "production" }
    runtimes: { javascript: "node:18", python: "python:3.11" }
    sandbox:
      allow_network: false
      allow_filesystem: false
      memory_limit_mb: 128
      timeout_sec: 10
```

### 2. DAG Layer
```
workflow:
  input: { query: string, context: object }
  vars: { counter: { type: integer, default: 0 } }
  output: { final_answer: "{{analyze.output.summary}}" }
  nodes: [search, analyze, report]
  edges:
    - search -> analyze
    - search -> report          # 数据边
    - source: search
      target: logger
      type: event
      event_name: "search_done"
  metrics:
    - name: "workflow_duration_sec"
      type: histogram
      value: "{{workflow.duration_sec}}"
  evaluation:
    - name: "final_answer_accuracy"
      type: exact_match
      reference: "{{workflow.input.expected}}"
      candidate: "{{workflow.output.final_answer}}"
      threshold: 1.0
      on_fail: "log"
```

### 3. PNode 规范

每个 PNode 包含以下字段（部分可选）：

```
id: string
type: string                     # 见下方类型列表
input: {}                         # 输入映射
process: {}                       # 执行细节
output: {}                        # 输出 schema
control: {}                       # 重试、超时、on_error、circuit_breaker
runtime: {}                       # 异构执行环境
compensate: string                # 补偿节点 ID
events: {}                        # 发布/订阅
metrics: []                       # 节点级指标
evaluation: []                    # 节点级评估
communication: {}                 # 显式通信协议
ippl: string | object             # 机器/LLM 优化表示
ppl: string                       # 人类可读表示
```

支持的节点类型

| 类型 | 说明 | 关键字段 |
|------|------|----------|
| `llm` | LLM 调用 | `process.llm_mode`, `prompt`, `messages`, `tools` |
| `tool` | 外部工具 (HTTP/gRPC) | `tool.name`, `endpoint`, `params` |
| `function` | 确定性函数 | `function.language`, `code` |
| `planning` | 动态生成子工作流 | `process.prompt`, `control.max_sub_nodes` |
| `subworkflow` | 复用已定义工作流 | `workflow` (引用) |
| `foreach` | 并行循环 | `input_items`, `iteratee`, `concurrency` |
| `skill` | 调用预定义 Skill | `skill` (引用) |
| `mcp` | 调用 MCP Server 工具 | `mcp.server`, `mcp.tool` |
| `openapi` | 调用 OpenAPI 操作 | `openapi.url`, `openapi.operation` |
| `grpc` | 调用 gRPC 方法 | `grpc.proto`, `grpc.service`, `grpc.method` |
| `script` | 执行 PMLScript | `script` (引用) 或内联 `code` |


### LLM 节点增强（LLM Modes）

在 `process` 中使用 `llm_mode` 选择交互模式：

```
pnodes:
  - id: chat_assistant
    type: llm
    process:
      llm_mode: chat
      model: "gpt-4"
      messages:
        - role: system
          content: "You are a helpful assistant."
        - role: user
          content: "{{workflow.input.question}}"
      messages_from: "{{workflow.vars.history}}"   # 从状态加载历史

  - id: stream_writer
    type: llm
    process:
      llm_mode: stream
      prompt: { user: "Write a story" }
    control:
      stream_callback: "handle_chunk"

  - id: tool_planner
    type: llm
    process:
      llm_mode: tool_calls
      tools:
        - type: function
          function:
            name: "get_weather"
            parameters: { type: object, properties: { location: { type: string } } }
```
若无 `llm_mode`，默认 `completion`。

## 通信协议 (Communication Protocol)   

节点可声明 `communication` 字段，实现显式消息传递。

### 顶层协议声明  

```
protocols:
  - id: "task_queue"
    type: mq
    provider: "rabbitmq"
    url: "amqp://localhost"
    exchange: "tasks"
  - id: "rpc_endpoint"
    type: rpc
    transport: grpc
    endpoint: "localhost:50051"
```
### 节点级通信  

```
pnodes:
  - id: producer
    type: function
    communication:
      outbound:
        - protocol: "task_queue"
          queue: "jobs"
          routing_key: "task.created"

  - id: consumer
    type: function
    communication:
      inbound:
        - protocol: "task_queue"
          queue: "jobs"
          concurrency: 3
          handler: "process_job"
```

支持模式：请求-响应、发布-订阅、消息队列、共享内存。通信协议与数据边（`edges`）共存。

## 双重视图：ippl 与 ppl  

每个 PNode 可携带两种视图：

```
pnodes:
  - id: web_search
    type: tool
    # 核心定义...
    ippl:
      type: "tool"
      operation: "search"
      parameters: ["query"]
      execution_hint: "idempotent"
    ppl: |
      ## Web Search
      This node searches the web using the provided query.
      **Input**: query (string)
      **Output**: results (array)
```

- 运行时忽略这两个字段。   

- CLI 支持 `ppcli show node --human` 和 `--machine`。  

## Skills 与 MCP 集成  

### Skill 定义  

```
skills:
  - id: "send_email"
    type: tool
    tool: { name: "email_sender", endpoint: "https://api.mail.com/send" }
    input_schema: { to: string, subject: string, body: string }
    output_schema: { status: string }
  - id: "calculate_risk"
    type: subworkflow
    source: "./risk_calculator.pml"
```

### 使用 Skill

```
pnodes:
  - id: notify
    type: skill
    skill: "send_email"
    input: { to: "user@example.com", subject: "Done", body: "Workflow finished" }
```

## MCP Server 配置

```
mcp_servers:
  - id: "filesystem"
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/workspace"]
    transport: stdio
    timeout_sec: 30
```

### 调用 MCP 工具
```
pnodes:
  - id: read_file
    type: mcp
    mcp:
      server: "filesystem"
      tool: "read_file"
      arguments: { path: "/workspace/data.txt" }
```

## PMLScript（嵌入式脚本）

```
scripts:
  - id: data_cleaner
    language: "python"
    code: |
      def run(input, ctx):
          return { "cleaned": input["data"].strip() }

pnodes:
  - id: clean
    type: script
    script: "data_cleaner"
    input: { data: "{{fetch.output.raw}}" }
```

脚本可访问 `state`, `secrets`, `logger`，支持异步，并能生成 CLI 命令（通过 `__cli_command` 返回值）。

## Metrics & Evaluation  
### Metrics  
```
workflow:
  metrics:
    - name: "total_tokens"
      type: counter
      value: "{{workflow.total_tokens}}"

pnodes:
  - id: llm_node
    metrics:
      - name: "latency_ms"
        type: histogram
        value: "{{node.duration_ms}}"
        buckets: [10, 50, 100, 500]
```

### Evaluation  

```
evaluation_templates:
  - id: "quality_check"
    evaluations:
      - type: rouge
        threshold: 0.5

pnodes:
  - id: generate_summary
    evaluation_template: "quality_check"
    evaluation:
      - name: "custom_score"
        type: custom
        script: "my_evaluator"
        threshold: 0.8
        on_fail: "retry"
```

## 工具链 CLI
```
ppcli compile workflow.pml -o workflow.ir
ppcli run workflow.ir --input data.json
ppcli validate workflow.pml --strict
ppcli test workflow.pml --case basic
ppcli migrate --from 1.0 --to 1.1 workflow.pml
ppcli profile workflow.ir --input data.json
ppcli graph workflow.ir --format svg --output dag.svg
ppcli serve --port 8080
ppcli generate-cli workflow.pml --output ./mycli
ppcli skill install send_email
ppcli mcp start filesystem
ppcli import openapi petstore.yaml --output skills.yaml
```

## 运行时 (NNOS) 支持特性

- DAG 调度：拓扑排序 + 并行执行 + 优先级队列

- 状态管理：不可变快照，支持检查点

- 错误处理：重试、熔断、回退、补偿

- 资源控制：token 限制、内存、成本预算

- 可观测性：日志、指标、分布式追踪

- 安全：沙箱、权限检查、密钥管理

- 脚本执行：JS/Python 隔离环境

- 通信适配器：MQ、gRPC、HTTP 等

- MCP 集成：支持 stdio/SSE/WebSocket

## 完整示例

```
pml:
  version: "1.0"

meta:
  name: "intelligent_document_processor"
  dialect: "agentic"
  taskgraph: "dynamic"
  script_engine:
    default_language: "python"

config:
  default_model: "gpt-4"
  resources: { max_concurrent_nodes: 3 }
  telemetry: { otlp_endpoint: "http://otel:4318" }
  metrics: { enabled: true }

workflow:
  input: { document: string, expected_answer: string }
  vars: { quality_scores: [] }
  output: { final_reply: "{{generate.output.text}}" }
  nodes: [extract, plan, search, generate, evaluate]
  edges:
    - extract -> plan
    - plan -> search
    - plan -> generate
    - search -> generate
    - generate -> evaluate
  metrics:
    - name: "workflow_duration"
      type: histogram
      value: "{{workflow.duration_sec}}"

skills:
  - id: "web_searcher"
    type: mcp
    mcp:
      server: "web_search_mcp"
      tool: "search"

mcp_servers:
  - id: "web_search_mcp"
    command: "python"
    args: ["-m", "mcp_server_web"]
    transport: stdio

pnodes:
  - id: extract
    type: script
    script: "text_extractor"
    input: { document: "{{workflow.input.document}}" }
    output: { extracted_text: "{{result.text}}" }
    ppl: "Extracts plain text from document."

  - id: plan
    type: planning
    process:
      prompt: "Given extracted text: {{extract.output.extracted_text}}, plan a research strategy."
    control: { max_sub_nodes: 5 }
    ippl:
      type: "planning"
      inputs: ["extracted_text"]
      outputs: ["sub_workflow"]

  - id: search
    type: skill
    skill: "web_searcher"
    input: { query: "{{plan.output.search_query}}" }
    output: { results: "{{response}}" }

  - id: generate
    type: llm
    process:
      llm_mode: chat
      messages:
        - role: system
          content: "You are an expert summarizer."
        - role: user
          content: "Based on: {{search.output.results}}, write a concise answer."
    evaluation:
      - name: "rouge_score"
        type: rouge
        reference: "{{workflow.input.expected_answer}}"
        candidate: "{{output.text}}"
        threshold: 0.6
        on_fail: "retry"

  - id: evaluate
    type: script
    script: "compute_bert_score"
    input:
      ref: "{{workflow.input.expected_answer}}"
      cand: "{{generate.output.text}}"
    output: { bertscore: "{{result.f1}}" }

scripts:
  - id: text_extractor
    language: "python"
    code: |
      def run(input, ctx):
          return { "text": input["document"][:1000] }
  - id: compute_bert_score
    language: "python"
    code: |
      from sentence_transformers import SentenceTransformer, util
      model = SentenceTransformer('all-MiniLM-L6-v2')
      def run(input, ctx):
          emb1 = model.encode(input['ref'])
          emb2 = model.encode(input['cand'])
          sim = util.cos_sim(emb1, emb2).item()
          return { "f1": sim }

test:
  cases:
    - name: "happy_path"
      input: { document: "AI is great.", expected_answer: "AI is great." }
      expected:
        generate.output.text: { exists: true }
      mocks:
        - node: "search"
          output: { results: "Mock search result" }
```

## 版本兼容性  

ppcli migrate 工具自动升级。



ref:

[A2UI](https://github.com/google/A2UI)
