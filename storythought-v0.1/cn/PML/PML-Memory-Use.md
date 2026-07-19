# PML记忆增强
> 针对记忆使用（Memory-Use）进行深度增强。新增**记忆模式声明**（确定性/不确定）、**情景记忆**（Episodic Memory）支持、**MemPML 高效索引视图**、**记忆事务**、**记忆可观测性**指标，以及基于 MemPML 的**PEM 工程记忆应用**（对标 Anthropic Claude Code Rules 体系）。这些扩展使 PML 的记忆子系统更贴近认知科学模型，显著提升大规模智能体系统的存储利用效率。


## 1. 记忆模式声明：确定性记忆与不确定记忆

### 1.1 背景

- **确定性记忆**：存储高可信、低噪声的事实，如知识库、编译结果。适合精确查询。
- **不确定记忆**：存储带置信度的观测、预测或传感器数据，可融合多个信源。

### 1.2 语法扩展

在 `memory_backends` 或 `config.memory.backends` 中增加 `mode` 字段。

- `mode: deterministic`：默认值，条目不自动携带置信度。
- `mode: uncertain`：每条写入的记忆自动生成 `confidence` 字段（0~1），也可由用户覆盖。

```
memory_backends:
  - id: "fact_db"
    mode: "deterministic"
  - id: "sensor_fusion"
    mode: "uncertain"
    default_confidence: 0.7
```

示例中，一个后端名为 `fact_db` 的模式为 deterministic，另一个后端名为 `sensor_fusion` 的模式为 uncertain，默认置信度为 0.7。

在 `type: memory` 节点写入时，可显式指定 `confidence` 值。例如，在节点 `write_obs` 中，向 `sensor_fusion` 后端写入位置 [10,20]，并设置置信度为 0.85。

```
pnodes:
  - id: write_obs
    type: memory
    operation: write
    backend: "sensor_fusion"
    value: { position: [10,20] }
    confidence: 0.85
```

查询时可按置信度过滤，例如在节点 `reliable_only` 中，搜索 `sensor_fusion` 后端，要求置信度不低于 0.8。

```
- id: reliable_only
  type: memory
  operation: search
  backend: "sensor_fusion"
  confidence_min: 0.8
```

## 2. 情景记忆（Episodic Memory）

### 2.1 概念

情景记忆按时间顺序存储智能体的经历，支持按时间窗口、事件上下文检索。区别于语义记忆（事实），情景记忆强调时间线。

### 2.2 新增后端类型


`type: episodic` 后端自动为每条记忆添加时间戳（`timestamp`）、事件标签（`event_tag`）、序列号（`seq`）。

```
memory_backends:
  - id: "user_experiences"
    type: "episodic"
    retention_days: 30
```

示例：定义后端 `user_experiences`，类型为 episodic，保留 30 天。

### 2.3 写入与查询

写入时自动附加时间元数据，也可手动指定 `event_tag`。

例如节点 `log_action` 向后端 `user_experiences` 写入值 `{ action: "click", target: "button" }`，并指定事件标签 `ui_interaction`。

```
pnodes:
  - id: log_action
    type: memory
    operation: write
    backend: "user_experiences"
    value: { action: "click", target: "button" }
    event_tag: "ui_interaction"
```

查询支持时间范围、事件标签过滤。
```
- id: recent_events
  type: memory
  operation: search
  backend: "user_experiences"
  time_range: ["2025-05-01T00:00:00Z", "2025-05-07T23:59:59Z"]
  event_tag: "ui_interaction"
  limit: 100
```
例如节点 `recent_events` 在 `user_experiences` 中搜索时间范围从 2025-05-01 到 2025-05-07，事件标签为 `ui_interaction`，限制最多 100 条。返回结果自动按时间升序排列，并包含 `timestamp` 字段。


## 3. MemPML：高效记忆索引视图

### 功能

MemPML 是一种数据视图模式，用于从现有记忆后端中提取、转换、聚合数据，形成专用的索引结构，加速模型检索。类似于数据库的物化视图。

