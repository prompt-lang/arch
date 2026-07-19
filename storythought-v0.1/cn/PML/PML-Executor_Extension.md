# PML执行系统扩展

# PML：执行系统扩展（Predictor / Actuator / 统一执行器模型）

> 增加对**四类执行程序**支持：信息程序执行器、估计预测执行器（ME）、确定执行器（DE）、物理动作执行器（AE）。通过新增 `predictor`、`actuator` 节点，并强化 `neuro_task` 为通用的确定性执行器，使 PML 能够声明式地表达从模型推理到物理控制的完整执行语义。

## 核心概念

| 概念 | 说明 | PML 实现 |
|------|------|----------|
| **信息程序执行器 (IE)** | 处理信息流、数据转换、通信 | `type: script`, `type: function`, `type: tool`, `type: mcp`, `type: pipeline`（已有） |
| **估计预测执行器 (ME)** | 神经网络模型推理、预测 | 新增 `type: predictor` 节点 |
| **确定执行器 (DE)** | 确定性算法、符号计算、代码解释 | 强化 `type: neuro_task`，支持 Wasm/Python 沙箱 |
| **物理动作执行器 (AE)** | 控制机器人、电机、传感器等 | 新增 `type: actuator` 节点 |
| **执行器元模型** | 后端能力声明与选择 | 新增顶层 `executors` 字段 |


## 新增节点类型

### 估计预测执行器：`type: predictor`

`predictor` 节点用于加载并执行非 LLM 的预测模型（如 CNN、RNN、MLP、传统机器学习模型）。

```
pnodes:
  - id: obstacle_detection
    type: predictor
    model: "yolov8n.onnx"          # 模型路径
    backend: "onnxruntime"         # 或 tensorflow, torch
    input:
      image: "{{camera.frame}}"
    output:
      schema:
        boxes: { type: array }
        scores: { type: array }
    control:
      timeout: 0.05                 # 实时要求
```

**语法字段**：
- `model`：模型文件路径或标识（支持 ONNX、TensorFlow SavedModel、PyTorch ScriptModule 等）。
- `backend`：推理后端，可选 `onnxruntime`, `tensorflow`, `torch`, `tflite`。
- `input`：输入数据映射，支持批量。
- `output`：输出 schema，与模型输出结构对应。
- `batch_size`：可选，自动分批。
- `control` 中可设置 `timeout`，适用于实时推理。

**示例描述**：
节点 id 为 `obstacle_detection`，类型为 `predictor`，模型文件 `yolov8n.onnx`，后端 `onnxruntime`。输入绑定来自相机帧，输出包含 `boxes` 数组和 `scores` 数组。控制设置超时 0.05 秒以满足实时要求。


### 3.2 物理动作执行器：`type: actuator`

`actuator` 节点用于向物理系统（机器人、电机、显示器等）发送动作指令，并可接收反馈。

```
pnodes:
  - id: move_arm
    type: actuator
    action: "joint_velocity"
    target_system: "robot_arm"      # 引用外部物理系统
    parameters:
      velocities: [0.5, 0.2, 0.0]
    feedback:
      - type: "position"
        output_to: "{{workflow.vars.arm_position}}"
    control:
      duration_sec: 2.0
      blocking: true                # 等待完成
```

**语法字段**：
- `action`：动作名称，如 `joint_velocity`, `gripper`, `move_to`。
- `target_system`：引用外部物理系统标识（通过 `executors` 声明）。
- `parameters`：动作参数（如速度、位置、力度）。
- `feedback`：定义反馈通道，输出到状态变量。
- `control` 中可设置 `duration_sec`（动作执行时间）、`blocking`（是否等待完成）、`repeat` 等。

**示例描述**：
节点 id 为 `move_arm`，类型 `actuator`，动作 `joint_velocity`，目标系统 `robot_arm`，参数 `velocities` 为 [0.5, 0.2, 0.0]。反馈通道包含位置，输出到 `workflow.vars.arm_position`。控制设置持续 2.0 秒，阻塞等待。


### 3.3 强化确定执行器：`type: neuro_task`

原有的 `neuro_task` 节点强化为通用确定执行器（DE），支持多种确定性语言和安全沙箱。

```
pnodes:
  - id: inverse_kinematics
    type: neuro_task
    language: "wasm"
    code: "./ik.wasm"
    input: { target_pos: [1,2,3] }
    output: { joint_angles: "{{result}}" }
```

**语法扩展**：
- 增加 `language` 字段，可选 `wasm`, `python`, `javascript`, `native`。
- 增加 `sandbox` 配置，限制资源（内存、CPU、文件系统）。
- 代码可以是内联的字符串或外部文件引用。

**示例描述**：
节点 id 为 `inverse_kinematics`，类型 `neuro_task`，语言 `wasm`，代码引用外部文件 `./ik.wasm`，输入 `target_pos` 为 [1,2,3]，输出 `joint_angles`。


## 统一执行器元模型：顶层 `executors`

新增顶层字段 `executors`，用于声明可用的执行器后端及其能力。

