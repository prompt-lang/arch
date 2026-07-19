# PML Memory Enhancement
> Deep enhancement for Memory-Use. Adds **memory mode declaration** (deterministic/uncertain), **Episodic Memory** support, **MemPML efficient index views**, **memory transactions**, **memory observability** metrics, and **PEM Project Engineering Memory** as a MemPML application (comparable to and surpassing Anthropic Claude Code Rules). These extensions bring PML's memory subsystem closer to cognitive science models, significantly improving storage utilization efficiency for large-scale agent systems.


## 1. Memory Mode Declaration: Deterministic vs. Uncertain Memory

### 1.1 Background

- **Deterministic Memory**: Stores high-confidence, low-noise facts such as knowledge bases, compilation results. Suitable for precise queries.
- **Uncertain Memory**: Stores observations, predictions, or sensor data with confidence levels, capable of fusing multiple sources.

### 1.2 Syntax Extension

Add `mode` field in `memory_backends` or `config.memory.backends`.

- `mode: deterministic`: Default, entries do not automatically carry confidence.
- `mode: uncertain`: Each written memory auto-generates `confidence` field (0~1), overridable by user.

```
memory_backends:
  - id: "fact_db"
    mode: "deterministic"
  - id: "sensor_fusion"
    mode: "uncertain"
    default_confidence: 0.7
```

When writing via `type: memory` nodes, `confidence` can be explicitly specified. Queries can filter by confidence minimum.

## 2. Episodic Memory

### 2.1 Concept

Episodic memory stores agent experiences in chronological order, supporting retrieval by time window and event context. Distinct from semantic memory (facts), episodic memory emphasizes timeline.

### 2.2 New Backend Type

`type: episodic` backend auto-adds timestamp, event tag, and sequence number to each memory.

```
memory_backends:
  - id: "user_experiences"
    type: "episodic"
    retention_days: 30
```

### 2.3 Write & Query

Write auto-attaches temporal metadata; `event_tag` can be manually specified. Queries support time range and event tag filtering. Results auto-sorted chronologically with `timestamp` field.

## 3. MemPML: Efficient Memory Index Views

### 3.1 Function

MemPML is a data view pattern for extracting, transforming, and aggregating data from existing memory backends into dedicated index structures, accelerating model retrieval. Similar to database materialized views.

### 3.2 New Node: `type: mem_view`

```
pnodes:
  - id: active_user_view
    type: mem_view
    source: "user_profiles"
    projection: ["user_id", "last_active", "preferences"]
    filter: "last_active > now - 7d"
    refresh_interval: 300
    output: { view_id: "{{result.view_id}}" }
```

Views can be queried directly by subsequent `type: memory` nodes.

### 3.3 Serialization Format `mem_pml`

A new serialization format specifically for storing pre-computed embedding vectors, inverted indices, or hierarchical clustering results. Based on `ppdata` with added index headers.

```
- id: fast_index
  type: "vector"
  format: "mem_pml"
  index_type: "hnsw"
```

## 4. Memory Transaction

### 4.1 Motivation

Multi-step memory operations require atomicity: "read → modify → write" must succeed or fail as a whole, preventing intermediate states from being visible to other nodes.

### 4.2 Syntax

Add `memory_transaction` flag in `control`, wrapping multiple memory operations in scripts or composite nodes.

```
pnodes:
  - id: atomic_transfer
    type: script
    control:
      memory_transaction: true
    script: |
      async function run(input, ctx) {
          let from = await ctx.memory.read("account_a");
          let to = await ctx.memory.read("account_b");
          if (from.balance >= input.amount) {
              from.balance -= input.amount;
              to.balance += input.amount;
              await ctx.memory.write("account_a", from);
              await ctx.memory.write("account_b", to);
          } else {
              throw new Error("Insufficient funds");
          }
          return { success: true };
      }
```

During transactions, related memory keys are locked (pessimistic) or use MVCC. If any step fails, all writes auto-rollback.

### 4.3 Nested Transactions

A transaction can invoke another sub-workflow marked `memory_transaction: true`, implementing nested transactions (following two-phase commit).

## 5. Memory Observability

### 5.1 Auto Metric Collection

PML runtime auto-collects Prometheus-style metrics for each memory backend:

- `memory_qps`: Operations per second, by operation type (read/write/search)
- `memory_latency_seconds`: Latency distribution (histogram)
- `memory_hit_rate`: Cache hit rate
- `memory_uncertainty_avg`: Average confidence of uncertain memory

### 5.2 Built-in Audit Logging

For critical memory operations (transaction commits, view refreshes), audit logging can be configured.

## 6. Complete Example: Episodic + Uncertain + Transaction

Defines two memory backends: `trip_log` (type episodic, 7-day retention) and `sensor_obs` (mode uncertain, default confidence 0.5). Workflow records uncertain sensor observations, fuses them through a transaction into episodic memory, and creates a fast-query view.

## 7. PEM: Project Engineering Memory (MemPML Application)

### 7.1 Background & Motivation

In Harness/Loop engineering scenarios, Agents need to continuously accumulate understanding of the project across multiple iterations (even across sessions): architectural conventions, historical decisions, error patterns, build commands, etc. This knowledge is **deterministic, structured, and needs cross-team sharing** — fundamentally different from uncertain sensor observations.

**PEM (Project Engineering Memory) is not a new PML extension, but a domain application of MemPML.** It leverages MemPML's existing `mem_view` + `mem_pml` + `memory_transaction` mechanisms to provide a structured, queryable, auto-accumulating persistent memory layer for Agent engineering.

