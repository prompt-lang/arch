# PML：回合会话、上下文分离、多步交互与步频控制

> 针对对话型、交互型和长时间运行的智能体系统，新增**会话对象**、**显式/隐式上下文分离**、**Inloop Session 模式**、**回合节点（MSCI）**以及**步频控制（StepFreq）**。这些扩展使 PML 能够声明式地管理多轮交互的共享上下文、自适应环境交互频率，并优化能耗与任务效率。

## 设计目标

- **会话对象化**：将一次对话或一段持续交互封装为会话实体，拥有独立 ID、参与者、话题、生命周期。
- **上下文分类融合**：区分显式上下文（直接提供）与隐式上下文（从记忆、标记、历史自动获取），运行时自动注入 LLM。
- **持续上下文交互**：支持 Inloop Session，会话期间上下文不释放，多步交互共享状态。
- **回合机制（MSCI）**：为多步上下文交互提供显式回合节点，支持回合计数、策略选择和回退。
- **步频动态调整**：智能体可实时调整与环境的交互频率，并关联能耗与效率指标。


## 核心概念

| 概念 | 说明 | PML 实现 |
|------|------|----------|
| **会话（Session）** | 封装一次持续交互的上下文，包含会话 ID、话题、参与者、显式上下文、隐式上下文、历史 | 顶层 `sessions` 字段 + `type: session` 节点 |
| **显式上下文** | 由工作流节点直接提供的 prompt 内容 | `llm.context.explicit` 字段 |
| **隐式上下文** | 从记忆、标记、会话历史、Agent 状态自动检索的内容 | `llm.context.implicit` 字段 |
| **Inloop Session** | 会话期间上下文不释放，支持无限轮交互直到显式关闭 | `session_mode: inloop` |
| **回合（Turn）** | MSCI 中的单步交互，包含用户输入、系统输出、回合状态 | `type: turn` 节点 |
| **步频（StepFreq）** | 智能体与环境的交互频率（Hz），可动态调整，影响能耗与响应速度 | `control.step_freq` + `ctx.setStepFreq()` |


## 语法与功能扩展

### 会话对象（Session）

在 `workflow` 顶层增加 `sessions` 字段，声明工作流中使用的会话。每个会话有唯一标识、类型、绑定设备或话题、TTL、检查点策略。

```
pnodes:
  - id: start_session
    type: session
    operation: create
    session_id: "{{workflow.input.session_id}}"
    context:
      explicit: { system_prompt: "You are a helpful assistant." }
      implicit_sources:
        - type: memory
          backend: "long_term"
          query: "user:{{workflow.input.user_id}}"
        - type: markup
          domain: "user_profile"
          filter: "active == true"
    output: { session_handle: "{{result.handle}}" }
```

示例（自然语言描述）：
- 会话 ID `customer_support`，类型 `conversation`。
- 绑定设备 `mobile_app`，话题 `billing`。
- TTL 为 3600 秒（空闲超时）。
- 启用检查点，间隔 60 秒。

在工作流内部，可以通过新增的 `type: session` 节点创建或恢复会话。例如，一个名为 `start_session` 的节点：操作类型为 `create`，会话 ID 从输入获取；显式上下文中包含系统提示；隐式上下文源包括从长期记忆中检索用户相关条目，以及从用户画像标记域中筛选活跃记录。输出为会话句柄。

其他节点可通过该句柄引用会话，自动读写其上下文和历史。

### 显式与隐式上下文分离

`type: llm` 节点新增 `context` 字段，分别配置显式和隐式源。

- **显式上下文**：直接提供字符串或数据绑定。
- **隐式上下文**：指定从记忆、标记、会话中检索规则。

例如：
- 显式部分：包含 system 角色指令和当前用户消息。
- 隐式部分：从长期记忆检索与话题相关的 top 3 条目；从标记域获取用户偏好。
- 运行时，PML 自动将隐式内容格式化为系统提示，追加到显式上下文之前，然后发送给 LLM。

### Inloop Session 模式

在 `workflow` 中设置 `session_mode: inloop`，表示整个工作流是一个持续会话，上下文在多次外部调用之间不释放。典型应用：聊天机器人、游戏 AI。

参数：
- `max_turns`：可选，最大交互轮数。
- `timeout_sec`：空闲超时，超时后自动归档会话。

工作流内部的 `type: llm` 节点会自动继承会话上下文，无需手动传递历史。

### 回合节点（MSCI）

新增节点类型 `type: turn`，专门用于实现多步上下文交互的每一步。它内置回合计数、回合策略和结束条件。

例如：
- 节点 `dialogue_turn` 类型为 `turn`。
- 输入：用户消息 `msg`，上一轮系统回复 `last_reply`（可选）。
- 输出：系统回复 `reply`，是否结束回合 `end_turn`，可信度 `confidence`。
- 控制：最大轮次 10，超时 30 秒；回合策略 `balanced`（另有 `greedy`, `explore` 可选）。
- 内部使用 LLM 生成回复，并自动更新会话历史。

`type: turn` 节点可以像普通节点一样放在 `control.loop` 中，实现多轮对话直到满足终止条件。

### 步频控制（StepFreq）

在 `control` 中增加 `step_freq` 字段（单位 Hz），表示该节点期望的执行频率。例如 `step_freq: 10` 表示每秒执行 10 次。

