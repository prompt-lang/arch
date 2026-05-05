# Prompt-Lang PNode  

## PNode 规范扩展：面向 AGI 任务执行的任务执行节点

> 本规范基于PML，定义 **PNode** 作为神经任务节点的角色、行为以及与 AGI 模型的交互协议。PNode 不进行复杂推理（Thought），但能高效执行 AGI 模型下发的模板任务，并通过 **PNode Protocol** 接受控制指令（如中断、模板更新）。


## 1. 设计原则

- **轻量快速**：PNode 专为计算密集型、低延迟的任务设计，避免 LLM 调用或复杂决策。
- **模板驱动**：AGI 模型输出带有占位符的 **Prompt Program**，PNode 根据当前 State 填充并执行。
- **受控执行**：AGI 可通过 PNode Protocol 动态调整 PNode 行为（启动、暂停、恢复、终止、更新模板）。


## 2. PNode 类型扩展

在已有 PML 节点类型基础上，增加一种新的 `type: neuro_task`（神经任务节点），用于表示此类高效计算节点。

```
pnodes:
  - id: fast_calculator
    type: neuro_task
    description: "快速执行数学运算，无 LLM 调用"
    process:
      template: |
        def compute(x, y):
            return { "result": x + y }
    input:
      x: number
      y: number
    output:
      schema:
        result: number
    control:
      timeout: 5
      priority: 10
```

### 2.1 与传统节点的区别

| 特性 | 普通 PNode (llm, tool, script) | 神经任务节点 (neuro_task) |
|------|-------------------------------|---------------------------|
| 推理能力 | 可能包含 LLM 或复杂逻辑 | 无，仅执行确定性/预定义计算 |
| 执行速度 | 可变，LLM 节点较慢 | 极快（微秒级到毫秒级） |
| 模板格式 | 自由文本或脚本 | 严格的 Prompt Program（一种结构化任务描述） |
| 与 AGI 交互 | 被动接收输入 | 支持主动状态协议（中断、更新） |


## 3. Prompt Program 规范

AGI 模型下发给 PNode 的任务模板称为 Prompt Program。它是一种轻量级指令语言，以 JSON 或 YAML 形式描述计算图或函数调用。

### 3.1 基本结构

```
program:
  version: "1.0"
  entry: "main"
  functions:
    - name: "main"
      type: "expression"
      expression: "func(input.x, input.y)"
    - name: "add"
      type: "builtin"
      name: "add"
```

### 3.2 调用模型

AGI模型给出Function调用的Index,PNode Program执行调用

### 3.3 模板填充

PNode 执行时，将 input 中的值代入 Prompt Program 的占位符（如 {{input.x}}），生成具体计算任务并执行。填充后的程序不包含任何未绑定变量。

## 4. PNode Protocol

PNode Protocol 定义了 AGI 模型与 PNode 之间的控制通道，用于管理 PNode 的生命周期和动态行为。

### 4.1 支持的操作


| 操作 | 描述 | 方向 |
|------|------|------|
| `start` | 启动 PNode 执行（若未启动） | AGI → PNode |
| `pause` | 暂停当前执行（可恢复） | AGI → PNode |
| `resume` | 恢复暂停的执行 | AGI → PNode |
| `terminate` | 终止执行，释放资源 | AGI → PNode |
| `update` | 动态更新 PNode 的 `process.template` 或 `input` 绑定 | AGI → PNode |
| `status` | 查询当前状态（idle, running, paused, terminated） | AGI ↔ PNode |
| `heartbeat` | 检测 PNode 是否存活 | AGI → PNode |

### 4.2 PML 中的协议声明

在 PNode 的 control 字段中增加 protocol 子字段，启用 PNode Protocol。

```
pnodes:
  - id: compute_node
    type: neuro_task
    control:
      protocol:
        enabled: true
        endpoint: "ipc:///tmp/pnode_compute.sock"   # 本地 IPC 或网络地址
        heartbeat_interval: 30                      # 心跳间隔（秒）
        auto_restart: true                          # 崩溃后自动重启
    process:
      template: |
        def run(input):
            return { "sum": input.a + input.b }
```

