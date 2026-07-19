# TBS: ThoughtBench-Set (Benchmark Tool Sets)

> TBS provides a suite of **standardized capability benchmark tools** for quantitatively measuring different Agents' thinking capabilities. Each benchmark includes standardized test cases, metrics, and baseline data, supporting automated execution and cross-comparison.

---

## Design Principles

1. **Tiered & Graded**: From basic capabilities to advanced reasoning, each tier has independent benchmarks.
2. **Standardized**: Unified test format, execution environment, and scoring criteria ensure comparability.
3. **Automated**: One-click execution via `ppcli benchmark`, machine-readable results.
4. **Extensible**: New benchmarks can be added per specification without affecting existing ones.
5. **Baselined**: Each benchmark includes human baselines and known model baseline data.

---

## Benchmark Catalog

### Efficiency Dimension

| Benchmark ID | Name | Measurement Target | Test Cases |
|-------------|------|-------------------|-----------|
| `TBS-IOE-001` | Basic Arithmetic IOE | LLM direct computation vs neuro_task basic arithmetic | 20 |
| `TBS-IOE-002` | String Processing IOE | LLM string processing vs script node execution | 15 |
| `TBS-IOE-003` | Sorting Algorithm IOE | LLM sorting vs deterministic executor sorting (varying scales) | 10 |
| `TBS-IOE-004` | Recursive Computation IOE | LLM recursive reasoning vs Wasm recursive execution | 8 |

### Memory Dimension

| Benchmark ID | Name | Measurement Target | Test Cases |
|-------------|------|-------------------|-----------|
| `TBS-MEM-001` | Short-Term Information Retention | Key info recall accuracy across 5 dialogue turns | 15 |
| `TBS-MEM-002` | Long-Term Memory Retrieval | Cross-session fact memory recall rate | 10 |
| `TBS-MEM-003` | Episodic Memory Timeline | Chronological event sequence recall accuracy | 12 |
| `TBS-MEM-004` | Memory Transaction Consistency | Data consistency under concurrent reads/writes | 8 |

### Intelligence Capacity & Scope Dimension

| Benchmark ID | Name | Measurement Target | Test Cases |
|-------------|------|-------------------|-----------|
| `TBS-ICS-001` | Local Fast Response | Success rate & latency for simple tasks within 5 steps | 25 |
| `TBS-ICS-002` | Global Long-Horizon Planning | Planning quality for 20+ step complex tasks | 10 |
| `TBS-ICS-003` | Hybrid Collaboration | Completion rate for Local + Global collaborative tasks | 8 |
| `TBS-ICS-004` | Tier Routing Correctness | Whether tasks are correctly routed to the appropriate tier | 20 |

### Reasoning Depth Dimension

| Benchmark ID | Name | Measurement Target | Test Cases |
|-------------|------|-------------------|-----------|
| `TBS-RD-001` | Single-Step Reasoning | Direct causal reasoning accuracy | 30 |
| `TBS-RD-002` | Multi-Step Chain Reasoning | 3-5 step logical chain accuracy | 20 |
| `TBS-RD-003` | Counterfactual Reasoning | "What if..." reasoning plausibility | 15 |
| `TBS-RD-004` | Mathematical Proof | Step correctness in formal proofs | 10 |

### Self-Awareness Dimension

| Benchmark ID | Name | Measurement Target | Test Cases |
|-------------|------|-------------------|-----------|
| `TBS-SA-001` | Capability Boundary Recognition | Whether Agent correctly judges its ability to complete tasks | 20 |
| `TBS-SA-002` | Resource Adaptation | Strategy adjustment rationality in low-resource scenarios | 15 |
| `TBS-SA-003` | Error Self-Correction | Success rate of autonomous correction after error detection | 12 |

### Rule Engine Dimension

| Benchmark ID | Name | Measurement Target | Test Cases |
|-------------|------|-------------------|-----------|
| `TBS-RE-001` | Rule Match Throughput | Match latency across varying rule counts | 10 |
| `TBS-RE-002` | Rule Conflict Resolution | Conflict resolution accuracy with multiple simultaneous matches | 15 |
| `TBS-RE-003` | Rule Hot-Reload | Correctness & latency of runtime rule set updates | 8 |

---

## Benchmark Definition Format

Each TBS benchmark contains the following structured definition:

```yaml
benchmark:
  id: "TBS-IOE-001"
  name: "Basic Arithmetic IOE"
  category: "efficiency"
  version: "1.0"
  description: "Measure the efficiency ratio of LLM direct computation vs neuro_task basic arithmetic execution"

  # ── Test Cases ──
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

  # ── Execution Configuration ──
  execution:
    internal:
      node_type: "llm"
      model: "default"
      timeout_ms: 5000
    external:
      node_type: "neuro_task"
      language: "wasm"
      timeout_ms: 1000
    iterations: 10                    # Repetitions per test case

  # ── Metrics ──
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

  # ── Baseline Data ──
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
      ioe_ratio: null                  # IOE not applicable to humans
      accuracy_internal: 0.99
```

---

## Execution Commands

```bash
# Run a single benchmark
ppcli benchmark run --set TBS-IOE-001 --output results.json

# Run an entire dimension
ppcli benchmark run --category efficiency --output results.json

# Run all benchmarks
ppcli benchmark run --suite thoughtbench --output results.json

# Compare against baseline
ppcli benchmark run --set TBS-IOE-001 --compare-baseline

# Specify iteration count
ppcli benchmark run --set TBS-IOE-001 --iterations 20
```

---

## Result Format

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

## Benchmark Extension Specification

New benchmarks must satisfy the following requirements:

1. **Naming Convention**: `TBS-{CATEGORY}-{SEQ}`, e.g., `TBS-IOE-005`
2. **Minimum 8 Test Cases**: Ensures statistical significance
3. **Include Baseline Data**: At least one known model baseline
4. **Deterministic External Implementation**: Each test case must provide `deterministic_impl`
5. **Complexity Annotation**: Each case annotated `simple`, `medium`, or `complex`
6. **Documentation**: Each benchmark includes usage instructions and known limitations

---

## Relationship with TBH

TBS provides standardized "measuring rulers"; TBH defines "what hypotheses to measure." TBH verification plans can directly reference TBS benchmarks:

```yaml
# TBH verification plan referencing TBS benchmarks
verification:
  metrics:
    primary: "accuracy"
    benchmark_ref: "TBS-RD-001"        # Use reasoning depth benchmark
    benchmark_ref: "TBS-MEM-001"       # Also use memory benchmark
```

TBS is the "toolbox"; TBH is the "experiment design" — together forming ThoughtBench's complete evaluation system.
