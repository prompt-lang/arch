## PML自身本体框架（Self Ontology Framework, SOF）


> 本版本为基于 PML 构建的 Agent 系统增加 **自身本体框架（Self Ontology Framework, SOF）**。SOF 用于声明 Agent 的物理结构（如 URDF）、认知架构、能力范围、资源限制等自身属性，使得 Agent 在执行工作流时能够自我感知、自我评估，并根据本体信息优化决策。


## 设计目标

- **本体声明**：Agent 可以描述自身组成部分（硬件、软件、传感器、执行器）、几何/运动学约束、认知能力、能耗模型等。
- **自知情决策**：工作流中的节点可查询 SOF，获取自身能力、限制、资源状态，从而避免不可能任务、选择合适策略。
- **与 PML 无缝集成**：SOF 作为 PML 工作流的一部分，可通过数据绑定（`{{self.*}}`）或 API（`ctx.self`）访问。
- **支持标准格式导入**：例如 URDF（机器人描述）、SDF、ONNX 模型元数据等可被转换为 PML 内部的 SOF 结构。
- **动态更新**：Agent 在运行时可以更新 SOF 的部分字段（如剩余电量、磨损程度）。

## 核心概念

| 概念 | 说明 | PML 实现 |
|------|------|----------|
| **本体声明（Self Ontology）** | 描述 Agent 自身结构、能力、约束的元数据 | 顶层 `self_ontology` 字段 |
| **物理本体** | 几何、运动学、动力学、传感器、执行器（如 URDF 等效） | `self_ontology.physical` |
| **认知本体** | 智能体的推理能力、记忆容量、规划层级、允许的模型 | `self_ontology.cognitive` |
| **资源本体** | 当前电量、CPU 使用率、网络带宽、存储空间 | `self_ontology.resource` |
| **技能本体** | Agent 所拥有的 Skills 及其前置条件、效果 | `self_ontology.skills`（可与顶层 `skills` 关联） |
| **本体访问** | 在工作流中查询自身属性 | 数据绑定 `{{self.physical.joint_limits}}`；脚本 API `ctx.self.get()` |

## 语法扩展

### 顶层 `self_ontology` 字段

在 PML 文件的根级别添加 `self_ontology`，描述 Agent 的本体信息。

```
self_ontology:
  version: "1.0"
  physical:
    urdf: "./robot.urdf"                    # 引用外部 URDF 文件
    # 或内联描述
    links:
      - name: "base_link"
        mass: 10.0
        inertia: [1.0, 0.0, 0.0, 1.0, 0.0, 1.0]
    joints:
      - name: "arm_joint"
        type: "revolute"
        limits: { lower: -1.57, upper: 1.57, velocity: 2.0 }
    sensors:
      - name: "camera"
        type: "rgb"
        fov: 1.2
        range: 10.0
    actuators:
      - name: "wheel_left"
        type: "velocity"
        max_speed: 2.0
  cognitive:
    planning_horizon: 10                    # 最大规划步数
    memory_capacity: { fast: 100, long: 1000 }
    allowed_models: ["bert-tiny", "yolov8n"]
    reasoning_depth: 3
  resource:
    battery: 0.85                           # 当前电量百分比
    cpu_load: 0.3
    memory_used_mb: 256
    storage_free_gb: 5.2
  skills:
    - name: "grasp"
      requires: ["arm_joint", "gripper"]
      preconditions: ["object within reach"]
      effects: ["object held"]
```

例如：

- 版本为 "1.0"
- 物理部分：引用外部 URDF 文件 `./robot.urdf`，也可内联描述连杆（如 `base_link` 的质量、惯性）和关节（如 `arm_joint` 的类型、运动范围、最大速度），以及传感器（如 `camera` 的视场角、范围）和执行器（如 `wheel_left` 的最大速度）
- 认知部分：规划步数上限为 10，快速记忆容量 100、长期记忆容量 1000，允许的模型列表包含 `bert-tiny` 和 `yolov8n`，推理深度为 3
- 资源部分：当前电量 85%，CPU 负载 0.3，已用内存 256MB，剩余存储 5.2GB
- 技能部分：例如技能 `grasp`，需要 `arm_joint` 和 `gripper`，前置条件为“物体在可达范围内”，效果为“物体被抓住”

### 数据绑定访问

在工作流中可以使用形如 `{{self.physical.joints.arm_joint.limits.upper}}` 的路径直接读取本体属性。

```
- id: move_arm
  type: actuator
  action: "joint_velocity"
  parameters:
    velocities: "{{plan.output.joints}}"
    # 确保不超过最大速度
    max_velocity: "{{self.physical.joints.arm_joint.limits.velocity}}"
```

节点输入也可绑定自身属性。例如，一个关节速度控制节点，其参数中的 `max_velocity` 可以绑定到本体中对应关节的速度极限。

### 脚本 API 访问

PMLScript 提供 `ctx.self` 对象，支持读取和受控写入。

```
// 读取本体信息
let maxSpeed = ctx.self.get("physical.joints.arm_joint.limits.velocity");
let battery = ctx.self.get("resource.battery");

// 更新资源状态（例如执行器消耗）
await ctx.self.update("resource.battery", battery - 0.05);
```

