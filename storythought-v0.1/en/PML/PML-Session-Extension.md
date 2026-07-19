# PML: Session, Context Separation, Multi-Step Interaction & Step Frequency Control

> For conversational, interactive, and long-running agent systems, introduces **Session objects**, **Explicit/Implicit Context Separation**, **Inloop Session mode**, **Turn nodes (MSCI)**, and **Step Frequency Control (StepFreq)**. These extensions enable PML to declaratively manage shared context across multi-turn interactions, adaptively control environment interaction frequency, and optimize energy consumption and task efficiency.

## Design Goals

- **Session Objectification**: Encapsulate a conversation or continuous interaction as a session entity with independent ID, participants, topic, and lifecycle.
- **Context Classification & Fusion**: Distinguish explicit context (directly provided) from implicit context (auto-retrieved from memory, markup, history), auto-injected into LLM at runtime.
- **Continuous Context Interaction**: Support Inloop Session where context is not released during the session, multi-step interactions share state.
- **Turn Mechanism (MSCI)**: Provide explicit turn nodes for multi-step context interaction, supporting turn counting, strategy selection, and fallback.
- **Dynamic Step Frequency**: Agents can dynamically adjust interaction frequency with the environment, linked to energy and efficiency metrics.

## Core Concepts

| Concept | Description | PML Implementation |
|---------|-------------|-------------------|
| **Session** | Encapsulates context for a continuous interaction, containing session ID, topic, participants, explicit context, implicit context, history | Top-level `sessions` field + `type: session` node |
| **Explicit Context** | Prompt content directly provided by workflow nodes | `llm.context.explicit` field |
| **Implicit Context** | Content auto-retrieved from memory, markup, session history, Agent state | `llm.context.implicit` field |
| **Inloop Session** | Context not released during session, supports unlimited interaction rounds until explicit close | `session_mode: inloop` |
| **Turn (MSCI)** | Single interaction step in multi-step context interaction, containing user input, system output, turn state | `type: turn` node |
| **StepFreq** | Agent-environment interaction frequency (Hz), dynamically adjustable, affecting energy and response speed | `control.step_freq` + `ctx.setStepFreq()` |

## Syntax & Feature Extensions

### Session Object

In the `workflow` top level, add `sessions` field to declare sessions used in the workflow. Each session has a unique identifier, type, bound device or topic, TTL, and checkpoint policy.

```
pnodes:
  - id: start_session
    type: session
    operation: create
    session_id: "{{workflow.input.session_id}}"
    context:
      explicit: { system_prompt: "You are a helpful assistant." }
      implicit_sources:
        - type: memory
          backend: "long_term"
          query: "user:{{workflow.input.user_id}}"
        - type: markup
          domain: "user_profile"
          filter: "active == true"
    output: { session_handle: "{{result.handle}}" }
```

### Explicit & Implicit Context Separation

`type: llm` nodes add a `context` field to configure explicit and implicit sources separately.

- **Explicit context**: Directly provided strings or data bindings.
- **Implicit context**: Specifies retrieval rules from memory, markup, and sessions.

At runtime, PML automatically formats implicit content as system prompts, prepended before explicit context, then sends to the LLM.

### Inloop Session Mode

Set `session_mode: inloop` in `workflow` to indicate the entire workflow is a continuous session where context is not released between external calls. Typical applications: chatbots, game AI.

Parameters:
- `max_turns`: Optional, maximum interaction rounds.
- `timeout_sec`: Idle timeout; session auto-archived on timeout.

### Turn Node (MSCI)

New node type `type: turn`, specifically for implementing each step of multi-step context interaction. It has built-in turn counting, turn strategy, and end conditions.

```
pnodes:
  - id: dialogue_turn
    type: turn
    input:
      msg: "{{workflow.input.user_message}}"
    output:
      reply: "{{result.reply}}"
      end_turn: "{{result.end_turn}}"
      confidence: "{{result.confidence}}"
    control:
      max_turns: 10
      timeout: 30
      strategy: balanced
```

### Step Frequency Control (StepFreq)

Add `step_freq` field in `control` (unit Hz), indicating the node's desired execution frequency.

- For `type: timer` nodes, `step_freq` replaces `delay` for more precise frequency control.
- For loop nodes, `step_freq` controls interval between batches.
- Scripts can dynamically adjust frequency via `ctx.setStepFreq(newFreq)`.

## Integration with Existing PML Features

- **Session & Memory**: Session's implicit context sources can point to `memory` backends (long-term memory) for cross-session knowledge reuse.
- **Session & Markup**: Session state can be written back to markup domains (e.g., updating user preference markup).
- **Turn & Evaluation**: `type: turn` nodes can embed `evaluation` to assess per-turn reply quality and decide early termination.
- **StepFreq & Resource Control**: `step_freq` works with `resources` CPU quota to ensure high-frequency nodes don't exceed limits.
