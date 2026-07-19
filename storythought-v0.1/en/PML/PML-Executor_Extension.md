# PML Execution System Extension

# PML: Execution System Extension (Predictor / Actuator / Unified Executor Model)

> Adds support for **four types of execution programs**: Information Program Executor, Model Estimation Predictor (ME), Deterministic Executor (DE), Physical Action Executor (AE). By adding `predictor` and `actuator` nodes, and strengthening `neuro_task` as a general deterministic executor, PML can declaratively express complete execution semantics from model inference to physical control.

## Core Concepts

| Concept | Description | PML Implementation |
|---------|-------------|-------------------|
| **Information Program Executor (IE)** | Processes information flow, data transformation, communication | `type: script`, `type: function`, `type: tool`, `type: mcp`, `type: pipeline` (existing) |
| **Model Estimator (ME)** | Neural network model inference, prediction | New `type: predictor` node |
| **Deterministic Executor (DE)** | Deterministic algorithms, symbolic computation, code interpretation | Strengthened `type: neuro_task`, supports Wasm/Python sandbox |
| **Physical Action Executor (AE)** | Controls robots, motors, sensors, etc. | New `type: actuator` node |
| **Executor Meta-Model** | Backend capability declaration and selection | New top-level `executors` field |

## New Node Types

### Model Estimator: `type: predictor`

The `predictor` node loads and executes non-LLM prediction models (e.g., CNN, RNN, MLP, traditional ML models).

```
pnodes:
  - id: obstacle_detection
    type: predictor
    model: "yolov8n.onnx"
    backend: "onnxruntime"
    input:
      image: "{{camera.frame}}"
    output:
      schema:
        boxes: { type: array }
        scores: { type: array }
    control:
      timeout: 0.05
```

**Syntax fields**:
- `model`: Model file path or identifier (supports ONNX, TensorFlow SavedModel, PyTorch ScriptModule, etc.)
- `backend`: Inference backend, options: `onnxruntime`, `tensorflow`, `torch`, `tflite`
- `input`: Input data mapping, supports batching
- `output`: Output schema matching model output structure
- `batch_size`: Optional, auto-batching
- `control` can set `timeout` for real-time inference

### Physical Action Executor: `type: actuator`

The `actuator` node sends action commands to physical systems (robots, motors, displays, etc.) and can receive feedback.

```
pnodes:
  - id: move_arm
    type: actuator
    action: "joint_velocity"
    target_system: "robot_arm"
    parameters:
      velocities: [0.5, 0.2, 0.0]
    feedback:
      - type: "position"
        output_to: "{{workflow.vars.arm_position}}"
    control:
      duration_sec: 2.0
      blocking: true
```

**Syntax fields**:
- `action`: Action name, e.g., `joint_velocity`, `gripper`, `move_to`
- `target_system`: Reference to external physical system identifier (declared via `executors`)
- `parameters`: Action parameters (e.g., velocity, position, force)
- `feedback`: Define feedback channels, output to state variables
- `control` can set `duration_sec`, `blocking`, `repeat`, etc.

### Strengthened Deterministic Executor: `type: neuro_task`

The original `neuro_task` node is strengthened as a general deterministic executor (DE), supporting multiple deterministic languages and secure sandboxes.

```
pnodes:
  - id: inverse_kinematics
    type: neuro_task
    language: "wasm"
    code: "./ik.wasm"
    input: { target_pos: [1,2,3] }
    output: { joint_angles: "{{result}}" }
```

**Syntax extension**:
- Added `language` field, options: `wasm`, `python`, `javascript`, `native`
- Added `sandbox` configuration to restrict resources (memory, CPU, filesystem)
- Code can be inline strings or external file references

## Unified Executor Meta-Model: Top-Level `executors`

New top-level field `executors` for declaring available executor backends and their capabilities.

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

## Integration with Existing PML Features

- **With Memory System**: `predictor` inputs can retrieve pre-computed features from `memory`
- **With Markup Environment**: `actuator` execution can trigger `markup_crud` to update object positions in the environment
- **With Training Loop**: Model artifacts produced by `training` nodes can be directly fed into `predictor`'s `model` field for continuous learning
- **With Multi-Agent**: Different Agents can configure different `executor` backends, e.g., robot Agent uses local ONNX inference, cloud Agent uses large LLM

## Complete Example

The following workflow demonstrates obstacle detection using predictor, path planning via neuro_task, and robot movement control via actuator. The entire flow covers ME → DE → AE chain calls.

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
    image: string
    goal: [10, 20, 5]
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
      schema:
        joint_angles: { type: array }

  - id: move
    type: actuator
    executor: "robot_arm_ctl"
    action: "joint_velocity"
    parameters:
      velocities: "{{plan.output.joint_angles}}"
    control:
      duration_sec: 2.0
      blocking: true

  - id: feedback
    type: script
    script: |
      async function run(input, ctx) {
          let pos = await ctx.markup.read("world_map", "robot_arm");
          ctx.state.set("workflow.vars.arm_position", pos);
          return { updated: true };
      }
```
