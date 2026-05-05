# PML智能体分层、记忆分层、元模式与抽象层级扩展


> 增加对**智能体规划能力分层**（Local Planner / SuperIntelligent Planner / MA-SuperIntelligent Planner）、**记忆分层**（Fast-Memory / Long-time-Memory）、**元模式**（内省与自修改）以及**抽象层级**（信息简化、概念建模）的支持。这些扩展使 PML 能够更精确地建模复杂智能体系统的认知架构。

---

## 智能体分层：Local Planner / SuperIntelligent Planner / MA-SUPER

### 能力范围声明（ThoughtBench / ICS）

```
agents:
  - id: "robot_local"
    capability_scope:
      level: "local"                     # local, global, super
      max_horizon_steps: 10
      allowed_domains: ["motion", "sensing"]
      thought_bench: "Tier1"             # 自定义评级
    entry: "./local_planner.pml"

  - id: "cloud_super"
    capability_scope:
      level: "global"
      max_horizon_steps: 10000
      allowed_domains: ["all"]
      thought_bench: "Tier3"
    entry: "./super_planner.pml"
```

在 `agents` 定义中增加 `capability_scope` 字段，描述智能体的规划层级、最大规划步数、允许的操作域等。

示例定义了两个智能体：

- 智能体 `robot_local` 的 `capability_scope` 包含：`level` 为 `local`，`max_horizon_steps` 为 10，`allowed_domains` 为 `["motion", "sensing"]`，`thought_bench` 为 `"Tier1"`。其 `entry` 指向本地规划器文件。
- 智能体 `cloud_super` 的 `capability_scope` 包含：`level` 为 `global`，`max_horizon_steps` 为 10000，`allowed_domains` 为 `["all"]`，`thought_bench` 为 `"Tier3"`。其 `entry` 指向超级规划器文件。

- **Local Planner** (`level: local`)：适合实时、短视距、资源受限的规划（如机器人避障）。
- **SuperIntelligent Planner (SIP)** (`level: global` 或 `super`)：适合长周期、跨域、计算密集的规划（如供应链优化）。
- **MA-SuperIntelligent Planner**：多个 `level: global` 智能体通过协作工作流（如 `type: foreach` + `agent` 节点）形成的联合系统，可视为隐式组合。

### 在工作流中请求特定规划能力

```
pnodes:
  - id: strategic_decompose
    type: planning
    required_capability: "global"        # 只能由全局规划器执行
    agent: "cloud_super"                 # 可选，直接指定智能体实例
```

`type: planning` 节点可通过 `required_capability` 字段指定所需的智能体层级。例如：

- 节点 id 为 `strategic_decompose`，类型为 `planning`，`required_capability` 为 `"global"`，`agent` 可选指定为 `"cloud_super"`。

若未指定 `agent`，NNOS 会根据 `required_capability` 自动选择一个可用的智能体。

---

## 记忆分层：Fast-Memory / Long-time-Memory

### 记忆后端分层配置

```
config:
  memory:
    backends:
      - id: "fast_cache"
        type: "key_value"
        tier: "fast"                     # fast, long, persistent
        ttl_sec: 300
        max_size_mb: 100
      - id: "long_term"
        type: "vector"
        tier: "long"
        ttl_sec: 2592000                # 30天
        endpoint: "http://qdrant:6333"
```

在 `config.memory` 或 `memory_backends` 中增加 `tier` 字段，区分快速短期记忆与长期持久化记忆。

示例配置了两个后端：

- 后端 id 为 `fast_cache`，类型 `key_value`，`tier` 为 `fast`，`ttl_sec` 为 300，`max_size_mb` 为 100。
- 后端 id 为 `long_term`，类型 `vector`，`tier` 为 `long`，`ttl_sec` 为 2592000（30天），`endpoint` 指向 Qdrant 服务。

### 工作流变量显式分层

`workflow.vars` 中的变量可指定 `memory_tier`，运行时自动映射到对应后端。

