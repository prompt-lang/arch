# TBS：ThoughtBench-Set（标尺工具集）

> TBS 提供一套**标准化的能力标尺工具**，用于量化衡量不同 Agent 的思维能力。每个标尺包含标准化的测试用例、度量指标和基线数据，支持自动化执行和横向对比。

---

## 设计原则

1. **分层分级**：从基础能力到高级推理，每个层级有独立的标尺。
2. **标准化**：统一的测试格式、执行环境、评分标准，确保可比较性。
3. **自动化**：通过 `ppcli benchmark` 一键运行，结果机器可读。
4. **可扩展**：新标尺可按规范添加，不影响已有标尺。
5. **有基线**：每个标尺附带人类基线和已知模型的基线数据。

---

## 标尺目录

### 效率维度 (Efficiency)

| 标尺 ID | 名称 | 测量目标 | 测试用例数 |
|---------|------|----------|-----------|
| `TBS-IOE-001` | 基础算术 IOE | LLM 直接计算 vs neuro_task 执行基础算术 | 20 |
| `TBS-IOE-002` | 字符串处理 IOE | LLM 处理字符串 vs script 节点执行 | 15 |
| `TBS-IOE-003` | 排序算法 IOE | LLM 排序 vs 确定执行器排序（不同规模） | 10 |
| `TBS-IOE-004` | 递归计算 IOE | LLM 递归推理 vs Wasm 递归执行 | 8 |

### 记忆维度 (Memory)

| 标尺 ID | 名称 | 测量目标 | 测试用例数 |
|---------|------|----------|-----------|
| `TBS-MEM-001` | 短期信息保持 | 5 轮对话中关键信息回忆准确率 | 15 |
| `TBS-MEM-002` | 长期记忆检索 | 跨会话的事实记忆召回率 | 10 |
| `TBS-MEM-003` | 情景记忆时间线 | 按时间顺序回忆事件序列的准确率 | 12 |
| `TBS-MEM-004` | 记忆事务一致性 | 并发读写下的数据一致性 | 8 |

### 智能层级维度 (Intelligence Capacity & Scope)

| 标尺 ID | 名称 | 测量目标 | 测试用例数 |
|---------|------|----------|-----------|
| `TBS-ICS-001` | Local 快速响应 | 5 步内简单任务的成功率与延迟 | 25 |
| `TBS-ICS-002` | Global 长周期规划 | 20+ 步复杂任务的规划质量 | 10 |
| `TBS-ICS-003` | Hybrid 协作 | Local + Global 协作任务的完成率 | 8 |
| `TBS-ICS-004` | 层级路由正确性 | 任务是否正确路由到对应层级 | 20 |

### 推理深度维度 (Reasoning Depth)

| 标尺 ID | 名称 | 测量目标 | 测试用例数 |
|---------|------|----------|-----------|
| `TBS-RD-001` | 单步推理 | 直接因果推理的正确率 | 30 |
| `TBS-RD-002` | 多步链式推理 | 3-5 步逻辑链的正确率 | 20 |
| `TBS-RD-003` | 反事实推理 | "如果...会怎样"类推理的合理性 | 15 |
| `TBS-RD-004` | 数学证明 | 形式化证明的步骤正确率 | 10 |

### 自感知维度 (Self-Awareness)

| 标尺 ID | 名称 | 测量目标 | 测试用例数 |
|---------|------|----------|-----------|
| `TBS-SA-001` | 能力边界识别 | Agent 是否正确判断自身能否完成任务 | 20 |
| `TBS-SA-002` | 资源自适应 | 低资源场景下的策略调整合理性 | 15 |
| `TBS-SA-003` | 错误自我修正 | 发现错误后自主修正的成功率 | 12 |

### 规则处理维度 (Rule Engine)

| 标尺 ID | 名称 | 测量目标 | 测试用例数 |
|---------|------|----------|-----------|
| `TBS-RE-001` | 规则匹配吞吐量 | 不同规则数量下的匹配延迟 | 10 |
| `TBS-RE-002` | 规则冲突解决 | 多规则同时匹配时的冲突解决正确率 | 15 |
| `TBS-RE-003` | 规则热更新 | 运行时更新规则集的正确性与延迟 | 8 |

---

## 标尺定义格式

每个 TBS 标尺包含以下结构化定义：

