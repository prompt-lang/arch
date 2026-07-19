# TBH：ThoughtBench-Hypothesis（假设验证）

> TBH 是一个**假设驱动的 Agent 能力验证体系**。研究者或开发者提出关于 Agent 行为的可验证假设，TBH 提供标准化的验证框架来自动化执行验证、收集证据、生成结论。

---

## 设计原则

1. **假设由人类提出**：假设来源于研究者的观察、理论推演或工程直觉，TBH 不自动生成假设。
2. **验证由系统执行**：PML 工作流自动化运行测试，消除人为偏差。
3. **证据可复现**：每个验证方案的输入、执行环境、输出完整记录，支持任何人在相同条件下重现。
4. **知识可积累**：验证结论记录到 PEM 工程记忆，构成 AGI 研发的知识库。
5. **渐进式验证**：从简单假设开始，逐步叠加复杂度，避免一次性验证过于复杂的命题。

---

## 假设结构

每个 TBH 假设包含以下字段：

```yaml
hypothesis:
  id: "TBH-001"
  title: "Prompt-in-Loop 提升多步推理质量"
  proposer: "researcher@example.com"
  date: "2026-07-19"

  # ── 假设命题 ──
  statement:
    given: "一个需要 3 步以上推理的复杂问题"
    when: "Agent 使用 Prompt-in-Loop 模式（最多 5 轮迭代）"
    then: "最终答案的正确率比单轮回答提升至少 30%"
    null_hypothesis: "Prompt-in-Loop 不会显著提升正确率"

  # ── 验证方案 ──
  verification:
    method: "controlled_experiment"
    metrics:
      primary: "accuracy"
      secondary: ["latency_ms", "token_usage", "iterations_used"]
    sample_size: 100
    control_group:
      description: "单轮直接回答"
      workflow: "./workflows/single_turn.pml"
    experiment_group:
      description: "Prompt-in-Loop 多轮迭代"
      workflow: "./workflows/loop_refinement.pml"
    dataset: "./datasets/multi_step_reasoning.ppdata"
    significance_level: 0.05

  # ── 验证状态 ──
  status: "proposed"          # proposed | running | verified | rejected | inconclusive
  results: null               # 验证完成后填充
```

---

## 假设分类

### 按验证维度分类

| 类别 | 示例假设 |
|------|----------|
| **效率假设** | "IOE > 1 的任务比例随 RBE 规则数增加而提升" |
| **记忆假设** | "情景记忆使 Agent 在 10 轮对话中上下文保持率 > 90%" |
| **层级假设** | "Local Planner 在 5 步内的路径规划成功率与 Global Planner 无显著差异" |
| **自感知假设** | "引入 SOF 后，低电量场景下的决策合理性提升 20%" |
| **安全性假设** | "Safe Prompts 机制能拦截 99% 的 Prompt Injection 攻击" |
| **进化假设** | "经过 100 轮 Learning Session 后，Agent 在目标任务的正确率提升 50%" |

### 按验证方法分类

| 方法 | 适用场景 | PML 实现 |
|------|----------|----------|
| **对照实验** | A/B 测试，比较两组条件 | `control_group` + `experiment_group` |
| **消融实验** | 移除某个组件后观察性能变化 | 逐步禁用 SOF/RBE/Memory 等 |
| **压力测试** | 极端条件下的行为边界 | 增大输入规模、降低资源配额 |
| **回归测试** | 确保新版本不降低已有能力 | 对比基线数据 |
| **对抗测试** | 恶意输入下的鲁棒性 | Prompt Injection、异常数据 |

---

## 验证流程

```
1. 提出假设 (Proposed)
   │
   ▼
2. 设计验证方案
   │  - 选择验证方法
   │  - 定义指标与样本量
   │  - 编写 PML 工作流
   │
   ▼
3. 执行验证 (Running)
   │  - ppcli benchmark run --hypothesis TBH-001
   │  - 自动收集指标数据
   │
   ▼
4. 分析结果
   │  - 统计显著性检验
   │  - 效应量计算
   │  - 可视化报告
   │
   ▼
5. 结论 (Verified / Rejected / Inconclusive)
   │  - 记录到 PEM
   │  - 更新知识库
   │
   ▼
6. 迭代：修正假设 → 重新验证
```

---

## PML 集成

TBH 验证方案以 PML Storyline 的形式组织：

```yaml
storyline:
  id: "verify_TBH-001"
  title: "验证 Prompt-in-Loop 假设"

  phases:
    - id: "setup"
      title: "准备测试环境与数据"
    - id: "control_run"
      title: "执行对照组（单轮回答）"
    - id: "experiment_run"
      title: "执行实验组（Loop 迭代）"
    - id: "analysis"
      title: "统计分析与结论生成"

  buffer:
    items:
      - id: "TBH-001-01"
        title: "运行对照组 100 次测试"
        phase: "control_run"
      - id: "TBH-001-02"
        title: "运行实验组 100 次测试"
        phase: "experiment_run"
      - id: "TBH-001-03"
        title: "计算 p-value 与效应量"
        phase: "analysis"
      - id: "TBH-001-04"
        title: "生成验证报告"
        phase: "analysis"
        requires_issue_trigger: true    # 需人工审核结论
```

---

## 假设库

TBH 维护一个版本化的假设库，记录所有已提出、已验证、已被推翻的假设：

```
tbh/
├── README.md
├── hypotheses/
│   ├── TBH-001_prompt_in_loop.yaml
│   ├── TBH-002_sof_energy_decision.yaml
│   └── TBH-003_rbe_latency_scale.yaml
├── results/
│   ├── TBH-001_result.json
│   └── TBH-002_result.json
└── workflows/
    ├── single_turn.pml
    └── loop_refinement.pml
```

---

## 与 TBS 的关系

TBH 验证方案中可以引用 TBS 的标准化标尺：

```yaml
verification:
  metrics:
    primary: "accuracy"
    benchmark_ref: "TBS-RD-001"    # 引用 TBS 推理深度标尺
```

TBS 提供"怎么测"的标尺工具，TBH 定义"测什么"的假设命题——二者互补构成完整的 ThoughtBench 体系。
