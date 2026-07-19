# PML Abstract Hierarchical Concept Modeling

## Agent Hierarchy

Facing the computational work that different computing platforms can handle, capability layering is implemented.

### Local Planner

An agent that implements local task planning.

### SuperIntelligent Planner (SIP)

An agent that implements planning beyond local scope. The planning range of the agent is measured by the agent's planning tier ThoughtBench/ICS: Intelligent's Capacity and tiered Scope.

### Multi-Agent SuperIntelligent Planner (MA-SIP)

A federated super-intelligent planning system composed of multiple agents.

## Memory Hierarchy

### Fast Memory

Rapid-access working memory for immediate task context and short-term information retention.

### Slow Memory

Long-term persistent storage for learned patterns, experiences, and accumulated knowledge.

## Concept Abstraction

### Hierarchical Concept Modeling

PML supports modeling concepts at different levels of abstraction:

```
concepts:
  - id: "vehicle"
    level: "abstract"
    attributes: ["speed", "position"]
    
  - id: "car"
    level: "concrete"
    inherits: ["vehicle"]
    attributes: ["doors", "fuel_type"]
    
  - id: "sedan"
    level: "specific"
    inherits: ["car"]
    attributes: ["trunk_capacity"]
```

### Concept Binding in Workflows

```
pnodes:
  - id: classify_object
    type: llm
    process:
      prompt:
        system: |
          Available concepts:
          {{concepts.hierarchy}}
          Classify the input object into the most specific concept.
```

### Capability Scope Mapping

Each planning tier maps to specific capability scopes:

| Tier | Scope | Max Horizon | Typical Latency |
|------|-------|-------------|-----------------|
| Local Planner | `local` | 5-10 steps | < 500ms |
| SIP | `global` | 20-100 steps | 1-60s |
| MA-SIP | `federated` | 100+ steps | Minutes |
