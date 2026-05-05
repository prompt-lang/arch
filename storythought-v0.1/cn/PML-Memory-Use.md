# PML记忆增强
> 针对记忆使用（Memory-Use）进行深度增强。新增**记忆模式声明**（确定性/不确定）、**情景记忆**（Episodic Memory）支持、**MemPML 高效索引视图**、**记忆事务**以及**记忆可观测性**指标。这些扩展使 PML 的记忆子系统更贴近认知科学模型，显著提升大规模智能体系统的存储利用效率。


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


## 核心特性

- **确定性/不确定记忆模式**：区分可靠事实与带置信度的观测。
- **情景记忆**：支持时间线与事件序列存储查询。
- **MemPML 视图**：预计算索引结构，加速模型检索。
- **记忆事务**：原子化多步操作，保证数据一致性。
- **可观测性**：指标与审计日志，便于调优与安全审计。
