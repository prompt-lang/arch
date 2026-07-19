# Agentic Learning Extension

# PML Agentic Learning 扩展

> 基于 PML，新增对智能体学习（Agentic Learning）的一流支持。通过声明式学习脚本、快速训练任务、学习会话管理等特性，使 PML 工作流能够驱动模型的自我进化、知识蒸馏、经验回放及多智能体协作学习。


## 核心概念

| 概念 | 说明 | PML 实现 |
|------|------|----------|
| **Learning Script** | 定义单步学习逻辑（前向、损失、反向）的 PMLScript | `type: script` + `learning_script: true` |
| **Training Task** | 一个完整的训练过程，包含模型、数据、超参数、学习脚本 | `type: training` 节点 |
| **Learning Session** | 训练任务的一次执行实例，支持断点续训和早停 | `workflow` + `config.checkpoint` |
| **Experience Replay** | 从记忆缓存中采样历史经验用于训练 | `memory` 后端 + `type: memory` 节点 |
| **Curriculum Learning** | 动态调整训练样本的难度或顺序 | `planning` 节点生成数据管道 |
| **Knowledge Distillation** | 大模型（教师）指导小模型（学生）进行迁移学习 | `control.loop` + 比较评估节点 |



## 新增节点：type: training
training 节点封装了一个完整的训练任务，支持模型加载、数据流、学习循环及指标收集。

#### 语法
```
pnodes:
  - id: my_trainer
    type: training
    model: "bert-base"                 # 模型标识（本地路径或远程服务）
    dataset:
      source: "./data/train.ppdata"    # 数据集来源
      batch_size: 8
      shuffle: true
    learning_script: "supervised_step" # 引用 learning_script
    hyperparams:
      learning_rate: 0.001
      epochs: 3
    optimizer: "adam"                  # 可选，内置优化器
    control:
      loop:                            # 迭代控制（epoch 循环）
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
节点执行时，NNOS 会加载 model 到执行环境（本地进程或推理服务）。

dataset 支持 ppdata、json、markup 查询结果或 memory 后端。

learning_script 必须是一个已声明的学习脚本（见第 4 节）。

control.loop 用于控制训练轮次（epoch）。每个 epoch 遍历全部 batch，每 batch 调用一次学习脚本。

训练指标自动记录，可通过 metrics 导出到外部系统。

#### 与 control.loop 的集成

training 节点内部隐式包含两层循环：batch 循环（由节点自动处理）和 epoch 循环（由用户通过 control.loop 控制）。用户只需关注 epoch 级条件，节点会自动迭代数据加载器。

### 学习脚本（Learning Script）
学习脚本是一个特殊的 PMLScript，用于定义单步训练逻辑。在 scripts 中通过 learning_script: true 标记。

#### 脚本接口
学习脚本必须导出一个 learn 函数，签名如下：
```
async function learn(sample, model, ctx) {
    // sample 是一个数据样本（对象）
    // model 是已加载的模型对象（提供 forward, backward, update 等方法）
    // ctx 包含 hyperparams（如 learning_rate）
    // 返回值：{ loss: number, ... } 任意字段将被记录
}
```

#### 示例
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

NNOS 运行时为每个训练任务创建独立的模型沙箱，脚本内可安全调用模型 API。

学习脚本也可调用外部训练库（如 PyTorch、TensorFlow），但需要节点环境具备相应依赖。

### 学习会话（Learning Session）
学习会话定义了一个长期运行的训练工作流，支持断点续训、早停和实验跟踪。

#### 会话配置
在 workflow 中启用 checkpoint，并设置 session_id。

```
workflow:
  input:
    session_id: string
  checkpoint:
    enabled: true
    interval_secs: 60
    storage: "s3://ml-runs/{{session_id}}"
```

检查点保存内容：当前 epoch、模型权重（或权重路径）、优化器状态、随机数种子。

工作流恢复时，从最近的检查点继续执行。

#### 早停与条件终止
可以在工作流中使用 script 节点判断验证集指标，主动调用 `ctx.terminate_workflow` 终止会话。

```
pnodes:
  - id: early_stop
    type: script
    script: |
      function run(input, ctx) {
          if (input.val_accuracy > 0.95) {
              ctx.terminate_workflow("early_stop", { best_acc: input.val_accuracy });
          }
      }
```

###  与现有特性的集成
#### 记忆（Memory）与经验回放
利用 memory 后端存储历史经验（状态、动作、奖励），使用 type: memory 节点进行采样。

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
    # ...
```

可以设置采样优先级（如 TD-error），通过标记的 Q 值或记忆元数据实现。

#### 合成 Prompt（Synthetic）生成训练数据
type: synthetic 节点可生成增强样本，用于数据增广或课程学习。

```
- id: augment_data
  type: synthetic
  method: "augment"
  sources: [{ source: "./base_samples.ppdata" }]
  augmentation:
    - type: "paraphrase"
      count: 5
  output: { augmented_set: "{{result}}" }
```

#### 多智能体协作学习
多个智能体（agent 类型）可以共享一个记忆后端，实现异步或同步的经验共享。

```
agents:
  - id: explorer
    memory_backend: "shared_pool"
  - id: learner
    memory_backend: "shared_pool"
```

#### 标记（Markup）驱动的课程学习
利用标记的 Q 值作为样本难度，通过 markup_query 动态调整采样顺序。

```
- id: curriculum_sampler
  type: markup_query
  domain: "training_set"
  query: "$..[*]"
  q_sort: { order: "asc" }        # 从易到难
  limit: 16
```

### 完整示例：小样本分类快速训练
```
pml:
  version: "1.7"

meta:
  name: "few_shot_learner"

workflow:
  input:
    support_set: array
    query: string
  vars:
    epoch: 0
  nodes: [prepare, train, infer]

pnodes:
  - id: prepare
    type: script
    script: |
      function run(input, ctx) {
          let dataset = input.support_set.map(ex => ({ input: ex.text, label: ex.label }));
          return { dataset };
      }

  - id: train
    type: training
    model: "local:bert-tiny"
    dataset:
      source: "{{prepare.output.dataset}}"
      batch_size: 4
    learning_script: "supervised_step"
    hyperparams: { learning_rate: 0.01, epochs: 3 }
    control:
      loop:
        condition: "{{epoch < 3}}"
        update_vars:
          - epoch: "{{epoch + 1}}"
    output: { model_artifact: "{{result.model_artifact}}" }

  - id: infer
    type: llm
    process:
      prompt_script: "build_inference_prompt"
      script_input:
        support: "{{workflow.input.support_set}}"
        query: "{{workflow.input.query}}"
    output: { category: "{{result.content}}" }

scripts:
  - id: supervised_step
    learning_script: true
    language: "python"
    code: |
      def learn(sample, model, ctx):
          pred = model.forward(sample["input"])
          loss = ((pred - sample["label"]) ** 2).mean()
          model.backward(loss)
          model.update(ctx.learning_rate)
          return {"loss": loss}
```

### 工具链支持
```
ppcli learn：启动学习会话。
ppcli learn session.pml --session-id exp_001 --input support.json

ppcli model export：导出训练后的模型 artifact（如 ONNX、PML 内部的 ippl 表示）。

ppcli model serve：将模型作为 MCP 工具暴露。
```
