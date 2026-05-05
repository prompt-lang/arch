## Agentic Learning Application


## 核心概念

| 概念 | 说明 | PML 实现 |
|------|------|----------|
| **Learning Script** | 定义学习策略的 PML 工作流或脚本，包含数据采样、损失计算、参数更新等步骤 | `type: script` + `learning` 标记 |
| **Training Task** | 一个具体的学习任务，包含数据集、模型、超参数等 | `type: training` 节点 |
| **Learning Session** | 一次训练执行的实例，可记录日志、保存检查点、早停 | `workflow` + `config.checkpoint` |
| **Knowledge Distillation** | 大模型指导小模型的过程 | `control.loop` + 比较节点 |
| **Experience Replay** | 从记忆缓冲池中采样历史经验进行学习 | `memory` 后端 + `query` |
| **Curriculum Learning** | 动态调整样本难度顺序 | `planning` 节点动态生成数据管道 |



##  学习脚本（Learning Script）

学习脚本是一个特殊的 PMLScript，用于描述学习步骤。在 `scripts` 中通过 `learning_script: true` 标记。

学习脚本接收 `sample` 或 `env` 以及 `model`（可通过 `ctx.model` 访问），返回损失或奖励。它不限于深度学习，可支持任何参数更新逻辑（如贝叶斯推理、规则库修正）。

```
scripts:
  - id: "supervised_step"
    language: "python"
    learning_script: true
    code: |
      def learn(sample, model, ctx):
          # 前向传播
          prediction = model.forward(sample["input"])
          loss = compute_loss(prediction, sample["label"])
          # 反向传播
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


##  快速训练任务：`type: training` 节点

```
pnodes:
  - id: fast_finetune
    type: training
    model: "bert-base"                      # 模型标识（本地或远程）
    dataset:
      source: "./data/samples.ppdata"       # 训练数据
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
        model_artifact: { type: string }    # 存储路径
```

新增节点类型 `training`，用于执行一个完整的训练过程。

- `model` 字段指定模型，运行时 NNOS 会加载模型到执行环境（本地进程或远程推理服务）。
- `dataset` 可以是 `ppdata`、`json` 或 `markup` 查询结果。
- 学习过程受 `control.loop` 控制，每次迭代执行一次学习脚本。
- 训练指标自动收集并可通过 `metrics` 导出。

---

## 学习会话（Learning Session）

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
    # 训练配置...
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

学习会话是一个长期运行的工作流实例，支持中断恢复、早停和实验跟踪。

- 通过 `checkpoint` 保存会话状态（模型权重、当前 epoch、优化器状态）。
- 可并发运行多个会话（不同 `session_id`）。



## Prompt In Loop 实现经验回放（Experience Replay）

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
    metadata: { q: 0.5 }                   # 可设置优先级

  - id: sample_replay
    type: memory
    operation: search
    backend: "replay_buffer"
    query: ""                               # 随机采样或按优先级
    limit: 32
    output: { batch: "{{result}}" }

  - id: train_on_replay
    type: training
    input: { samples: "{{sample_replay.output.batch}}" }
```

利用记忆后端存储历史经验，在训练循环中随机采样。


## 分布式学习与多智能体协作

```
agents:
  - id: explorer
    entry: "./explorer.pml"
  - id: learner
    entry: "./learner.pml"
    memory_backend: "shared_experience"

workflow:
  nodes: [run_explorer, run_learner]
  edges:
    - run_explorer -> run_learner          # 探索完再训练，或并行
```

多个智能体可共同学习：一个智能体探索环境生成数据，另一个智能体利用这些数据进行训练。

`explorer` 将其经验写入 `shared_experience` 记忆，`learner` 从中采样并更新模型。两者可通过 `protocols` 实时同步。



## 完整示例：小样本分类快速训练

该工作流执行：从支持集构建 prompt → 合成变体查询 → 循环 5 轮微调（训练 → 评估）→ 最终推理。整个过程不包含外部 Python 训练循环，全由 PML 声明式描述。

```
pml:
  version: "1.6"

meta:
  name: "few_shot_classifier"

config:
  memory:
    backends:
      - id: "experience_pool"
        type: "key_value"

workflow:
  input: { support_set: array, query: string }
  vars: { iteration: 0 }
  nodes: [build_prompt, synthetic_query, train_step, test]

pnodes:
  - id: build_prompt
    type: script
    script: |
      function run(input, ctx) {
          let prompt = "分类任务：\n";
          for (let ex of input.support_set) {
              prompt += `输入: ${ex.text} 类别: ${ex.label}\n`;
          }
          return { prompt };
      }

  - id: synthetic_query
    type: synthetic
    method: "augment"
    sources:
      - source: "{{input.query}}"
    augmentation:
      - type: "paraphrase"
        count: 5
    output: { queries: "{{result}}" }

  - id: train_step
    type: training
    model: "local:bert-tiny"
    dataset: { source: "{{input.support_set}}" }
    learning_script: "supervised_step"
    hyperparams: { learning_rate: 0.01, epochs: 1 }
    control:
      loop:
        condition: "{{workflow.vars.iteration < 5}}"
        update_vars:
          - iteration: "{{workflow.vars.iteration + 1}}"
    output: { model_artifact: "{{result.model_artifact}}" }

  - id: test
    type: llm
    process:
      prompt: "{{build_prompt.output.prompt}}\n问题: {{input.query}}"
    output: { category: "{{result.content}}" }
```


## 工具链支持

- **`ppcli learn`**：执行学习工作流的便捷命令，可传入会话 ID。
- **`ppcli model export`**：导出训练后的模型 artifact（如 ONNX 文件或 PML 内部的 ippl 表示）。
- **`ppcli model serve`**：将训练好的模型作为 MCP 工具暴露，供其他工作流调用。

##  实现注意事项

1. **模型运行时**：NNOS 需要集成轻量级推理引擎（如 ONNX Runtime、llama.cpp），训练脚本应能在沙箱中执行模型更新操作。
2. **分布式训练**：对于大规模学习，训练节点可分发到外部计算集群，PML 通过 `type: tool` 调用训练服务。
3. **学习脚本安全性**：学习脚本可能运行不可信代码，必须严格限制其资源（CPU、内存、网络）并审计。
4. **模型版本管理**：每次训练产生的模型 artifact 应带版本号，并可回滚。





Reference

[deep learning](https://www.deeplearningbook.org/contents/ml.html)

[A Systematic Survey of Prompting Methods](https://arxiv.org/abs/2107.13586)   

[prompt engineering](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview)   

[AutoPDL](https://arxiv.org/abs/2504.04365)    

[pMoE](https://openreview.net/forum?id=Z0eiiV3Yyh)  

[AgentRL](https://arxiv.org/abs/2510.04206)


