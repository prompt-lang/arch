# Prompt-Lang Prompt Pipeline Extension

## 1. Design Principles
- **Separation of Representation & Execution**: Prompt content (ppl/ippl) is independent of workflow logic, with bidirectional conversion via ppcli tools.
- **Pipeline Orchestration**: mppl/mippl pipelines support sequential, conditional, and reusable prompt composition.
- **Native Loop Support**: `control.loop` makes Prompt In Loop a first-class citizen, without needing timer + script.
- **Context Injection**: Intrinsic Prompts auto-inject Agent state; Shared Prompts support common fragment reuse.
- **Security & Metadata**: Header protocol declares security rules, compression, encryption, modality, etc.
- **Data Acceleration**: pptensor/ppdata provide tensorized storage and acceleration for memory and markup.

## Representation Layer Enhancement: Standardized ppl/ippl File Protocol

### ppl & ippl Files
- **ppl** (Public Presentation Language): Human-readable prompt fragment, suffix `.ppl`, cannot be directly loaded and run by models.
- **ippl** (Internal Processing Language): Model-readable content format, suffix `.ippl`, can be directly loaded and run by models.
- **i-space**: The space composed of data corresponding to ippl; not human-readable, but model-readable.

### Referencing External ppl/ippl Files
Support `source` field in PNode's `process.prompt` to reference external files:
```
pnodes:
  - id: assistant
    type: llm
    process:
      prompt:
        system:
          source: "./prompts/system.ppl"
          format: "ppl"
        user: "{{workflow.input.query}}"
```

## pplc Compilation Tool
Provide CLI tool integrated in ppcli for ppl ↔ ippl conversion:

```
# Basic conversion
ppcli convert input.ppl --to ippl --output input.ippl
ppcli convert input.ippl --to ppl --output input.ppl

# Auto-detect input format
ppcli convert input.ppl --to ppdata --output data.ppdata

# Batch convert directory
ppcli convert ./prompts/ --pattern "*.ppl" --to ippl --out-dir ./ippl_cache/

# Use mapping table
ppcli convert input.ppl --to ippl --mapping sentiment_labels --output output.ippl
```

### Mapping List
Declare mapping rules in config for vocabulary replacement during conversion.
```
config:
  mapping_list:
    - id: "sentiment_labels"
      mapping:
        positive: 1
        negative: -1
        neutral: 0
```

## Orchestration Layer Enhancement: mppl / mippl Pipeline Nodes

`type: pipeline` node for sequentially composing multiple ppl/ippl fragments.
```
pnodes:
  - id: reasoning_pipeline
    type: pipeline
    mode: "mppl"
    sequences:
      - name: "preprocess"
        source: "./prompts/preprocess.ppl"
      - name: "reasoning"
        source: "./prompts/reasoning.ppl"
      - name: "format"
        source: "./prompts/format.ppl"
    variable_mapping:
      - input: "{{preprocess.output}}"
        output: "{{reasoning.input.user_message}}"
    conditioned:
      - condition: "{{workflow.vars.needs_reasoning}}"
        execute: ["reasoning"]
    output:
      schema:
        final_prompt: { type: string }
```

## Execution Layer Enhancement: Native Prompt In Loop

Add `loop` field in `control` for native loop execution.
```
pnodes:
  - id: iterative_refinement
    type: llm
    process:
      prompt:
        user: |
          Current iteration: {{workflow.vars.iteration}}
          Previous output: {{workflow.vars.last_output}}
          User question: {{workflow.input.query}}
    control:
      loop:
        condition: "{{output.quality < 0.8 and workflow.vars.iteration < 5}}"
        max_iterations: 10
        update_vars:
          - iteration: "{{workflow.vars.iteration + 1}}"
          - last_output: "{{output}}"
    output:
      schema:
        answer: { type: string }
        quality: { type: number }
```

## Context Enhancement: Shared Prompts & Intrinsic Prompts

### Shared Prompts
Define reusable common prompt fragments in `workflow.vars`.
```
workflow:
  vars:
    shared_prompts:
      system_context: |
        You are a professional assistant. User preferences: concise answers, avoid jargon.
      global_rules: |
        Answers must be based on provided context, do not fabricate information.
```

### Intrinsic Prompts
Add `intrinsic` field in PNode for auto-injecting Agent state information.
```
pnodes:
  - id: self_aware_agent
    type: llm
    intrinsic:
      - agent_name: "{{workflow.meta.name}}"
      - agent_version: "{{workflow.meta.version}}"
      - capabilities: "{{workflow.capabilities}}"
      - current_iteration: "{{workflow.vars.iteration}}"
    process:
      prompt:
        user: "{{workflow.input.query}}"
```

## Synthetic Prompts

New `type: synthetic` node for generating synthetic prompt data, suitable for data augmentation and test case generation.
```
pnodes:
  - id: augment_training
    type: synthetic
    method: "mix"
    sources:
      - source: "./prompts/base.ppl"
        weight: 0.6
      - source: "./prompts/variant.ppl"
        weight: 0.4
    augmentation:
      - type: "paraphrase"
        model: "gpt-3.5-turbo"
        temperature: 0.8
      - type: "noise_injection"
        rate: 0.05
    output:
      candidates: 10
      format: "ppl"
```

## Data Acceleration Formats

- **pptensor**: Prompt-Lang Tensors — optimized tensor storage for accelerating prompt-lang operations.
- **ppdata**: Unified data format for accelerated storage of Shared Prompts, Synthetic Prompts, and other generated data.
- **Prompt Script**: Scripting rules for automated prompt operations — programmatic prompt generation and manipulation.