### 3.2 新增节点 `type: mem_view`

```
pnodes:
  - id: active_user_view
    type: mem_view
    source: "user_profiles"               # 源记忆后端 ID
    projection: ["user_id", "last_active", "preferences"]
    filter: "last_active > now - 7d"
    refresh_interval: 300                 # 秒，自动刷新
    output: { view_id: "{{result.view_id}}" }
```
- source：已有的记忆后端（可以是 deterministic, uncertain 或 episodic）。

- projection：选择需要包含的字段。

- filter：类 SQL 表达式，可通过 eval 引擎计算。

- refresh_interval：可选，定时刷新视图。

- 视图本身可被后续的 type: memory 查询直接引用：

```
- id: query_view
  type: memory
  operation: search
  backend: "active_user_view"            # 直接查询视图
  limit: 10
```

### 3.3 序列化格式 `mem_pml`

新增一种序列化格式 `mem_pml`，专门用于存储预计算的嵌入向量、倒排索引或层次聚类结果。它基于 `ppdata` 但增加了索引头部。

在 `memory_backends` 中可指定格式和索引类型。例如后端 `fast_index` 类型为 vector，格式为 mem_pml，索引类型为 hnsw。

```
- id: fast_index
  type: "vector"
  format: "mem_pml"                       # 使用 mem_pml 格式
  index_type: "hnsw"
```


## 4. 记忆事务（Memory Transaction）

### 4.1 动机

多步记忆操作需要原子性：例如“读取→修改→写入”必须整体成功或失败，避免中间状态被其他节点看到。

### 4.2 语法

在 `control` 中增加 `memory_transaction` 标志，并在脚本或复合节点中包裹多个记忆操作。

```
pnodes:
  - id: atomic_transfer
    type: script
    control:
      memory_transaction: true
    script: |
      async function run(input, ctx) {
          let from = await ctx.memory.read("account_a");
          let to = await ctx.memory.read("account_b");
          if (from.balance >= input.amount) {
              from.balance -= input.amount;
              to.balance += input.amount;
              await ctx.memory.write("account_a", from);
              await ctx.memory.write("account_b", to);
          } else {
              throw new Error("Insufficient funds");
          }
          return { success: true };
      }
```

节点示例：id 为 atomic_transfer，类型 script，control 中包含 memory_transaction: true。脚本中实现账户转账：读取 account_a 和 account_b，检查余额后分别扣减和增加，然后写回。如果任何写操作失败，整个事务回滚。

- 事务期间，相关记忆键会被锁定（悲观锁）或采用 MVCC。
- 若任何一步失败，所有写操作自动回滚。

### 4.3 嵌套事务

一个事务中可调用另一个标记为 `memory_transaction: true` 的子工作流，实现嵌套事务（遵循两阶段提交）。


## 5. 记忆可观测性（Memory Observability）

### 5.1 自动指标收集

PML 运行时自动为每个记忆后端收集以下 Prometheus 风格指标：

- `memory_qps`：每秒操作次数，按操作类型（read/write/search）细分。
- `memory_latency_seconds`：延迟分布（直方图）。
- `memory_hit_rate`：缓存命中率（如果后端有缓存层）。
- `memory_uncertainty_avg`：不确定记忆的平均置信度。

在 `config` 中可开启详细指标，例如设置 `memory_metrics` 启用，导出间隔 15 秒，自定义桶边界。

```
config:
  memory_metrics:
    enabled: true
    export_interval: 15
    buckets: [0.001, 0.005, 0.01, 0.05]
```

### 5.2 内置审计日志

对于关键记忆操作（如事务提交、视图刷新），可配置审计日志。

示例：后端 `critical_store` 启用审计，记录 write 和 delete 操作，并指定 webhook 地址。审计记录包含：操作时间、操作类型、键、事务 ID、结果状态。