### 7.2 Comparison with Anthropic Claude Code Rules

Claude Code provides two layers of memory — CLAUDE.md (human-written static instructions) and Auto Memory (Claude's self-written unstructured notes). PEM provides fundamental enhancements:

| Capability | CLAUDE.md (Human) | Auto Memory (Claude) | PML PEM (MemPML App) |
|------------|:---:|:---:|:---:|
| **Content Source** | Manual writing | Agent auto-records | Agent auto-accumulates + human-editable |
| **Structure Level** | Markdown text | Unstructured Markdown | Schema-driven structured storage |
| **Queryability** | Full-text scan | Full-text scan | `mem_view` projection/filter + `mem_pml` vector semantic search |
| **Index Acceleration** | None | None | hnsw vector index + inverted index |
| **Deterministic Guarantee** | None | None | `mode: deterministic` explicit semantics |
| **Atomic Transactions** | None | None | `memory_transaction` multi-step write consistency |
| **View Projection** | None | None | `type: mem_view` per-domain projection |
| **Cross-Team Sharing** | ✅ Git VCS | ❌ Machine-local only | ✅ `sync: git` + `ppdata` serialization |
| **Auto-Accumulation** | ❌ Manual updates | ✅ Auto but unstructured | ✅ Auto + structured schema |
| **Cross-Session Persistence** | ✅ | ✅ | ✅ |
| **SOF Integration** | ❌ | ❌ | ✅ Query `self.cognitive` for strategy |
| **Path-Level Rules** | ✅ `.claude/rules/` | ❌ | Via `mem_view.filter` equivalent |

### 7.3 Architecture: PEM in MemPML Layers

```
┌──────────────────────────────────────────────────────────┐
│           PEM Application Layer                           │
│  mem_view: eng_context    mem_view: decisions             │
│  (conventions/architecture) (decision records)            │
│         │                        │                        │
│         ▼                        ▼                        │
│  ┌──────────────────────────────────────────────────┐    │
│  │     mem_pml format · hnsw index · semantic search │    │
│  └──────────────────────┬───────────────────────────┘    │
│                         │                                 │
│                         ▼                                 │
│  ┌──────────────────────────────────────────────────┐    │
│  │  memory_backend: deterministic · sync: git        │    │
│  │  Raw: engineering facts, decisions, patterns      │    │
│  └──────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

### 7.4 Syntax & Application Examples

**Engineering memory backend:**

```yaml
memory_backends:
  - id: "engineering_kb"
    mode: "deterministic"
    format: "mem_pml"
    index_type: "hnsw"
    sync: "git"
    audit:
      enabled: true
      log_operations: ["write", "delete"]
```

**Engineering context view (CLAUDE.md equivalent):**

```yaml
pnodes:
  - id: view_engineering_context
    type: mem_view
    source: "engineering_kb"
    projection:
      - "module"
      - "convention"
      - "build_command"
      - "constraint"
    filter: "category == 'context'"
    refresh_interval: 300
```

**Decision records view (Auto Memory equivalent):**

```yaml
  - id: view_decisions
    type: mem_view
    source: "engineering_kb"
    projection:
      - "decision_id"
      - "timestamp"
      - "agent_id"
      - "task_summary"
      - "decision_rationale"
      - "outcome"
      - "confidence"
    filter: "category == 'decision' AND confidence > 0.7"
    refresh_interval: 60
```

### 7.5 Claude Code → PEM Migration Map

| Claude Code Concept | PEM Equivalent | Migration Benefit |
|--------------------|----------------|-------------------|
| `CLAUDE.md` project instructions | `mem_view: eng_context` (category=context) | Full text → structured queryable view |
| `CLAUDE.md` build commands | `engineering_kb` category=context, type=build_command | Per-module query, versioned |
| `.claude/rules/*.md` path rules | `mem_view` + `filter: "path LIKE 'src/api/%'"` | Equivalent + semantic search |
| Auto Memory auto notes | `mem_view: decisions` + auto-write | Unstructured local → structured shared |
| Auto Memory `MEMORY.md` index | `mem_pml` + hnsw index | 200-line text → vector semantic search |
| Dynamic Workflows Harness | PML Pipeline + `control.loop` | JS script → declarative workflow |
| `/memory` command | `type: memory` search node | Programmatic query |

### 7.6 Design Principles

1. **PEM is a MemPML domain application, not a new extension** — reuses all existing mechanisms.
2. **Deterministic engineering knowledge vs uncertain observations** — conventions and decisions are deterministic (`mode: deterministic`).
3. **Auto-accumulate, not manual maintenance** — Harness loop auto-writes decision records.
4. **Git sync for cross-team sharing** — solves Claude Auto Memory's "machine-local" limitation.
5. **SOF integration for adaptive engineering** — Agent queries self-capability boundaries to adjust strategy.


## Core Features

- **Deterministic/Uncertain Memory Modes**: Distinguish reliable facts from confidence-bearing observations.
- **Episodic Memory**: Timeline and event-sequence storage and query.
- **MemPML Views**: Pre-computed index structures accelerating model retrieval.
- **Memory Transactions**: Atomic multi-step operations ensuring data consistency.
- **Observability**: Metrics and audit logs for tuning and security auditing.
- **PEM Project Memory**: MemPML's engineering domain application providing structured, queryable, cross-team-shared persistent engineering memory — comparable to and surpassing Anthropic Claude Code's CLAUDE.md + Auto Memory system.
