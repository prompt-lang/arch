# PML Environments Extension

# PML环境建模与环境感知扩展

> 对环境（Environment）的支持。通过扩展标记（Markup）系统，实现环境的分层描述（全局环境 / 局部环境）、环境属性声明、Agent 对环境的感知建模（Ego View 已知性等）。本设计使 PML 能够声明式地定义智能体所处的物理或虚拟世界，并与场景标记、虚实变换、多智能体协作深度集成。


## 核心概念

| 概念 | 说明 | PML 实现 |
|------|------|----------|
| **Global Environment** | 所有 Agent 共享的全局环境，如地图、世界状态 | `markup_domain` 标记 `scope: global` |
| **Local Environment** | 单个 Agent 私有的局部环境，如感知视野、临时记忆 | `markup_domain` 标记 `scope: local` |
| **Ego View** | Agent 自身的第一人称感知范围（已知/未知、可见/不可见） | 标记的 `visibility` 属性或 `ego_view` 字段 |
| **环境属性** | 环境的静态/动态特性，如坐标系、时间尺度、物理规则 | `markup_domain` 的 `properties` |
| **环境感知** | Agent 从环境中获取标记信息的过程 | `markup_query` 配合 `visibility` 过滤 |


## 环境分层语法扩展

### 标记域（Markup Domain）的环境扩展

在原有 `markup_domains` 声明中增加 `scope`、`parent`、`properties` 字段。

```
markup_domains:
  - id: "world_map"
    scope: "global"                     # global / local
    type: "spatial"
    coordinate_system: "world"
    properties:
      time_scale: 1.0
      gravity: [0, -9.8, 0]
      static: false
      known: true                       # 全局环境通常完全已知
    parent: null                        # 无父环境

  - id: "agent_1_perception"
    scope: "local"
    parent: "world_map"                 # 局部环境从全局环境继承部分标记
    coordinate_system: "ego_centric"
    properties:
      ego_view:
        known_radius: 50.0              # 已知区域半径
        update_frequency: 30            # Hz
      visibility: "partial"             # 部分已知
    filters:                            # 只保留父环境中满足条件的标记
      - "distance < known_radius"
```

示例中，全局环境 `world_map` 声明了 `scope: global`，并包含坐标系统、重力、时间尺度等属性。局部环境 `agent_1_perception` 以 `world_map` 为父环境，采用 `ego_centric` 坐标系，并通过 `filters` 自动过滤距离小于已知半径的标记。

### 环境的分层继承规则

- **全局环境（global）**：所有 Agent 可读（权限允许），通常包含世界模型、静态地图等。
- **局部环境（local）**：隶属于某个 Agent 或某个任务，可继承全局环境中的标记，但可添加私有标记（如临时障碍物）。
- **父环境**：通过 `parent` 字段指定，子环境的标记查询会自动包含父环境中可见的部分（受 `filters` 限制）。


## Agent 对环境感知的扩展

在 Agent 定义（`agents` 或工作流中的 `type: agent` 节点）中增加 `perception` 字段，声明 Agent 的环境接口。

### Agent 感知配置

```
agents:
  - id: "robot_1"
    entry: "./robot_agent.pml"
    perception:
      environment: "agent_1_perception"    # 引用的局部环境 ID
      ego_view:
        position: "{{self.position}}"       # 从 Agent 状态获取
        orientation: "{{self.orientation}}"
        known_threshold: 0.8                # 置信度阈值
      output:
        - type: "markup"
          domain: "agent_1_perception"
          operation: "update"               # Agent 可将自己的动作结果写入环境

  - id: "human_operator"
    perception:
      environment: "world_map"
      ego_view:
        known: true                         # 人类操作者拥有全局视图
```

Agent 配置示例中，`robot_1` 关联了局部环境 `agent_1_perception`，并定义了自身位置和朝向。`human_operator` 则直接关联全局环境，获得完整视图。

### Agent 节点的感知运行时

当 Agent 工作流执行时，NNOS 会自动将其 `perception` 配置转化为一组内置的 `markup_query` 能力。Agent 内部可通过 `ctx.environment` 对象执行查询、创建标记等操作，例如获取可见车辆标记、获取已知区域标记、向环境写入路径等。

```
// 在 Agent 的 PMLScript 中
async function run(input, ctx) {
    // 获取当前环境中的可见车辆标记
    let visible_vehicles = await ctx.environment.query("$..[?(@.type=='vehicle')]");
    // 获取已知区域的标记
    let known_objects = await ctx.environment.getKnown();
    // 向环境写入新标记（如规划路径）
    await ctx.environment.create({ type: "path", points: [...] });
}
```


## go View 与已知性（Known / Unknown）建模

在标记中添加 `visibility` 属性，表示该标记对于特定 Agent 的已知程度。可通过标记的 `q` 值扩展为置信度。