```
memory_backends:
  - id: "critical_store"
    audit:
      enabled: true
      log_operations: ["write", "delete"]
      webhook: "https://audit.example.com/memory"
```


## 6. 完整示例：情景记忆 + 不确定记忆 + 事务

以下示例（自然语言描述）展示了三个节点：

```
pml:
  version: "1.12"

memory_backends:
  - id: "trip_log"
    type: "episodic"
    retention_days: 7
  - id: "sensor_obs"
    mode: "uncertain"
    default_confidence: 0.5

workflow:
  input: { vehicle_id: string }
  vars:
    current_pos: { type: array }
  nodes: [record_position, conflict_resolution, update_view]

pnodes:
  - id: record_position
    type: memory
    operation: write
    backend: "sensor_obs"
    value: { pos: "{{workflow.vars.current_pos}}" }
    confidence: "{{sensor.confidence}}"
    event_tag: "gps"

  - id: conflict_resolution
    type: script
    control:
      memory_transaction: true
    script: |
      async function run(input, ctx) {
          let readings = await ctx.memory.search("sensor_obs", "pos near current", { confidence_min: 0.7 });
          let fused = average(readings);
          await ctx.memory.write("trip_log", { event: "position_update", fused_pos: fused });
          return { fused };
      }

  - id: update_view
    type: mem_view
    source: "trip_log"
    projection: ["fused_pos", "timestamp"]
    filter: "timestamp > now - 1h"
    refresh_interval: 60
```


- 定义记忆后端：`trip_log` 类型 episodic，保留 7 天；`sensor_obs` 模式 uncertain，默认置信度 0.5。
- 工作流输入 vehicle_id，变量 current_pos。
- 节点 `record_position`：向 `sensor_obs` 写入当前位姿，置信度来自 sensor.confidence，事件标签 gps。
- 节点 `conflict_resolution`：类型 script，启用 memory_transaction。脚本内从 `sensor_obs` 搜索高置信度（>0.7）的读数，融合后写入 `trip_log` 作为位置更新事件。返回融合结果。
- 节点 `update_view`：类型 mem_view，源为 `trip_log`，投影 fused_pos 和 timestamp，过滤最近一小时，刷新间隔 60 秒。

整个流程记录不确定传感器观测，通过事务融合后存入情景记忆，并创建可快速查询的视图。


## 7. PEM：工程记忆应用（MemPML Application）

### 7.1 背景与动机

在 Harness/Loop 工程场景中，Agent 需要在多轮迭代（甚至跨会话）中持续积累对工程的理解：架构约定、历史决策、错误模式、构建命令等。这类知识是**确定性的、结构化的、需要跨团队共享的**，与传感器观测等不确定记忆有本质区别。

**PEM（Project Engineering Memory）不是一个新的 PML 扩展，而是 MemPML 在工程记忆领域的领域应用。** 它利用 MemPML 已有的 `mem_view` + `mem_pml` + `memory_transaction` 机制，为 Agent 工程提供结构化、可查询、自动积累的持久记忆层。

### 7.2 与 Anthropic Claude Code Rules 的对比

Claude Code 提供两层记忆系统——CLAUDE.md（人写静态指令）和 Auto Memory（Claude 自写的非结构化笔记）。PEM 在此基础上有本质性增强：

