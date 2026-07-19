# PML System Benchmark Design

To evaluate agent systems built on PML, we design a standardized benchmark suite — **ThoughtBench**. The benchmark covers three core dimensions: **Efficiency (IOE)**, **Memory Capability**, and **Intelligence Tier (ICS)**. Each dimension includes several test cases and measurement metrics.


## 1. Efficiency Benchmark: IOE (Inside-Outside Efficiency)

**Purpose**: Measure the efficiency ratio between "internal computation" (AGI model direct inference) and "external computation" (deterministic executors, PNodes, etc.). IOE reflects whether the PML system appropriately offloads deterministic computation to efficient executors.

### 1.1 Test Task Set

Define a set of computational tasks `QA_i`, each containing:
- **Natural language description** (e.g., "calculate the sum of two numbers and multiply by 2")
- **Deterministic function equivalent implementation** (e.g., pseudocode or Python function)

Example tasks:
1. `add_multiply(x,y)`: compute `x + y*2`
2. `factorial(n)`: compute n! (n≤10)
3. `string_reverse(s)`: reverse a string
4. `array_sum(arr)`: compute sum
5. `sort_list(lst)`: sort in ascending order (length ≤10)

### 1.2 Execution Modes

- **Internal Execution**: LLM node (e.g., `type: llm`) directly understands natural language and computes results. Record time `T1(i)`, mark `false` if result is incorrect.
- **External Execution**: `type: neuro_task` or `type: function` node runs deterministic code for the same task. Record time `T2(i)`, mark `false` on failure.

### 1.3 Metrics

For each task \( i \), if both succeed, define per-task IOE:
\[
IOE_1(i) = \frac{T1(i)}{T2(i)}
\]

Overall IOE is the median or geometric mean of all successful tasks:
\[
IOE = \text{median}\{IOE_1(i) \mid T1(i) \text{ and } T2(i) \text{ succeed}\}
\]

- **IOE < 1**: Internal computation is faster (AGI inference advantage)
- **IOE > 1**: External computation is faster (deterministic execution advantage)
- **IOE ≈ 1**: Roughly equivalent

The ideal goal for PML systems is **IOE > 1** (i.e., prioritize offloading deterministic tasks to external executors).

### 1.4 Complexity Classification

IOE can be stratified by task complexity (operand count, loop depth), e.g.:
- `IOE_simple`: single-step operations
- `IOE_medium`: few loops (n≤5)
- `IOE_complex`: recursion or large inputs


## 2. Memory Benchmark: Memory Efficiency

**Purpose**: Compare agent task completion quality and efficiency with memory (long-term/short-term) versus without memory.

### 2.1 Test Task Set

Design multi-turn interaction tasks requiring reliance on prior information:

1. **Repeated Information Recall**: User provides 5 keywords in round 1, asks "what was the 2nd keyword" in round 5.
2. **Contextual Math Reasoning**: Partial data given incrementally across 3 rounds, final round asks for total sum.
3. **User Preference Learning**: First 2 rounds query user preferences, round 3 recommendation must match prior preferences.
4. **Session Continuation**: Resume conversation after 10-minute interruption, must correctly recall prior topic.

### 2.2 Execution Modes

- **With Memory**: Enable `sessions` + `memory` (long-term or short-term), use `type: turn` nodes or `llm_mode: chat` with history preservation.
- **Without Memory**: Each request is independent, no historical state preserved.

### 2.3 Metrics

For each task \( j \), define:
- **Accuracy** \(Acc_j\): proportion of correct responses
- **Average Response Time** \(RT_j\): time from user input to reply

Composite memory score:
\[
MemoryScore = \frac{1}{N} \sum_j \frac{Acc_j^{mem}}{Acc_j^{no\_mem}} \times \frac{RT_j^{no\_mem}}{RT_j^{mem}}
\]

- If memory yields higher accuracy and faster responses, MemoryScore > 1.


## 3. Intelligence Tier Benchmark: ICS (Intelligent's Capacity and tiered Scope)

**Purpose**: Measure agent task-solving capability across different planning tiers (Local Planner vs. Global Planner), and verify that `capability_scope` is correctly enforced.

### 3.1 Test Task Set

Design three categories of tasks suited to different planning tiers:

| Tier | Example Tasks | Requirements |
|------|---------------|--------------|
| **Local (short-horizon, fast response)** | Real-time obstacle avoidance, simple navigation (≤5 steps), instant Q&A | Response time < 500ms |
| **Global (long-horizon, multi-step planning)** | Travel planning (multi-city), supply chain optimization, essay writing | Planning steps > 20, minute-level latency acceptable |
| **Hybrid** | Global route planning first, then local adjustments | Requires collaboration across both tiers |

### 3.2 Execution Configuration

- Define two agents: `local_planner` and `global_planner`, with `capability_scope.level` set to `local` and `global` respectively.
- Use `required_capability` field to enforce task routing to the corresponding agent.

### 3.3 Metrics

- **Success Rate**: proportion of tasks completed
- **Step Efficiency**: actual execution steps / theoretically optimal steps
- **Resource Consumption**: CPU/memory/LLM token count

For each task \( k \), ICS score = Success Rate × (1 - step waste rate) × (baseline resources / actual resources).

If a Local task is incorrectly routed to Global (or vice versa), the task scores 0 (verifying permission alignment).


## Full Benchmark Suite Execution

PML provides CLI tools to run ThoughtBench:

```bash
pml benchmark run thoughtbench --suite all --output results.json
```

Options:

--suite ioe|memory|ics: specify test dimension

--iterations 10: repetitions per task

--compare: compare against baseline (e.g., unoptimized PML)

Results generate a JSON report with per-metric values and pass/fail status.

## Relationship to PML Design

IOE validates the performance advantages of `type: neuro_task` and `type: function` nodes.

Memory validates the effectiveness of `sessions`, `memory` backends, and `type: turn`.

ICS validates the correct implementation of `capability_scope` and `required_capability`.

These benchmarks serve as quality gates for PML implementations, ensuring version updates do not degrade core capabilities.