```yaml
benchmark:
  id: "TBS-IOE-001"
  name: "基础算术 IOE"
  category: "efficiency"
  version: "1.0"
  description: "测量 LLM 直接计算与 neuro_task 执行基础算术的效率比值"

  # ── 测试用例 ──
  test_cases:
    - id: "arithmetic_01"
      input: { a: 3, b: 5 }
      task: "calculate a + b * 2"
      expected_output: 13
      deterministic_impl: "function add_multiply(a,b) { return a + b * 2; }"
      complexity: "simple"

    - id: "arithmetic_02"
      input: { n: 5 }
      task: "calculate factorial of n"
      expected_output: 120
      deterministic_impl: "function factorial(n) { return n <= 1 ? 1 : n * factorial(n-1); }"
      complexity: "medium"

  # ── 执行配置 ──
  execution:
    internal:
      node_type: "llm"
      model: "default"
      timeout_ms: 5000
    external:
      node_type: "neuro_task"
      language: "wasm"
      timeout_ms: 1000
    iterations: 10                    # 每个测试用例重复次数

  # ── 度量指标 ──
  metrics:
    - id: "ioe_ratio"
      formula: "median(T_internal / T_external)"
      unit: "ratio"
    - id: "accuracy_internal"
      formula: "correct_count / total_count"
      unit: "percentage"
    - id: "accuracy_external"
      formula: "correct_count / total_count"
      unit: "percentage"

  # ── 基线数据 ──
  baselines:
    - model: "gpt-4"
      date: "2026-01-15"
      ioe_ratio: 2.3
      accuracy_internal: 0.92
      accuracy_external: 1.0
    - model: "claude-opus-4"
      date: "2026-03-20"
      ioe_ratio: 1.8
      accuracy_internal: 0.95
      accuracy_external: 1.0
    - human_baseline:
      ioe_ratio: null                  # 人类不适用 IOE
      accuracy_internal: 0.99
```

---

## 执行命令

```bash
# 运行单个标尺
ppcli benchmark run --set TBS-IOE-001 --output results.json

# 运行整个维度
ppcli benchmark run --category efficiency --output results.json

# 运行全部标尺
ppcli benchmark run --suite thoughtbench --output results.json

# 与基线对比
ppcli benchmark run --set TBS-IOE-001 --compare-baseline

# 指定迭代次数
ppcli benchmark run --set TBS-IOE-001 --iterations 20
```

---

## 结果格式

```json
{
  "benchmark_id": "TBS-IOE-001",
  "timestamp": "2026-07-19T12:00:00Z",
  "model": "gpt-4",
  "pml_version": "1.16",
  "results": {
    "ioe_ratio": 2.1,
    "accuracy_internal": 0.93,
    "accuracy_external": 1.0,
    "test_cases_passed": 18,
    "test_cases_total": 20
  },
  "per_case": [
    {
      "case_id": "arithmetic_01",
      "ioe": 3.2,
      "internal_correct": true,
      "internal_time_ms": 320,
      "external_time_ms": 100
    }
  ],
  "comparison": {
    "baseline_model": "gpt-4",
    "baseline_date": "2026-01-15",
    "delta_ioe_ratio": -0.2,
    "delta_accuracy": +0.01
  }
}
```

---

## 标尺扩展规范

添加新标尺需满足以下要求：

1. **命名规范**：`TBS-{CATEGORY}-{SEQ}`，如 `TBS-IOE-005`
2. **最少 8 个测试用例**：确保统计意义
3. **包含基线数据**：至少一个已知模型的基线
4. **确定性外部实现**：每个测试用例需提供 `deterministic_impl`
5. **复杂度标注**：每个用例标注 `simple`, `medium`, `complex`
6. **文档**：每个标尺包含使用说明和已知局限性

---

## 与 TBH 的关系

TBS 提供标准化的"量尺"，TBH 定义"测量什么假设"。在 TBH 验证方案中可直接引用 TBS 标尺：

```yaml
# TBH 验证方案中引用 TBS 标尺
verification:
  metrics:
    primary: "accuracy"
    benchmark_ref: "TBS-RD-001"        # 使用推理深度标尺
    benchmark_ref: "TBS-MEM-001"       # 同时使用记忆标尺
```

TBS 是"工具箱"，TBH 是"实验设计"——二者共同组成 ThoughtBench 的完整评估体系。