| 能力维度 | CLAUDE.md (人写) | Auto Memory (Claude自写) | PML PEM (MemPML应用) |
|----------|:---:|:---:|:---:|
| **内容来源** | 人手动编写 | Agent 自动记录 | Agent 自动积累 + 人可编辑 |
| **结构化程度** | Markdown 文本 | 非结构化 Markdown | Schema 驱动的结构化存储 |
| **可查询性** | 全文扫描 | 全文扫描 | `mem_view` 投影/过滤 + `mem_pml` 向量语义检索 |
| **索引加速** | 无 | 无 | hnsw 向量索引 + 倒排索引 |
| **确定性保证** | 无区分 | 无区分 | `mode: deterministic` 明确语义 |
| **原子事务** | 无 | 无 | `memory_transaction` 保证多步写入一致性 |
| **视图投影** | 无 | 无 | `type: mem_view` 按领域投影（决策记录/错误模式/工程约定） |
| **跨团队共享** | ✅ Git 版本控制 | ❌ 仅限于机器本地 (`~/.claude/projects/`) | ✅ `sync: git` + `ppdata` 序列化 |
| **自动积累** | ❌ 需手动更新 | ✅ 自动但非结构化 | ✅ 自动 + 结构化 schema |
| **跨会话持久** | ✅ | ✅ | ✅ |
| **与 SOF 联动** | ❌ | ❌ | ✅ 可查询 `self.cognitive` 决定工程策略 |
| **路径级规则限定** | ✅ `.claude/rules/` | ❌ | 可通过 `mem_view.filter` 实现等价能力 |
| **动态多智能体 Harness** | ✅ Dynamic Workflows | ❌ | ✅ PML Pipeline + `control.loop` |

### 7.3 架构：PEM 在 MemPML 中的层次

```
┌──────────────────────────────────────────────────────────────────┐
│                  PEM 应用层 (MemPML Domain Application)            │
│                                                                   │
│  mem_view: eng_context     mem_view: decisions   mem_view: errors │
│  (工程约定/架构/构建命令)    (决策ID/时间/结果/置信度)  (错误类型/频率/修复)   │
│         │                        │                      │          │
│         ▼                        ▼                      ▼          │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │          mem_pml 格式 · hnsw 索引 · 语义检索                │  │
│  └──────────────────────────────┬──────────────────────────────┘  │
│                                 │                                  │
│                                 ▼                                  │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │     memory_backend: deterministic · sync: git                │  │
│  │     Raw: engineering facts, decisions, error_patterns        │  │
│  └─────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

### 7.4 语法与应用示例

#### 7.4.1 工程记忆后端声明

工程知识是确定性的，同步到 Git 实现跨团队共享：

```yaml
memory_backends:
  - id: "engineering_kb"
    mode: "deterministic"
    format: "mem_pml"
    index_type: "hnsw"
    sync: "git"                         # 通过 git 跨团队共享
    audit:
      enabled: true
      log_operations: ["write", "delete"]
```

#### 7.4.2 工程上下文视图（类似 CLAUDE.md，但结构化可查询）

```yaml
pnodes:
  - id: view_engineering_context
    type: mem_view
    source: "engineering_kb"
    projection:
      - "module"
      - "convention"
      - "build_command"
      - "constraint"
      - "architecture_summary"
    filter: "category == 'context'"
    refresh_interval: 300
    output:
      schema:
        context_snapshot: { type: object }
```

与 CLAUDE.md 的关键区别：CLAUDE.md 是一个文本文件，Agent 只能全文读取；`mem_view` 是按需投影的结构化视图，Agent 可以按模块、按类别精确查询，且语义检索远快于全文扫描。

#### 7.4.3 工程决策记录视图（类似 Auto Memory，但结构化）

```yaml
  - id: view_decisions
    type: mem_view
    source: "engineering_kb"
    projection:
      - "decision_id"
      - "timestamp"
      - "agent_id"
      - "task_summary"
      - "decision_rationale"
      - "outcome"
      - "confidence"
    filter: "category == 'decision' AND confidence > 0.7"
    refresh_interval: 60
    output:
      schema:
        decisions: { type: array }
