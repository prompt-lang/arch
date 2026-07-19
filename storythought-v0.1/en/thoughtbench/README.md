# ThoughtBench

> ThoughtBench is the **benchmark system for evaluating, verifying, and calibrating agent thinking capabilities** within the Prompt-Lang ecosystem.
> It consists of two complementary subsystems: **TBH (Hypothesis Verification)** and **TBS (Benchmark Tool Sets)**.

---

## Architecture

```
thoughtbench/
├── README.md           ← This file: system overview & navigation
├── tbh/                ← ThoughtBench-Hypothesis: hypothesis-driven verification
│   └── README.md
└── tbs/                ← ThoughtBench-Set: standardized capability benchmark tools
    └── README.md
```

---

## TBH: ThoughtBench-Hypothesis

**Problem**: During AGI R&D, researchers or developers propose **hypotheses** about Agent capabilities that need systematic verification.

**Core Ideas**:
- Hypotheses are **proposed by humans** — researchers formulate verifiable propositions based on observation, theory, or intuition
- Verification is **executed by the system** — PML workflows automatically run tests and collect evidence
- Results are **reproducible and cumulative** — each verification outcome builds a knowledge base guiding future R&D

**Example Hypotheses**:
- "Agent answer quality after 3 rounds of Prompt-in-Loop exceeds single-round answers"
- "With SOF self-ontology, Agent decision rationality improves 20% in resource-constrained scenarios"
- "RBE rule engine latency stays under 50ms when processing 100+ rules"
- "With episodic memory, Agent context retention rate > 90% across 10 dialogue turns"

See [`tbh/README.md`](./tbh/README.md)

---

## TBS: ThoughtBench-Set

**Problem**: A **standardized, quantifiable** toolset is needed to measure different Agents' thinking capabilities — analogous to IQ tests for human intelligence.

**Core Ideas**:
- Provides **tiered benchmarks** — from basic capabilities to advanced reasoning
- Each benchmark includes **standardized test cases, metrics, and baseline data**
- Supports **automated execution** — via `ppcli benchmark` commands
- Results are **comparable** — cross-Agent and cross-version thinking capabilities can be compared

**Benchmark Dimensions**:

| Dimension | Benchmark ID | Measurement Target |
|-----------|-------------|-------------------|
| Efficiency | `IOE` | Internal inference vs external computation efficiency ratio |
| Memory | `MEM` | Task completion quality with vs without memory |
| Intelligence Tier | `ICS` | Local/Global/Federated planning capability classification |
| Reasoning Depth | `RD` | Multi-step logical reasoning accuracy and step efficiency |
| Self-Awareness | `SA` | SOF-driven adaptive decision rationality |
| Rule Engine | `RE` | RBE rule matching throughput and accuracy |

See [`tbs/README.md`](./tbs/README.md)

---

## Usage Flow

```
1. Researcher proposes hypothesis ──→ TBH defines verification plan
2. TBH references TBS benchmarks ──→ Select appropriate capability dimensions
3. PML workflow executes ──→ Auto-collect data
4. Result analysis ──→ Hypothesis confirmed/rejected/revised
5. Knowledge accumulation ──→ Record to PEM engineering memory
```

---

## Relationship with PML System

| PML Component | Role in ThoughtBench |
|---------------|---------------------|
| **IOE Metric** | Core metric for efficiency dimension |
| **Memory Backend** | Data storage for memory dimension |
| **SOF** | Capability declaration for self-awareness dimension |
| **RBE** | Execution engine for rule processing dimension |
| **Storyline** | Narrative container for TBH verification plans |
| **PEM** | Knowledge accumulation for verification results |
| **ppcli benchmark** | Execution entry point for TBS benchmarks |
