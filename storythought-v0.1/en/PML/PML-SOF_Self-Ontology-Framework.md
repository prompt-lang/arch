# PML Self Ontology Framework (SOF)

> This version adds the **Self Ontology Framework (SOF)** for PML-based Agent systems. SOF is used to declare the Agent's physical structure (e.g., URDF), cognitive architecture, capability scope, resource constraints, and other self-attributes, enabling the Agent to be self-aware during workflow execution, self-evaluate, and optimize decisions based on ontological information.

## Design Goals

- **Ontology Declaration**: Agents can describe their own components (hardware, software, sensors, actuators), geometric/kinematic constraints, cognitive capabilities, energy models, etc.
- **Self-Aware Decision Making**: Workflow nodes can query SOF to obtain self-capabilities, constraints, and resource states, thereby avoiding impossible tasks and selecting appropriate strategies.
- **Seamless PML Integration**: SOF is part of the PML workflow, accessible via data bindings (`{{self.*}}`) or API (`ctx.self`).
- **Standard Format Import Support**: For example, URDF (robot description), SDF, ONNX model metadata can be converted to PML-internal SOF structures.
- **Dynamic Updates**: Agents can update portions of SOF at runtime (e.g., remaining battery, wear level).

## Core Concepts

| Concept | Description | PML Implementation |
|---------|-------------|-------------------|
| **Self Ontology** | Metadata describing the Agent's own structure, capabilities, and constraints | Top-level `self_ontology` field |
| **Physical Ontology** | Geometry, kinematics, dynamics, sensors, actuators (URDF equivalent) | `self_ontology.physical` |
| **Cognitive Ontology** | Agent reasoning capability, memory capacity, planning tiers, allowed models | `self_ontology.cognitive` |
| **Resource Ontology** | Current battery, CPU usage, network bandwidth, storage | `self_ontology.resource` |
| **Skills Ontology** | Skills the Agent possesses with their preconditions and effects | `self_ontology.skills` (can relate to top-level `skills`) |
| **Ontology Access** | Query self-attributes within workflows | Data binding `{{self.physical.joint_limits}}`; script API `ctx.self.get()` |

## Syntax Extension

### Top-Level `self_ontology` Field

Add `self_ontology` at the PML file root level to describe the Agent's ontological information.

```
self_ontology:
  version: "1.0"
  physical:
    urdf: "./robot.urdf"                    # Reference external URDF file
    # or inline description
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
    planning_horizon: 10
    memory_capacity: { fast: 100, long: 1000 }
    allowed_models: ["bert-tiny", "yolov8n"]
    reasoning_depth: 3
  resource:
    battery: 0.85
    cpu_load: 0.3
    memory_used_mb: 256
    storage_free_gb: 5.2
  skills:
    - name: "grasp"
      requires: ["arm_joint", "gripper"]
      preconditions: ["object within reach"]
      effects: ["object held"]
```

### Data Binding Access

In workflows, self-attributes can be read directly using paths like `{{self.physical.joints.arm_joint.limits.upper}}`.

```
- id: move_arm
  type: actuator
  action: "joint_velocity"
  parameters:
    velocities: "{{plan.output.joints}}"
    # Ensure not exceeding max velocity
    max_velocity: "{{self.physical.joints.arm_joint.limits.velocity}}"
```

### Script API Access

PMLScript provides the `ctx.self` object, supporting reads and controlled writes.

```
// Read ontology information
let maxSpeed = ctx.self.get("physical.joints.arm_joint.limits.velocity");
let battery = ctx.self.get("resource.battery");

// Update resource state (e.g., actuator consumption)
await ctx.self.update("resource.battery", battery - 0.05);
```

### Dynamic Conditions & Constraints

`control` `condition` can use self-attributes for judgment. For example, a decision node checks if battery is above 20%:

```
- id: check_battery
  type: decision
  condition: "{{self.resource.battery > 0.2}}"
  next_nodes: ["continue_task", "charge"]
```

### Importing from External Formats

PML compiler supports importing from URDF, SDF, ONNX files and generating `self_ontology` physical portions. For example, command `ppcli import urdf robot.urdf --output robot.pml` auto-populates `self_ontology.physical` content.

## Integration with Existing PML Features

- **With `capability_scope`**: `self_ontology.cognitive.planning_horizon` can serve as the default value for `capability_scope.max_horizon_steps`.
- **With `permissions`**: SOF resource update operations may require specific permissions (e.g., `write_self_resource`).
- **With `memory`**: SOF `memory_capacity` can guide memory backend selection and configuration.
- **With `markup`**: The Agent's own state (position, pose) can sync to markup domains for environmental awareness.

## Complete Example: SOF-Based Adaptive Robot

The following example demonstrates how a robot uses SOF information to implement energy-saving planning in low-battery mode.

```
pml:
  version: "1.16"

self_ontology:
  physical:
    urdf: "./robot.urdf"
  cognitive:
    planning_horizon: 20
  resource:
    battery: 0.25                     # Battery is low

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
      // Dynamically adjust planner search depth
      ctx.self.update("cognitive.planning_horizon", 5);
      // Notify scheduler to lower step frequency
      ctx.setStepFreq("plan_motion", 1.0);   // originally 10Hz

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
    # Use joint limits from ontology
    max_velocity: "{{self.physical.joints.arm_joint.limits.velocity}}"
```

When battery is low, the Agent automatically reduces planning depth and execution frequency to save energy.
