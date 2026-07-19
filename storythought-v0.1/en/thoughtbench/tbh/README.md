# TBH: ThoughtBench-Hypothesis

> TBH is a **hypothesis-driven Agent capability verification system**. Researchers or developers propose verifiable hypotheses about Agent behavior, and TBH provides a standardized verification framework to automate execution, collect evidence, and generate conclusions.

---

## Design Principles

1. **Hypotheses proposed by humans**: Hypotheses originate from researcher observation, theoretical deduction, or engineering intuition — TBH does not auto-generate hypotheses.
2. **Verification executed by system**: PML workflows automate test execution, eliminating human bias.
3. **Evidence is reproducible**: Each verification plan's inputs, execution environment, and outputs are fully recorded, enabling anyone to reproduce under identical conditions.
4. **Knowledge is cumulative**: Verification conclusions are recorded to PEM engineering memory, building an AGI R&D knowledge base.
5. **Progressive verification**: Start from simple hypotheses, gradually add complexity — avoid verifying overly complex propositions in one pass.

---

## Hypothesis Structure

Each TBH hypothesis contains the following fields:

```yaml
hypothesis:
  id: "TBH-001"
  title: "Prompt-in-Loop improves multi-step reasoning quality"
  proposer: "researcher@example.com"
  date: "2026-07-19"

  # ── Hypothesis Statement ──
  statement:
    given: "A complex problem requiring 3+ reasoning steps"
    when: "Agent uses Prompt-in-Loop mode (max 5 iterations)"
    then: "Final answer accuracy improves at least 30% over single-round answering"
    null_hypothesis: "Prompt-in-Loop does not significantly improve accuracy"

  # ── Verification Plan ──
  verification:
    method: "controlled_experiment"
    metrics:
      primary: "accuracy"
      secondary: ["latency_ms", "token_usage", "iterations_used"]
    sample_size: 100
    control_group:
      description: "Single-round direct answering"
      workflow: "./workflows/single_turn.pml"
    experiment_group:
      description: "Prompt-in-Loop multi-round iteration"
      workflow: "./workflows/loop_refinement.pml"
    dataset: "./datasets/multi_step_reasoning.ppdata"
    significance_level: 0.05

  # ── Verification Status ──
  status: "proposed"          # proposed | running | verified | rejected | inconclusive
  results: null               # Populated after verification completes
```

---

## Hypothesis Categories

### By Verification Dimension

| Category | Example Hypothesis |
|----------|-------------------|
| **Efficiency** | "Proportion of tasks with IOE > 1 increases as RBE rule count grows" |
| **Memory** | "Episodic memory enables > 90% context retention across 10 dialogue turns" |
| **Hierarchy** | "Local Planner path planning success rate within 5 steps shows no significant difference from Global Planner" |
| **Self-Awareness** | "With SOF, decision rationality improves 20% in low-battery scenarios" |
| **Security** | "Safe Prompts mechanism blocks 99% of Prompt Injection attacks" |
| **Evolution** | "After 100 Learning Session rounds, Agent accuracy on target task improves 50%" |

### By Verification Method

| Method | Use Case | PML Implementation |
|--------|----------|-------------------|
| **Controlled Experiment** | A/B testing, comparing two conditions | `control_group` + `experiment_group` |
| **Ablation Study** | Observe performance change after removing a component | Gradually disable SOF/RBE/Memory etc. |
| **Stress Test** | Behavior boundaries under extreme conditions | Increase input scale, reduce resource quotas |
| **Regression Test** | Ensure new versions don't degrade existing capabilities | Compare against baseline data |
| **Adversarial Test** | Robustness under malicious input | Prompt Injection, anomalous data |

---

## Verification Flow

```
1. Propose Hypothesis (Proposed)
   │
   ▼
2. Design Verification Plan
   │  - Select verification method
   │  - Define metrics and sample size
   │  - Write PML workflow
   │
   ▼
3. Execute Verification (Running)
   │  - ppcli benchmark run --hypothesis TBH-001
   │  - Auto-collect metric data
   │
   ▼
4. Analyze Results
   │  - Statistical significance test
   │  - Effect size calculation
   │  - Visualization report
   │
   ▼
5. Conclusion (Verified / Rejected / Inconclusive)
   │  - Record to PEM
   │  - Update knowledge base
   │
   ▼
6. Iterate: Revise hypothesis → Re-verify
```

---

## PML Integration

TBH verification plans are organized as PML Storylines:

```yaml
storyline:
  id: "verify_TBH-001"
  title: "Verify Prompt-in-Loop Hypothesis"

  phases:
    - id: "setup"
      title: "Prepare test environment and data"
    - id: "control_run"
      title: "Execute control group (single-round answering)"
    - id: "experiment_run"
      title: "Execute experiment group (Loop iteration)"
    - id: "analysis"
      title: "Statistical analysis and conclusion generation"

  buffer:
    items:
      - id: "TBH-001-01"
        title: "Run control group 100 tests"
        phase: "control_run"
      - id: "TBH-001-02"
        title: "Run experiment group 100 tests"
        phase: "experiment_run"
      - id: "TBH-001-03"
        title: "Calculate p-value and effect size"
        phase: "analysis"
      - id: "TBH-001-04"
        title: "Generate verification report"
        phase: "analysis"
        requires_issue_trigger: true    # Requires human review of conclusion
```

---

## Hypothesis Library

TBH maintains a versioned hypothesis library recording all proposed, verified, and rejected hypotheses:

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

## Relationship with TBS

TBH verification plans can reference TBS standardized benchmarks:

```yaml
verification:
  metrics:
    primary: "accuracy"
    benchmark_ref: "TBS-RD-001"    # Reference TBS reasoning depth benchmark
```

TBS provides the "how to measure" benchmark tools, TBH defines the "what to measure" hypothesis propositions — together forming the complete ThoughtBench system.
