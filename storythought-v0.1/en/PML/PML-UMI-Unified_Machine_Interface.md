# PML UMI — Universe Machine Interface

## Interaction Model

The interaction model defines the data interfaces and access methods connecting AGI agents with their environments, including programmatic interactions.

### A-A-Human

    Agent(Task) ↔ Agent(Delegate) ↔ Human

The Delegate Agent's primary function is human-computer interaction, or interaction with external systems, processing interaction data, etc. The Task Agent focuses more on task-related information and strategy execution.

### Self-Loop

    Agent(time i) → Agent(time i+n)

At time i, the Agent preserves prompt information to be used at time i+n.

- Use an external processor to remember and trigger this prompt.
- Use a step-by-step approach where each processing step of the Agent evaluates whether this prompt needs to be processed.

### Multi-Loop Task

The Agent completes a task through multiple output executions.

- The Task is too complex to solve in one round; complete it through multiple rounds.
- The Task is ongoing (e.g., a game), continuously producing action outputs throughout the period to complete the cycle's tasks.

### Seek-for-Prompt

The Agent acquires more Prompts to obtain additional information.

### Prompt Rank

When receiving multiple prompts, prompts can be sorted or processed according to priority.

### Realtime Prompt

Prompt signals connected as real-time interactive feedback.

### Session

See: [Session Extension](./PML-Session-Extension.md)

### Prompt Modeling

### Prompt Canvas

Prompt views and view-based prompts.

## AGI Input & Output Modeling

Unified modeling techniques can be used for inputs — for example, different tokenizers implement different input modeling processing. The AGI main model possesses full-spectrum input processing capability. Modeling for inputs is Input Modeling.

For non-text outputs, such as action control domain outputs and speech generation, modeling is Output Modeling.

## UMC: Universe Machine Connector

UMC is the implementation of the inter-agent UMI communication protocol, or the middleware interaction system between agents and neural computing nodes.

## Context Modeling

See: [Session Extension](./PML-Session-Extension.md)

## Prompt-lang/UMI

Use Prompt-lang to define and implement UMI.


## Appendix

### Reference Description Formats for Simulation

[MuJoCo Modeling](https://mujoco.readthedocs.io/en/latest/modeling.html)

[USD](https://developer.nvidia.com/usd)

[URDF](http://wiki.ros.org/urdf)


### References

[Prompt Chain](https://www.promptingguide.ai/zh/techniques/prompt_chaining)

[RAG](https://ai.meta.com/blog/retrieval-augmented-generation-streamlining-the-creation-of-intelligent-natural-language-processing-models/)

[Anthropic MCP](https://www.anthropic.com/news/model-context-protocol)

[Google A2A](https://github.com/google/A2A)

[OpenAI Swarm](https://github.com/openai/swarm)

[APPL](https://github.com/appl-team/appl)

[PromptWizard](https://arxiv.org/abs/2405.18369)

[DSPy](https://github.com/stanfordnlp/dspy)

[RAGFlow](https://github.com/infiniflow/ragflow)