```
executors:
  - id: "onnx_runtime"
    type: "predictor"
    supported_formats: ["onnx"]
    max_batch_size: 32
  - id: "robot_driver"
    type: "actuator"
    actions: ["joint_velocity", "gripper"]
    feedback_channels: ["position", "force"]
```

**字段结构**：
- `id`：唯一标识。
- `type`：`predictor`, `actuator`, `neuro_task`, `script` 等。
- `supported_formats`：对于 predictor，列出模型格式列表。
- `actions`：对于 actuator，列出支持的动作名称。
- `max_batch_size`、`device` 等可选。

**示例描述**：
声明两个执行器：`onnx_runtime`（类型 predictor，支持 onnx 格式，最大 batch size 32）和 `robot_driver`（类型 actuator，支持 `joint_velocity` 和 `gripper` 动作，反馈通道包含 `position` 和 `force`）。

工作流中可通过 `executor: "onnx_runtime"` 字段选择特定后端。

## 5. 与现有 PML 特性的集成

- **与记忆系统**：`predictor` 的输入可从 `memory` 中获取预计算特征。
- **与标记环境**：`actuator` 执行后可触发 `markup_crud` 更新环境中物体的位置。
- **与训练闭环**：`training` 节点产出的模型工件可直接填入 `predictor` 的 `model` 字段，实现持续学习。
- **与多智能体**：不同 Agent 可配置不同的 `executor` 后端，例如机器人 Agent 使用本地 ONNX 推理，云端 Agent 使用大型 LLM。


## 完整示例

以下工作流演示了使用 predictor 进行障碍物检测，通过 neuro_task 计算避障路径，再通过 actuator 控制机器人移动。整个流程覆盖了 ME → DE → AE 的链式调用。

```yaml
pml:
  version: "1.9"

markup_domains:
  - id: "world_map"
    scope: "global"
    type: "spatial"
    coordinate_system: "world"

executors:
  - id: "onnx_runtime"
    type: "predictor"
    supported_formats: ["onnx"]
    max_batch_size: 1
  - id: "robot_arm_ctl"
    type: "actuator"
    actions: ["joint_velocity", "gripper"]
    feedback_channels: ["position", "force"]

workflow:
  input:
    image: string          # 摄像头图像路径
    goal: [10, 20, 5]     # 目标位置
  vars:
    arm_position: null
  nodes: [detect, plan, move, feedback]
  edges:
    - detect -> plan
    - plan -> move
    - move -> feedback

pnodes:
  - id: detect
    type: predictor
    executor: "onnx_runtime"
    model: "yolov8n.onnx"
    input:
      image: "{{workflow.input.image}}"
    output:
      schema:
        boxes: { type: array }
        scores: { type: array }
    control:
      timeout: 0.05

  - id: plan
    type: neuro_task
    language: "wasm"
    code: "./ik.wasm"
    input:
      obstacles: "{{detect.output.boxes}}"
      current_pos: "{{workflow.vars.arm_position}}"
      target: "{{workflow.input.goal}}"
    output:
      joints: { type: array }

  - id: move
    type: actuator
    executor: "robot_arm_ctl"
    action: "joint_velocity"
    target_system: "robot_arm"
    parameters:
      velocities: "{{plan.output.joints}}"
    feedback:
      - type: "position"
        output_to: "{{workflow.vars.arm_position}}"
    control:
      duration_sec: 2.0
      blocking: true

  - id: feedback
    type: script
    script: |
      async function run(input, ctx) {
        let newPos = ctx.state.get("workflow.vars.arm_position");
        // 更新全局标记域中的机器人位置
        await ctx.markup.update("world_map", "robot_1", {
          region: { type: "point", coordinates: newPos }
        });
        return { success: true };
      }
```

**全局定义**：
- 标记域 `world_map`（全局）。
- 执行器声明：`onnx_runtime`（predictor）和 `robot_arm_ctl`（actuator）。

**工作流节点**：
1. `detect`：类型 predictor，使用 yolov8 模型检测摄像头图像中的障碍物，输出边界框。
2. `plan`：类型 neuro_task（Wasm），基于检测结果和当前位置计算避障轨迹。
3. `move`：类型 actuator，向机器人发送关节速度指令，持续 2 秒，并等待完成。
4. `feedback`：类型 script，读取 actuator 反馈的位置，更新标记域中的机器人标记。


## 工具链支持

- **`ppcli model info`**：查看模型文件元数据（输入输出形状、后端兼容性）。
- **`ppcli actuator list`**：列出当前环境可用的物理动作及参数。
- **`ppcli executor bench`**：测试执行器性能（延迟、吞吐）。


## 实现注意事项

1. **模型沙箱**：`predictor` 的模型执行应处于隔离进程，防止模型崩溃影响主调度器。
2. **物理动作超时**：`actuator` 必须支持超时机制，并在超时后触发 `on_error` 策略。
3. **异步反馈**：对于长时间动作，`blocking: false` 时节点立即返回，但需提供订阅反馈的方式（通过 `events` 或 `markup_stream`）。
4. **执行器热插拔**：`executors` 后端应支持动态加载（插件机制），便于扩展新硬件或新模型格式。