例如，脚本中可以获取关节的最大速度，读取当前电量；也可以更新资源状态，比如在执行器动作后减少电池电量。

### 3.4 动态条件与约束

`control` 中的 `condition` 可以使用自身属性来判断。例如，一个决策节点检查电池电量是否大于 20%，若是则继续执行任务，否则跳转到充电节点。

```
- id: check_battery
  type: decision
  condition: "{{self.resource.battery > 0.2}}"
  next_nodes: ["continue_task", "charge"]
```

### 3.5 从外部格式导入

PML 编译器支持从 URDF、SDF、ONNX 等文件导入并生成 `self_ontology` 的物理部分。例如命令 `ppcli import urdf robot.urdf --output robot.pml` 会自动填充 `self_ontology.physical` 内容。


## 4. 与现有 PML 特性的集成

- **与 `capability_scope` 集成**：`self_ontology.cognitive.planning_horizon` 可作为 `capability_scope.max_horizon_steps` 的默认值。
- **与 `permissions` 集成**：SOF 的资源更新操作可能需要特定权限（如 `write_self_resource`）。
- **与 `memory` 集成**：SOF 中的 `memory_capacity` 可以指导记忆后端的选择和配置。
- **与 `markup` 集成**：Agent 自身的状态（位置、姿态）可同步到标记域，供环境感知。


## 5. 完整示例：基于 SOF 的自适应机器人

以下示例展示了机器人如何利用 SOF 信息实现低电量模式下的节能规划。

```
pml:
  version: "1.16"

self_ontology:
  physical:
    urdf: "./robot.urdf"
  cognitive:
    planning_horizon: 20
  resource:
    battery: 0.25                     # 电量偏低

workflow:
  input: { goal: string }
  vars: { energy_mode: "normal" }
  nodes: [check_energy, plan_motion, execute]

pnodes:
  - id: check_energy
    type: decision
    condition: "{{self.resource.battery < 0.3}}"
    next_nodes: ["set_low_energy", "plan_motion"]

  - id: set_low_energy
    type: script
    script: |
      ctx.state.set("energy_mode", "low");
      // 动态调整规划器的搜索深度
      ctx.self.update("cognitive.planning_horizon", 5);
      // 通知调度器降低步频
      ctx.setStepFreq("plan_motion", 1.0);   // 原本 10Hz

  - id: plan_motion
    type: planning
    input: { goal: "{{workflow.input.goal}}" }
    output: { path: "{{result.plan}}" }
    control:
      max_horizon: "{{self.cognitive.planning_horizon}}"

  - id: execute
    type: actuator
    action: "joint_velocity"
    parameters:
      velocities: "{{plan_motion.output.path}}"
    # 使用本体中的关节极限
    max_velocity: "{{self.physical.joints.arm_joint.limits.velocity}}"
```

- `self_ontology` 定义：
  - 物理部分引用 `./robot.urdf`
  - 认知部分：`planning_horizon: 20`
  - 资源部分：`battery: 0.25`（电量偏低）
- 工作流输入：目标字符串 `goal`
- 变量：`energy_mode` 初始为 `normal`
- 节点：
  1. `check_energy`：决策节点，条件为当前电量低于 0.3。如果成立则跳转到 `set_low_energy`，否则到 `plan_motion`
  2. `set_low_energy`：脚本节点，将 `energy_mode` 设为 `low`；更新认知部分的 `planning_horizon` 为 5；调用 `ctx.setStepFreq` 将 `plan_motion` 节点的步频降至 1 Hz（原本 10 Hz）
  3. `plan_motion`：规划节点，根据目标生成路径，其最大规划步数绑定到 `self.cognitive.planning_horizon`
  4. `execute`：执行器节点，动作为关节速度，参数中的 `max_velocity` 绑定到本体中关节的速度极限

当电量低时，Agent 自动降低规划深度和执行频率，实现节能。


## 6. 扩展性：SOF 与 Agent 学习

- Agent 可以通过训练调整 SOF 中的某些参数（如认知负荷、能耗模型），实现自我建模的进化。
- 当 Agent 检测到自身性能与 SOF 声明不符时（例如实际耗电快于模型），可以触发 `planning` 节点重新校准 SOF。

## 7. 向后兼容性

- `self_ontology` 字段为可选，不存在的 Agent 仍可运行。
- 未定义 SOF 时，`{{self.*}}` 绑定返回 `null`，不会导致工作流失败（除非显式要求）。
- 提供 `pplci self validate` 命令检查 SOF 结构完整性。


## 核心特性

引入的 **Self Ontology Framework** 使 Agent 能够：

- **自我描述**：物理结构、认知能力、资源状态
- **自我感知**：通过数据绑定或 API 在决策中实时查询自身属性
- **自适应行为**：根据资源变化动态调整策略（如低电量降频）
- **与生态互通**：兼容 URDF 等标准格式，与技能、权限、记忆等模块联动

SOF 是构建具有自我认知、自我监控和自我优化能力的智能体系统的关键基础设施。
