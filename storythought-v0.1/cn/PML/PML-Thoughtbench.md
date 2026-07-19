
# PML 系统基准测试设计

为评估基于 PML 构建的智能体系统，我们设计一套标准化的基准测试——**ThoughtBench**。该基准涵盖三个核心维度：**效率（IOE）**、**记忆能力** 和 **智能层级（ICS）**。每个维度包含若干测试用例和度量指标。


## 1. 效率基准：IOE (Inside-Outside Efficiency)

**目的**：衡量智能体在“内部计算”（AGI 模型直接推理）与“外部计算”（确定性执行器、PNode 等）之间的效率比值。IOE 反映了 PML 系统是否合理地将可确定计算卸载到高效执行器。

### 1.1 测试任务集

定义一组计算任务 `QA_i`，每个任务包含：
- **自然语言描述**（如“求两个数的和，并乘以 2”）
- **确定性函数等价实现**（如伪代码或 Python 函数）

示例任务：
1. `add_multiply(x,y)`：计算 `x + y*2`
2. `factorial(n)`：计算 n!（n≤10）
3. `string_reverse(s)`：反转字符串
4. `array_sum(arr)`：求和
5. `sort_list(lst)`：升序排序（长度 ≤10）

### 1.2 执行方式

- **内部执行**：由 LLM 节点（如 `type: llm`）直接通过自然语言理解并计算结果。记录时间 `T1(i)`，若结果错误则标记 `false`。
- **外部执行**：由 `type: neuro_task` 或 `type: function` 节点运行确定性代码执行相同任务。记录时间 `T2(i)`，若失败则 `false`。

### 1.3 度量指标

对于每个任务 \( i \)，若两者均成功，定义单任务 IOE：
\[
IOE_1(i) = \frac{T1(i)}{T2(i)}
\]

整体 IOE 为所有成功任务的中位数或几何平均：
\[
IOE = \text{median}\{IOE_1(i) \mid T1(i) \text{ and } T2(i) \text{ succeed}\}
\]

- **IOE < 1**：内部计算更快（AGI 推理优势）
- **IOE > 1**：外部计算更快（确定性执行优势）
- **IOE ≈ 1**：两者相当

PML 系统的理想目标是 **IOE > 1**（即优先将确定性任务卸载到外部执行器）。

### 1.4 复杂度分类

可根据任务复杂度（操作数、循环深度）将 IOE 分层，例如：
- `IOE_simple`：单步运算
- `IOE_medium`：少量循环（n≤5）
- `IOE_complex`：递归或较大输入


## 2. 记忆基准：Memory Efficiency

**目的**：比较智能体在有记忆（长期/短期记忆）和无记忆情况下的任务完成质量与效率。

### 2.1 测试任务集

设计多轮交互任务，需要依赖先前信息：

1. **重复信息记忆**：用户在第 1 轮给出 5 个关键词，第 5 轮询问“第 2 个关键词是什么”。
2. **上下文数学推理**：前 3 轮逐步给出部分数据，最后一轮要求计算总和。
3. **用户偏好学习**：前 2 轮询问用户喜好，第 3 轮推荐结果需符合之前偏好。
4. **会话延续**：中断 10 分钟后恢复对话，要求正确回忆之前话题。

### 2.2 执行模式

- **有记忆模式**：启用 `sessions` + `memory`（长期或短期），使用 `type: turn` 节点或 `llm_mode: chat` 并保存历史。
- **无记忆模式**：每个请求独立，不保存任何历史状态。

### 2.3 度量指标

对每个任务 \( j \) ，定义：
- **正确率** \(Acc_j\)：正确回答的比例
- **平均响应时间** \(RT_j\)：从用户输入到回复的时间

综合记忆得分：
\[
MemoryScore = \frac{1}{N} \sum_j \frac{Acc_j^{mem}}{Acc_j^{no\_mem}} \times \frac{RT_j^{no\_mem}}{RT_j^{mem}}
\]

- 若带记忆正确率更高且响应更快，则 MemoryScore > 1。


## 3. 智能层级基准：ICS (Intelligent's Capacity and tiered Scope)

**目的**：衡量智能体在不同规划层级（Local Planner vs. Global Planner）下的任务解决能力，并验证 `capability_scope` 是否正确限制。

### 3.1 测试任务集

设计三类任务，分别适配不同规划层级：

| 层级 | 任务示例 | 要求 |
|------|----------|------|
| **Local (短视距，快速响应)** | 实时避障、简单导航（5步内）、即时问答 | 响应时间 < 500ms |
| **Global (长周期，多步规划)** | 旅行计划（多城市）、供应链优化、论文写作 | 规划步数 > 20，允许分钟级延迟 |
| **Hybrid** | 先全局规划路线，再局部调整 | 需两个层级协作 |

### 3.2 执行配置

- 定义两个智能体：`local_planner` 和 `global_planner`，分别设置 `capability_scope.level` 为 `local` 和 `global`。
- 使用 `required_capability` 字段强制将任务路由到对应智能体。

### 3.3 度量指标

- **成功率**：任务完成比例
- **步数效率**：实际执行步数 / 理论最优步数
- **资源消耗**：CPU/内存/LLM token 数

对每个任务 \( k \) ，ICS 得分 = 成功率 × (1 - 步数浪费率) × (基准资源 / 实际资源)。

若 Local 任务被错误路由到 Global（或反之），则该任务计 0 分（验证权限对齐）。


## 完整基准套件执行

PML 提供命令行工具运行 ThoughtBench：

```bash
pml benchmark run thoughtbench --suite all --output results.json
```

选项：

--suite ioe|memory|ics：指定测试维度

--iterations 10：每个任务重复次数

--compare：与基线（如无优化 PML）对比

结果生成 JSON 报告，包含每个指标的数值及通过/失败状态。

## 与 PML 设计的关系

IOE 验证 type: neuro_task 和 type: function 节点的性能优势。

Memory 验证 sessions、memory 后端、type: turn 的有效性。

ICS 验证 capability_scope 和 required_capability 的正确实施。

这些基准将作为 PML 实现的质量门禁，确保各版本更新不降低核心能力。
