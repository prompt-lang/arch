# PML Agentic Learning Extension

> Based on PML, adds first-class support for Agentic Learning. Through declarative learning scripts, fast training tasks, and learning session management, PML workflows can drive model self-evolution, knowledge distillation, experience replay, and multi-agent collaborative learning.

## Core Concepts

| Concept | Description | PML Implementation |
|---------|-------------|-------------------|
| **Learning Script** | Defines single-step learning logic (forward, loss, backward) in PMLScript | `type: script` + `learning_script: true` |
| **Training Task** | A complete training process containing model, data, hyperparameters, learning script | `type: training` node |
| **Learning Session** | One execution instance of a training task, supporting checkpoint resume and early stopping | `workflow` + `config.checkpoint` |
| **Experience Replay** | Sampling historical experiences from memory cache for training | `memory` backend + `type: memory` node |
| **Curriculum Learning** | Dynamically adjusting training sample difficulty or ordering | `planning` node generating data pipeline |
| **Knowledge Distillation** | Large model (teacher) guiding small model (student) for transfer learning | `control.loop` + comparative evaluation nodes |

## New Node: `type: training`

The training node encapsulates a complete training task, supporting model loading, data flow, learning loop, and metric collection.

### Syntax

```
pnodes:
  - id: my_trainer
    type: training
    model: "bert-base"
    dataset:
      source: "./data/train.ppdata"
      batch_size: 8
      shuffle: true
    learning_script: "supervised_step"
    hyperparams:
      learning_rate: 0.001
      epochs: 3
    optimizer: "adam"
    control:
      loop:
        condition: "{{epoch < epochs}}"
        max_iterations: 100
        update_vars:
          - epoch: "{{epoch + 1}}"
    metrics:
      - name: "train_loss"
        type: scalar
    output:
      schema:
        final_loss: { type: number }
        model_artifact: { type: string }
```

### Learning Script

Learning scripts are special PMLScripts defining single-step training logic. Marked with `learning_script: true` in scripts.

```
scripts:
  - id: supervised_step
    language: "python"
    learning_script: true
    code: |
      def learn(sample, model, ctx):
          output = model.forward(sample["input"])
          loss = cross_entropy(output, sample["label"])
          model.backward(loss)
          model.update(ctx.learning_rate)
          return {"loss": loss}
```

### Learning Session

Learning sessions define long-running training workflows supporting checkpoint resume, early stopping, and experiment tracking.

```
workflow:
  input:
    session_id: string
  checkpoint:
    enabled: true
    interval_secs: 60
    storage: "s3://ml-runs/{{session_id}}"
```

## Integration with Existing Features

### Memory & Experience Replay

Leverage memory backends to store historical experiences (state, action, reward), sample using `type: memory` nodes.

```
pnodes:
  - id: sample_replay
    type: memory
    operation: search
    backend: "replay_buffer"
    limit: 32
    output: { batch: "{{result}}" }

  - id: train_on_replay
    type: training
    input: { samples: "{{sample_replay.output.batch}}" }
```

### Synthetic Prompt Data Generation

`type: synthetic` nodes can generate augmented samples for data augmentation or curriculum learning.

### Multi-Agent Collaborative Learning

Multiple agents can share a memory backend for asynchronous or synchronous experience sharing.

### Markup-Driven Curriculum Learning

Use markup Q-values as sample difficulty, dynamically adjust sampling order via `markup_query`.

## Complete Example: Few-Shot Classification Fast Training

```
pml:
  version: "1.7"

meta:
  name: "few_shot_learner"
  taskgraph: "static"

workflow:
  input: { support_set: array, query: object }
  nodes: [train, predict]
  edges:
    - train -> predict

pnodes:
  - id: train
    type: training
    model: "proto_net"
    dataset:
      source: "{{workflow.input.support_set}}"
      batch_size: 4
    learning_script: "prototypical_step"
    hyperparams:
      learning_rate: 0.001
      epochs: 10
    control:
      loop:
        condition: "{{epoch < epochs}}"
        max_iterations: 50
        update_vars:
          - epoch: "{{epoch + 1}}"
    output:
      model_artifact: "{{result.model_artifact}}"

  - id: predict
    type: predictor
    model: "{{train.output.model_artifact}}"
    input: { sample: "{{workflow.input.query}}" }
    output:
      schema:
        class: { type: string }
        confidence: { type: number }
```