```

与 Auto Memory 的关键区别：Auto Memory 是非结构化的 Markdown，Claude "自己决定记住什么"；PEM 的决策记录有明确的 schema，且可设置 `auto_accumulate` 策略确保 harness loop 每轮迭代自动记录。

#### 7.4.4 Harness/Loop 工作流：自动查询 + 自动记录

```yaml
workflow:
  input:
    task: string
    agent_id: string

  nodes:
    # Step 1: 查询已有工程记忆和自身能力
    - id: consult_context
      type: memory
      operation: search
      backend: "view_engineering_context"
      query: "{{workflow.input.task}}"
      output:
        relevant_context: "{{result}}"

    # Step 2: 查询历史相关决策
    - id: consult_history
      type: memory
      operation: search
      backend: "view_decisions"
      query: "{{workflow.input.task}}"
      limit: 5
      output:
        prior_decisions: "{{result}}"

    # Step 3: 结合 SOF + 工程记忆 + 历史决策，执行任务
    - id: execute_task
      type: llm
      process:
        prompt:
          system: |
            工程架构: {{consult_context.output.relevant_context}}
            历史相关决策: {{consult_history.output.prior_decisions}}
            你的能力边界: planning_horizon={{self.cognitive.planning_horizon}}
            当前资源: battery={{self.resource.battery}}
            请基于以上信息执行任务，若能力不足则降级策略。

    # Step 4: 自动记录本次决策（原子写入）
    - id: record_decision
      type: memory
      operation: write
      backend: "engineering_kb"
      value:
        category: "decision"
        decision_id: "DEC-{{workflow.vars.decision_counter}}"
        timestamp: "{{workflow.now}}"
        agent_id: "{{workflow.input.agent_id}}"
        task_summary: "{{workflow.input.task}}"
        decision_rationale: "{{execute_task.output.rationale}}"
        outcome: "{{execute_task.output.outcome}}"
        confidence: "{{execute_task.output.confidence}}"
      control:
        memory_transaction: true
      metrics:
        - name: "decision_recorded"
          type: counter
```

### 7.5 Claude Code Rules 迁移到 PEM 的对照

| Claude Code 概念 | PEM 对应 | 迁移收益 |
|------------------|----------|----------|
| `CLAUDE.md` 项目指令 | `mem_view: eng_context` (category=context) | 从全文文本变为结构化可查询视图 |
| `CLAUDE.md` 构建命令 | `engineering_kb` 中 category=context, type=build_command | 可按模块查询、版本化管理 |
| `.claude/rules/*.md` 路径级规则 | `mem_view` + `filter: "path LIKE 'src/api/%'"` | 等价能力，增加语义检索 |
| Auto Memory 自动笔记 | `mem_view: decisions` + 自动写入 | 从非结构化本地笔记变为结构化共享记录 |
| Auto Memory `MEMORY.md` 索引 | `mem_pml` + hnsw 索引 | 从200行文本扫描变为向量语义检索 |
| Dynamic Workflows Harness | PML Pipeline + `control.loop` | 从 JS 脚本变为声明式工作流 |
| `/memory` 命令查看 | `type: memory` search 节点 | 程序化查询，非交互式 |

### 7.6 设计原则总结

1. **PEM 是 MemPML 的领域应用，不是新扩展** — 复用 `mem_view`、`mem_pml`、`memory_transaction` 全部现有机制。
2. **确定性工程知识 vs 非确定性观测** — 工程约定和决策是确定性的（`mode: deterministic`），传感器数据是非确定性的（`mode: uncertain`）。
3. **自动积累，非手动维护** — Harness loop 自动写入决策记录，避免依赖人工更新 CLAUDE.md。
4. **Git 同步实现跨团队共享** — 解决 Claude Auto Memory 的"机器本地"局限性。
5. **SOF 联动实现自适应工程** — Agent 查询自身能力边界后调整工程策略（如低电量时降低规划深度）。


## 核心特性

- **确定性/不确定记忆模式**：区分可靠事实与带置信度的观测。
- **情景记忆**：支持时间线与事件序列存储查询。
- **MemPML 视图**：预计算索引结构，加速模型检索。
- **记忆事务**：原子化多步操作，保证数据一致性。
- **可观测性**：指标与审计日志，便于调优与安全审计。
- **PEM 工程记忆**：MemPML 的工程领域应用，为 Agent Harness/Loop 提供结构化、可查询、跨团队共享的持久工程记忆——对标并超越 Anthropic Claude Code 的 CLAUDE.md + Auto Memory 体系。
