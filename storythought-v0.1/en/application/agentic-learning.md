## Agentic Learning Application


## Core Concepts

| Concept | Description | PML Implementation |
|---------|-------------|-------------------|
| **Learning Script** | PML workflow or script defining learning strategy, including data sampling, loss computation, parameter updates | `type: script` + `learning` flag |
| **Training Task** | A specific learning task containing dataset, model, hyperparameters | `type: training` node |
| **Learning Session** | An instance of training execution, capable of logging, saving checkpoints, early stopping | `workflow` + `config.checkpoint` |
| **Knowledge Distillation** | Large model guiding small model process | `control.loop` + comparison nodes |
| **Experience Replay** | Sampling historical experiences from memory buffer pool for learning | `memory` backend + `query` |
| **Curriculum Learning** | Dynamically adjusting sample difficulty ordering | `planning` node dynamically generating data pipeline |



## Learning Script

A learning script is a special PMLScript describing learning steps. Marked with `learning_script: true` in `scripts`.

The learning script receives `sample` or `env` and `model` (accessible via `ctx.model`), returning loss or reward. It is not limited to deep learning and can support any parameter update logic (e.g., Bayesian inference, rule base correction).

```
scripts:
  - id: "supervised_step"
    language: "python"
    learning_script: true
    code: |
      def learn(sample, model, ctx):
          # Forward pass
          prediction = model.forward(sample["input"])
          loss = compute_loss(prediction, sample["label"])
          # Backward pass
          model.backward(loss)
          model.update(ctx.learning_rate)
          return {"loss": loss, "updated": True}

  - id: "rl_step"
    learning_script: true
    code: |
      def learn(env, policy, ctx):
          state = env.reset()
          done = False
          total_reward = 0
          while not done:
              action = policy.act(state)
              next_state, reward, done = env.step(action)
              policy.learn(state, action, reward, next_state)
              total_reward += reward
              state = next_state
          return {"total_reward": total_reward}
```


## Fast Training Task: `type: training` Node

```
pnodes:
  - id: fast_finetune
    type: training
    model: "bert-base"
    dataset:
      source: "./data/samples.ppdata"
      batch_size: 8
    learning_script: "supervised_step"
    hyperparams:
      learning_rate: 0.001
      epochs: 3
    control:
      loop:
        condition: "{{epoch < epochs}}"
        max_iterations: 10
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

New node type `training` for executing a complete training process.

- `model` field specifies the model; at runtime NNOS loads the model into the execution environment (local process or remote inference service).
- `dataset` can be `ppdata`, `json`, or `markup` query results.
- The learning process is controlled by `control.loop`, executing the learning script once per iteration.
- Training metrics are automatically collected and exportable via `metrics`.

---

## Learning Session

```
workflow:
  input:
    session_id: string
    initial_model: string
  vars:
    best_loss: 1e9
    patience: 3
  nodes: [data_loader, trainer, evaluator, early_stop]
  checkpoint:
    enabled: true
    interval_secs: 60
    storage: "s3://ml-runs/{{session_id}}"

pnodes:
  - id: data_loader
    type: markup_query
    domain: "training_data"
    query: "$..[?(@.split=='train')]"
    output: { samples: "{{result}}" }

  - id: trainer
    type: training
    input: { samples: "{{data_loader.output.samples}}" }
    output: { loss: "{{result.loss}}", model: "{{result.model_artifact}}" }

  - id: evaluator
    type: evaluation
    input: { model: "{{trainer.output.model}}", test_set: "./test.ppdata" }
    output: { accuracy: "{{result.acc}}" }

  - id: early_stop
    type: script
    control:
      condition: "{{evaluator.output.accuracy > 0.95}}"
    script: |
      function run(input, ctx) {
          ctx.terminate_workflow("early_stop", { best_accuracy: input.acc });
      }
```

A learning session is a long-running workflow instance supporting interruption recovery, early stopping, and experiment tracking.

- Session state (model weights, current epoch, optimizer state) is saved via `checkpoint`.
- Multiple sessions (different `session_id`) can run concurrently.



## Experience Replay via Prompt In Loop

```
pnodes:
  - id: store_experience
    type: memory
    operation: write
    backend: "replay_buffer"
    value:
      state: "{{env.state}}"
      action: "{{agent.action}}"
      reward: "{{env.reward}}"
      next_state: "{{env.next_state}}"

  - id: sample_and_learn
    type: training
    input:
      samples: "{{replay_sample.output.batch}}"
    learning_script: "rl_step"
```

- Use `memory` backend to store experience tuples (state, action, reward).
- Priority sampling can be configured (e.g., based on TD-error).
- Learning scripts can mix online interaction with offline replay.


## Knowledge Distillation

```
pnodes:
  - id: teacher_inference
    type: llm
    model: "gpt-4"
    input: { query: "{{workflow.input.question}}" }
    output: { answer: "{{result}}" }

  - id: student_learn
    type: training
    input:
      samples:
        query: "{{workflow.input.question}}"
        target: "{{teacher_inference.output.answer}}"
    learning_script: "distillation_step"
```

- Large model (teacher) generates soft labels.
- Small model (student) learns from teacher's output distribution.
- Loop can iterate to gradually improve student quality.


## Multi-Agent Collaborative Learning

Multiple agents share a memory backend for asynchronous or synchronous experience sharing.

```
agents:
  - id: explorer
    memory_backend: "shared_pool"
  - id: learner
    memory_backend: "shared_pool"
```

## Markup-Driven Curriculum Learning

Use markup Q-values as sample difficulty, dynamically adjust sampling order via `markup_query`.

```
- id: curriculum_sampler
  type: markup_query
  domain: "training_set"
  query: "$..[*]"
  q_sort: { order: "asc" }
  limit: 16
```