运行时，NNOS 会为 pnode_task 节点创建独立的进程或线程，并监听上述协议端点。

### 4.3 AGI 模型调用协议示例

AGI 模型（通常由另一个工作流或外部系统驱动）可通过标准协议消息控制 PNode。

启动 PNode：
```
{
  "operation": "start",
  "node_id": "compute_node",
  "input": { "a": 10, "b": 20 }
}
```
更新模板：
```
{
  "operation": "update",
  "node_id": "compute_node",
  "process": {
    "template": "def run(input): return { \"product\": input.a * input.b }"
  }
}
```
查询状态
```
{
  "operation": "status",
  "node_id": "compute_node"
}
```
响应
```
{
  "status": "running",
  "progress": 0.5,
  "last_output": null
}
```

### 4.4 集成到 PML 工作流

一个 AGI 工作流可以通过 type: mcp 或 type: tool 节点与 PNode Protocol 端点通信，从而控制远程/本地的神经任务节点。
```
pnodes:
  - id: agi_controller
    type: llm
    process:
      prompt: "根据用户需求，控制 compute_node 执行任务"
    output: { control_cmd: "{{result.json}}" }

  - id: protocol_client
    type: tool
    tool:
      name: "pnode_protocol"
      endpoint: "ipc:///tmp/pnode_compute.sock"
      method: "send"
      params: { command: "{{agi_controller.output.control_cmd}}" }
```

## 5. 与 UMI 交互模型的关系

PNode 作为 UMI 中 高效任务执行单元，与 AGI 模型构成标准 UMI 交互模式：

A-A-Human：PNode 可视为 Task Agent 的底层执行器，由 Delegate Agent 通过 Protocol 控制。

Self-Loop：PNode 可由定时器周期性执行，无需 AGI 持续干预。

Multi-Step Task：PNode 可重复执行模板，处理流水线数据。

Realtime Prompt：PNode 通过 Protocol 的 update 操作实时调整行为，无需重启。

## 6. 实现注意事项

隔离性：PNode 应运行在独立的轻量级沙箱（线程/进程/Wasm）中，避免影响 NNOS 主调度器。

资源限制：通过 control.resources 限制 CPU、内存，防止失控。

协议安全性：PNode Protocol 端点应支持认证（如 IPC 仅允许本地进程，网络端点启用 mTLS）。

模板执行语言：推荐使用 WebAssembly 或受限的 JavaScript 解释器，确保快速且安全。

状态持久化：PNode 的当前执行状态（如暂停点）可序列化到 config.checkpoint，用于故障恢复。


## 7. 完整示例

以下 PML 工作流演示 AGI 模型动态下发模板并控制 PNode。

```
pml:
  version: "1.2"

meta:
  name: "agi_pnode_coordinator"

workflow:
  input: { user_query: string }
  nodes: [agi_planner, pnode_controller, pnode_executor]

pnodes:
  - id: agi_planner
    type: llm
    process:
      prompt: |
        用户需求：{{workflow.input.user_query}}
        生成一个 Prompt Program 模板（YAML 格式）用于计算任务。
    output: { template: "{{result.text}}" }

  - id: pnode_controller
    type: mcp
    mcp:
      server: "pnode_protocol_broker"
      tool: "send_command"
    input:
      node_id: "fast_math"
      operation: "update"
      process: { template: "{{agi_planner.output.template}}" }

  - id: pnode_executor
    type: neuro_task
    id: fast_math   # 实际部署的 PNode 实例
    control:
      protocol:
        enabled: true
        endpoint: "tcp://127.0.0.1:9001"
    process:
      template: "def run(input): return { 'result': 0 }"   # 初始模板，会被更新覆盖
    output: { result: "{{output.result}}" }
```