### 标记的可见性字段

标记定义可包含 `visibility` 列表，其中每个条目指定某个 Agent 对该标记的已知状态和置信度。

```
markup:
  id: "building_1"
  type: "building"
  region: {...}
  visibility:
    - agent_id: "robot_1"
      known: false          # 该 Agent 尚未发现此建筑
      confidence: 0.2
    - agent_id: "human_operator"
      known: true
```

### 自动更新已知性

环境支持根据 Agent 位置自动更新标记的可见性（通过 `markup_stream` 或定时任务）。例如，可以配置一个流节点，基于距离或视线计算每个标记对于特定 Agent 的可见性，并输出更新。

```
pnodes:
  - id: update_visibility
    type: markup_stream
    domain: "world_map"
    source:
      type: "ego_view_updater"
      agent_id: "robot_1"
      update_interval: 0.1
    process:
      function: "compute_visibility"   # 基于距离、视线等计算
    output: { visibility_updates: "{{result}}" }
```


## 环境查询的已知性过滤

在 `markup_query` 节点中增加 `known_filter` 字段，仅返回特定 Agent 已知的标记。查询时指定 `agent_id` 和最低置信度，即可过滤出符合要求的标记。
```
- id: get_known_obstacles
  type: markup_query
  domain: "world_map"
  query: "$..[?(@.type=='obstacle')]"
  known_filter:
    agent_id: "robot_1"
    min_confidence: 0.5
```


## 虚实变换与环境同步

虚实变换（`markup_transform` 节点）可用于在全局环境和局部环境之间转换坐标，以及将虚拟环境中的标记反映到真实世界。例如，将模拟器中的标记经过变换后注入真实环境标记域，供真实 Agent 使用。

```
- id: virtual_to_real
  type: markup_transform
  source_domain: "simulation_env"     # 虚拟环境
  target_domain: "world_map"          # 真实环境
  transform:
    method: "sim2real"
    parameters:
      noise_model: "gaussian"
```

- 支持将模拟器中的标记（如虚拟传感器检测到的物体）经过变换后注入真实环境标记域，供真实 Agent 使用。


## 完整示例：多 Agent 环境感知与导航

```
pml:
  version: "1.8"

markup_domains:
  - id: "global_map"
    scope: "global"
    type: "spatial"
    properties: { static: false }
  - id: "robot_1_view"
    scope: "local"
    parent: "global_map"
    properties:
      ego_view: { known_radius: 30.0 }
    filters: ["distance(self.position, markup.region) < known_radius"]

agents:
  - id: "robot_1"
    entry: "./robot.pml"
    perception:
      environment: "robot_1_view"
      ego_view:
        position: "{{self.position}}"

workflow:
  input: { goal: [10, 20] }
  nodes: [sense, plan, act]

pnodes:
  - id: sense
    type: markup_query
    domain: "robot_1_view"
    query: "$..[?(@.type=='obstacle')]"
    known_filter: { agent_id: "robot_1", min_confidence: 0.6 }
    output: { obstacles: "{{result}}" }

  - id: plan
    type: planning
    process:
      prompt: "避开障碍 {{sense.output.obstacles}} 规划从 {{self.position}} 到 {{workflow.input.goal}} 的路径"
    output: { path: "{{result.path}}" }

  - id: act
    type: neuro_task
    process:
      template: "move_along({{plan.output.path}})"
    # 执行后更新环境：将新位置写入 self.position
    output: { new_position: "{{result.position}}" }
```

以下工作流演示了机器人 Agent 如何通过局部环境感知障碍物、规划路径并更新自身位置。

首先定义标记域：全局地图 `global_map` 和机器人局部视图 `robot_1_view`，后者继承全局地图并通过距离过滤筛选标记。

定义机器人 Agent `robot_1`，配置其感知环境为 `robot_1_view`，并指定自身位置来源。

工作流包含三个节点：
- `sense`：查询局部环境中置信度大于 0.6 的障碍物标记。
- `plan`：基于查询到的障碍物和当前目标规划路径。
- `act`：执行移动，并将新位置写回 Agent 状态。


## 工具链扩展

- **`ppcli env inspect`**：查看环境摘要，列出所有标记域及其属性。
- **`ppcli env snapshot`**：导出环境当前状态为 JSON 或 ppdata。
- **`ppcli env diff`**：比较两个时间点环境的变化。

## 实现注意事项

1. **继承性能**：局部环境查询父环境时需合并结果集，应使用增量更新和缓存，避免每次完整遍历。
2. **可见性更新**：高频更新 Ego View 可能导致大量标记重算，建议采用空间索引和脏区域标记。
3. **权限控制**：全局环境的写入应有严格权限，局部环境仅关联 Agent 可写。
4. **已知性持久化**：Agent 的已知性应可存储（通过 `memory` 后端），以便恢复会话。