```
workflow:
  vars:
    current_goal: { type: string, memory_tier: "fast" }
    user_profile: { type: object, memory_tier: "long", ttl_sec: 86400 }
```

示例：
- 变量 `current_goal` 类型 `string`，`memory_tier` 为 `fast`。
- 变量 `user_profile` 类型 `object`，`memory_tier` 为 `long`，`ttl_sec` 为 86400。

- `fast` 层变量存储在内存缓存中，快速读写，过期自动清理。
- `long` 层变量持久化到磁盘或远程存储，跨会话保留。

### 节点级记忆操作的分层选择

`type: memory` 节点可通过 `tier` 字段直接操作特定分层，无需指定后端 ID。

```
pnodes:
  - id: recall_preferences
    type: memory
    operation: read
    tier: "long"
    key: "user_pref"
```

例如：
- 节点 id 为 `recall_preferences`，类型 `memory`，操作 `read`，`tier` 为 `long`，`key` 为 `user_pref`。

---

## 元模式（Meta-Mode）

### 内省与自修改

```
workflow:
  meta_mode: "reflective"                # normal (默认) 或 reflective
```

在 `workflow` 顶层增加 `meta_mode` 字段，启用工作流的元编程能力。

- `meta_mode` 可选值为 `normal`（默认）或 `reflective`。

- `normal`：工作流结构不可变。
- `reflective`：允许工作流中的脚本通过 `ctx.workflow` API 读取当前图结构（节点、边），并可动态添加/删除节点或边。

脚本示例（仅概念，实际调用受权限限制）：

```
// 在 reflective 模式下
let nodes = await ctx.workflow.getNodes();
if (someCondition) {
    await ctx.workflow.addNode({ id: "new_task", type: "llm", ... });
    await ctx.workflow.addEdge("existing", "new_task");
}
```

在 `reflective` 模式下，脚本可以获取节点列表，若满足条件，则添加新节点和新边。

### 受控元操作权限

元操作受 `permissions` 控制：

```
permissions:
  meta_operations: ["read_graph", "modify_graph"]   # 需要显式授权
```

- 在 `permissions` 中声明 `meta_operations` 列表，例如 `["read_graph", "modify_graph"]`。

未授权的 `reflective` 工作流只能读取图结构，不能修改。

---

## 抽象层级：信息简化与概念建模

### 信息简化（Reduction）

通过 `type: markup_transform` 节点的新操作 `reduce`，将高维标记（如点云、密集区域）简化为低维抽象表示。

```
pnodes:
  - id: simplify_pointcloud
    type: markup_transform
    operation: "reduce"
    method: "voxel_grid"                 # 体素降采样
    source_domain: "lidar_points"
    target_domain: "obstacle_primitives"
    parameters:
      voxel_size: 0.1
```

示例节点：
- id 为 `simplify_pointcloud`，类型 `markup_transform`，操作 `reduce`，方法 `voxel_grid`，源域 `lidar_points`，目标域 `obstacle_primitives`，参数 `voxel_size` 为 0.1。

支持的方法：`voxel_grid`、`cluster`、`bounding_box`、`skeleton`。

### 概念建模（Conceptual Markup）

新增 `type: conceptual` 的标记域，用于存储逻辑概念，不绑定具体几何位置，可基于规则或查询动态推导。

```
markup_domains:
  - id: "safety_concepts"
    type: "conceptual"
    concepts:
      - name: "danger_zone"
        definition: "any markup with type='obstacle' and q > 0.9"
      - name: "high_value_target"
        definition: "markup with attribute.priority > 8"
```

示例定义了一个概念域 `safety_concepts`，类型 `conceptual`，包含两个概念：
- `danger_zone`，定义为：任何类型为 `obstacle` 且 Q 值大于 0.9 的标记。
- `high_value_target`，定义为：标记中属性 `priority` 大于 8。