- 对于 `type: timer` 节点，`step_freq` 替代 `delay` 提供更精确的频率控制。
- 对于循环节点（如 `foreach` 批量处理），`step_freq` 控制批次之间的间隔。
- 脚本内可通过 `ctx.setStepFreq(newFreq)` 动态调整频率，调整会立即影响调度器。

步频调整可以结合 `metrics` 作为奖励信号。例如：
- 配置节点 `sensor_reader` 初始频率 30 Hz。
- 当任务进展平稳时，降低频率至 1 Hz（记录能耗降低的奖励）。
- 当检测到异常时，提升频率至 60 Hz。

NNOS 调度器确保实际执行频率不超过请求值，并自动计算能耗指标（`node.energy_consumption`）。


## 与现有 PML 特性的集成

- **会话与记忆**：会话的隐式上下文源可指向 `memory` 后端（长期记忆），实现跨会话知识复用。
- **会话与标记**：会话状态可写回标记域（例如更新用户偏好标记）。
- **回合与评估**：`type: turn` 节点可内嵌 `evaluation`，评估每轮回复质量，决定是否提前终止。
- **步频与资源控制**：`step_freq` 与 `resources` 中的 CPU 配额配合，确保高频节点不超限。


## 完整示例：智能客服机器人（Inloop Session + 回合 + 步频）

以下示例展示一个集成了会话、回合和动态步频的客服机器人工作流。

```
# PML v1.15 完整示例：智能客服机器人（Inloop Session + 回合 + 步频）

sessions:
  - id: support_bot
    type: conversation
    device: web_chat
    topic: product_issue
    ttl_sec: 3600
    checkpoint_interval: 60
    implicit_sources:
      - type: memory
        backend: long_term
        query: "user:{{workflow.input.user_id}}"
      - type: markup
        domain: user_profile
        filter: "active == true"

workflow:
  session_mode: inloop
  max_turns: 20
  timeout_sec: 300
  input:
    user_message: string
    user_id: string
  vars:
    turn_count: 0
    session_active: true
  nodes: [process_turn, adjust_freq, maybe_sleep, end_session]
  edges:
    - process_turn -> adjust_freq
    - adjust_freq -> maybe_sleep
    - maybe_sleep -> end_session

pnodes:
  - id: process_turn
    type: turn
    input:
      msg: "{{workflow.input.user_message}}"
    output:
      reply: "{{result.reply}}"
      end_turn: "{{result.end_turn}}"
      confidence: "{{result.confidence}}"
    control:
      max_turns: 20
      timeout: 30
      strategy: balanced
    llm_config:
      model: gpt-4
      temperature: 0.7

  - id: adjust_freq
    type: script
    script: |
      function run(input, ctx) {
          let turn = ctx.state.get("workflow.vars.turn_count");
          let newFreq = turn < 5 ? 2.0 : 0.2;
          ctx.setStepFreq("process_turn", newFreq);
          ctx.state.set("workflow.vars.turn_count", turn + 1);
          return { freq: newFreq };
      }

  - id: maybe_sleep
    type: timer
    control:
      step_freq: 0.5        # 每2秒检查一次空闲超时
    script: |
      function run(input, ctx) {
          let lastActive = ctx.state.get("workflow.vars.last_active_time");
          if (Date.now() - lastActive > 300000) {
              ctx.trigger("end_session");
          }
      }

  - id: end_session
    type: script
    script: |
      function run(input, ctx) {
          let sessionId = ctx.state.get("workflow.vars.session_id");
          // 保存完整会话历史到长期记忆
          ctx.memory.write("long_term", sessionId, ctx.session.getHistory());
          ctx.session.close();
          ctx.terminateWorkflow("session_end");
      }
```

首先定义会话：ID `support_bot`，类型 conversation，绑定设备 `web_chat`，话题 `product_issue`，隐式上下文源为长期记忆（查询用户历史问题）和用户偏好标记。

工作流设置 `session_mode: inloop`，最大轮次 20，空闲超时 300 秒。

工作流输入：`user_message` 字符串。

变量：`turn_count` 初始 0，`session_active` 初始 true。

节点：
- `process_turn`：类型 `turn`，输入用户消息，输出系统回复和是否结束回合。控制回合最大 20，策略 `balanced`。
- `maybe_sleep`：类型 `timer`，控制频率为 `step_freq: 0.5`（每 2 秒检查一次空闲超时）。
- `adjust_freq`：类型 `script`，根据当前 `turn_count` 动态调整步频：前 5 轮高频率 2 Hz，后期降低到 0.2 Hz 以节能。
- `end_session`：当用户说“再见”或达到最大轮次时，关闭会话并保存历史到长期记忆。

执行流程：用户每发送一条消息，`process_turn` 节点被触发，生成回复；同时 `adjust_freq` 节点根据轮次调整后续交互频率；空闲超时或用户主动结束则归档会话。


## 核心特性

通过引入 **会话对象**、**上下文分离与融合**、**Inloop Session 模式**、**回合节点**以及 **步频控制**，使 PML 能够声明式地构建复杂、长时间运行、自适应交互频率的智能体系统。这些扩展特别适用于：

- 客服机器人与多轮对话系统
- 游戏 AI 与回合制策略
- 实时感知-决策循环（如自动驾驶、机器人导航）
- 资源受限环境下的能效优化

所有特性均向后兼容，并为未来更高级的交互策略（如强化学习调频）打下基础。