概念标记可通过 `type: markup_query` 查询，结果将返回满足定义的所有标记（来自其他域）。例如节点 `get_danger_zones` 查询概念域中名称为 `danger_zone` 的概念。

```
pnodes:
  - id: get_danger_zones
    type: markup_query
    domain: "safety_concepts"
    query: "$..[?(@.name=='danger_zone')]"
```

概念域也可以作为其它节点的输入，支持高级决策。

---

## 完整示例：多层级规划与记忆系统

以下工作流展示了一个具备分层智能体和记忆的自主导航系统。

```
pml:
  version: "1.11"

agents:
  - id: "local_nav"
    capability_scope: { level: "local", max_horizon_steps: 5 }
    entry: "./local_planner.pml"
  - id: "global_router"
    capability_scope: { level: "global", max_horizon_steps: 100 }
    entry: "./global_planner.pml"

markup_domains:
  - id: "concepts"
    type: "conceptual"
    concepts:
      - name: "unexplored_area"
        definition: "markup where visited == false"

config:
  memory:
    backends:
      - id: "fast_ram"
        tier: "fast"
        ttl_sec: 60
      - id: "long_disk"
        tier: "long"

workflow:
  input: { start: [0,0], goal: [100,100] }
  vars:
    current_pos: { type: array, memory_tier: "fast" }
    learned_map: { type: object, memory_tier: "long" }
  meta_mode: "reflective"
  nodes: [global_plan, local_execute, update_memory]

pnodes:
  - id: global_plan
    type: planning
    required_capability: "global"
    input: { goal: "{{workflow.input.goal}}" }
    output: { wps: "{{result.waypoints}}" }

  - id: local_execute
    type: planning
    required_capability: "local"
    input: { waypoints: "{{global_plan.output.wps}}", current: "{{workflow.vars.current_pos}}" }
    output: { new_pos: "{{result.position}}" }

  - id: update_memory
    type: memory
    operation: write
    tier: "fast"
    key: "current_pos"
    value: "{{local_execute.output.new_pos}}"
```

定义两个智能体：
- `local_nav`：`capability_scope` 包含 `level: local`，`max_horizon_steps: 5`，`entry` 指向 `local_planner.pml`。
- `global_router`：`capability_scope` 包含 `level: global`，`max_horizon_steps: 100`，`entry` 指向 `global_planner.pml`。

标记域 `concepts`：类型 `conceptual`，概念 `unexplored_area` 定义为标记中 `visited == false`。

记忆配置：两个后端 `fast_ram`（`tier fast`，ttl 60秒）和 `long_disk`（`tier long`）。

工作流输入：`start [0,0]`，`goal [100,100]`。
变量：
- `current_pos`：类型 `array`，`memory_tier fast`。
- `learned_map`：类型 `object`，`memory_tier long`。
`meta_mode` 为 `reflective`。

节点：
- `global_plan`：类型 `planning`，`required_capability global`，输入 `goal`，输出 `waypoints`。
- `local_execute`：类型 `planning`，`required_capability local`，输入 `waypoints` 和 `current_pos`，输出 `new_pos`。
- `update_memory`：类型 `memory`，操作 `write`，`tier fast`，`key current_pos`，值来自 `local_execute` 输出的 `new_pos`。

执行流程：全局规划器生成远距离航点，局部规划器执行短距离运动，快速记忆存储当前位姿，长期记忆保存学习到的地图。元模式启用后，可在运行时动态加入“重新规划”节点。


## 特性汇总

通过以下扩展，增强对复杂智能体系统的建模能力：

- **智能体分层**：通过 `capability_scope` 区分局域/超级/联合超级规划器，与 ThoughtBench/ICS 对齐。
- **记忆分层**：明确 `fast` 与 `long` 层级，优化读写性能和持久化。
- **元模式**：启用工作流自省和动态结构修改，支持元认知。
- **抽象层级**：`reduce` 操作简化感知数据，`conceptual` 域支持高阶逻辑概念。
