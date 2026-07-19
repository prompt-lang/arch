# PML Storyline Extension

> For LLM-Loop complex task execution scenarios, introduces the **Storyline** container, **Prompt-Buffer**, **Issue Trigger**, and **Buffer State Machine**. These extensions enable Agents to manage complex task execution flows as Issue Lists, pause at key decision points to request user discussions, and form structured human-machine collaboration loops.


## Design Goals

- **Storyline Containerization**: Encapsulate the complete execution process of a complex task as a Storyline, containing goals, phases, buffer pool, and decision history.
- **Prompt-Buffer**: Agent manages pending prompts as an Issue List — ordering, dependencies, status tracking.
- **Issue Trigger**: Agent pauses execution at key decision points, raises structured issues for user discussion, and continues after receiving feedback.
- **Buffer State Machine**: Each Buffer Item follows a deterministic state transition of `pending → in_progress → blocked → resolved/closed`.
- **Human Collaboration Loop**: User discussion input is structured as Issue Responses, allowing the Agent to update strategy and resume execution.


## Core Concepts

| Concept | Description | PML Implementation |
|---------|-------------|-------------------|
| **Storyline** | Complete narrative container for a complex task. Contains goals, phase graph, buffer pool, issue records, decision history. One Inloop Session can contain multiple Storylines; Storylines can be serial or nested. | Top-level `storyline` field + `type: storyline` node |
| **Prompt-Buffer** | The Agent's "to-do list". Each entry is a structured Prompt Issue with title, description, priority, dependencies, status, and assignee (Agent or Human). | `storyline.buffer` + `type: buffer` node |
| **Buffer Item** | A single pending item in the buffer pool. Contains `id`, `title`, `description`, `priority`, `status`, `dependencies`, `assignee`. | `storyline.buffer.items[]` |
| **Issue Trigger** | When the Agent cannot decide autonomously, generates a structured issue, pauses the current Buffer Item, and waits for user input before continuing. Issues can include options, recommendations, and impact analysis. | `type: issue_trigger` node |
| **Issue Response** | User's structured reply to an issue. Contains decision selection, rationale, and constraints. | `type: issue_response` node |
| **Buffer State Machine** | Each Buffer Item's state machine: `pending` → `in_progress` → (`blocked` → `in_progress`) → `resolved` / `closed` | `item.status` enum |


## Syntax & Feature Extensions

### 1. Storyline Top-Level Declaration

```yaml
storyline:
  id: "refactor_user_module"
  title: "Refactor User Module"
  description: "Split User model into Account + Profile, migrate all references while maintaining API compatibility"
  phases:
    - id: "phase_1_analysis"
      title: "Current State Analysis"
      order: 1
      goal: "Map all User model reference points and dependencies"
    - id: "phase_2_design"
      title: "Solution Design"
      order: 2
      goal: "Determine split approach and migration steps"
      requires_human_approval: true
    - id: "phase_3_implementation"
      title: "Implementation"
      order: 3
      goal: "Execute split and migration"
    - id: "phase_4_verification"
      title: "Verification & Cleanup"
      order: 4
      goal: "Test coverage, cleanup old code, update documentation"

  # ── Prompt-Buffer Initial Definition ──
  buffer:
    items:
      - id: "BUF-001"
        title: "Analyze all User model references"
        description: "Search codebase for all locations referencing the User struct, group and count by file"
        priority: 10
        dependencies: []
        assignee: "agent"
        phase: "phase_1_analysis"

      - id: "BUF-002"
        title: "Analyze User-related API endpoints"
        description: "List all APIs exposing the User model, mark public interfaces"
        priority: 9
        dependencies: ["BUF-001"]
        assignee: "agent"
        phase: "phase_1_analysis"

      - id: "BUF-003"
        title: "Determine split approach"
        description: "Based on analysis results, propose Account/Profile split approach, mark impact scope"
        priority: 8
        dependencies: ["BUF-001", "BUF-002"]
        assignee: "agent"
        phase: "phase_2_design"
        requires_issue_trigger: true

      - id: "BUF-004"
        title: "Execute field migration"
        description: "Execute User → Account/Profile field migration per approved plan"
        priority: 7
        dependencies: ["BUF-003"]
        assignee: "agent"
        phase: "phase_3_implementation"

      - id: "BUF-005"
        title: "Update test cases"
        description: "Modify all test cases referencing the User model"
        priority: 6
        dependencies: ["BUF-004"]
        assignee: "agent"
        phase: "phase_4_verification"

  # ── Issue Records ──
  issues:
    - id: "ISS-001"
      buffer_item: "BUF-003"
      title: "Split approach requires confirmation"
      description: "Two approaches: 1) Aggressive split (immediately remove User) 2) Gradual split (keep User alias for 3 months)"
      options:
        - id: "aggressive"
          label: "Aggressive Split"
          impact: "High risk, but thorough code cleanup"
        - id: "gradual"
          label: "Gradual Split"
          impact: "Low risk, requires longer maintenance window"
      recommendation: "gradual"
      status: "open"

  # ── Decision History ──
  decisions: []
```

### 2. Prompt-Buffer State Machine

Each Buffer Item follows a **5-state deterministic transition**:

```
                    ┌──────────┐
                    │ pending  │  ← Initial state
                    └────┬─────┘
                         │ Agent begins processing
                         ▼
                    ┌──────────┐
              ┌────→│in_progress│
              │     └────┬─────┘
              │          │ Blocked (needs human decision / dependency unresolved)
              │          ▼
              │     ┌──────────┐
              │     │ blocked  │  ← Awaiting external input
              │     └────┬─────┘
              │          │ Issue Trigger receives user response
              │          │ or dependency completed
              │          ▼
              │     ┌──────────┐
              │     │in_progress│ (resumed)
              │     └────┬─────┘
              │          │ Task complete
              │          ▼
              │     ┌──────────┐     ┌──────────┐
              └─────│ resolved │     │  closed  │  ← Cancelled / no longer needed
                    └──────────┘     └──────────┘
```

Transition rules:
- `pending → in_progress`: Agent begins processing (automatic or triggered by dispatcher node)
- `in_progress → blocked`: Issue Trigger activated (needs human decision) / dependencies not yet resolved
- `blocked → in_progress`: Received Issue Response / dependencies resolved
- `in_progress → resolved`: Task complete, result written to buffer item's `resolution` field
- `in_progress → closed`: Task cancelled or no longer needed

### 3. Buffer Node Type

#### `type: buffer` — Buffer Pool Management

```yaml
pnodes:
  # Query buffer pool status
  - id: get_pending_items
    type: buffer
    operation: query
    storyline: "refactor_user_module"
    query:
      status: ["pending", "blocked"]
      priority_min: 5
      phase: "phase_1_analysis"
      dependencies_resolved: true
    order_by: "priority desc"
    limit: 5
    output:
      schema:
        items: { type: array }

  # Update buffer item status
  - id: start_item
    type: buffer
    operation: update
    storyline: "refactor_user_module"
    item_id: "BUF-001"
    updates:
      status: "in_progress"
      started_at: "{{workflow.now}}"
    control:
      validate_transition: true

  # Complete buffer item
  - id: resolve_item
    type: buffer
    operation: update
    storyline: "refactor_user_module"
    item_id: "BUF-001"
    updates:
      status: "resolved"
      resolution:
        summary: "Found 47 User references across 12 files"
        artifact_ref: "{{analysis_result.output.report}}"
      completed_at: "{{workflow.now}}"

  # Dynamically create new buffer item (Agent discovers new task)
  - id: add_new_item
    type: buffer
    operation: create
    storyline: "refactor_user_module"
    item:
      title: "Fix discovered circular dependency"
      description: "Circular reference found between account.go and profile.go during migration"
      priority: 10
      dependencies: ["BUF-004"]
      assignee: "agent"
      phase: "phase_3_implementation"
```

### 4. Issue Trigger

#### `type: issue_trigger` — Pause and Request Human Decision

```yaml
pnodes:
  - id: request_split_decision
    type: issue_trigger
    storyline: "refactor_user_module"
    buffer_item: "BUF-003"

    issue:
      title: "User Model Split Approach Confirmation"
      summary: |
        Current state analysis complete (BUF-001, BUF-002). Found User model referenced in 47 locations.

        Two split approaches available, please confirm selection:

        **Option A: Aggressive Split**
        - Immediately create Account + Profile to replace User
        - Remove all User references in one pass
        - Risk: High (may miss hidden references)
        - Timeline: 2-3 days

        **Option B: Gradual Split**
        - Create Account + Profile first
        - Keep User as type alias, marked deprecated
        - Remove alias after 3 months
        - Risk: Low (backward compatible)
        - Timeline: 5-7 days (including transition maintenance)

      options:
        - id: "aggressive"
          label: "Option A: Aggressive Split"
          risk: "high"
        - id: "gradual"
          label: "Option B: Gradual Split (Recommended)"
          risk: "low"
          recommended: true

      recommendation:
        option: "gradual"
        rationale: "Given User model's widespread references including third-party integration code, gradual split reduces regression risk."
        confidence: 0.85

      required_context:
        - question: "Is there an upcoming release that must not be impacted?"
          type: "boolean"
        - question: "Is the 3-month transition period for the gradual approach acceptable?"
          type: "text"

    blocking:
      mode: "pause_storyline"
      timeout_hours: 72
      on_timeout: "remind"

    output:
      schema:
        issue_id: { type: string }
        status: { type: string }
```

#### `type: issue_response` — Receive Human Decision

```yaml
  - id: receive_decision
    type: issue_response
    storyline: "refactor_user_module"
    issue_id: "ISS-001"

    response:
      selected_option: "gradual"
      rationale: "v2.3 release next week, aggressive approach too risky. Gradual 3-month transition acceptable."
      additional_context:
        next_release_date: "2026-08-01"
        accept_transition_period: true
      constraints:
        - "Must not impact v2.3 release"
        - "API compatibility must not break"

    on_resolve:
      resume_buffer_item: true
      update_storyline: true
      notify_phase: "phase_3_implementation"
```

### 5. Storyline Recovery & Context Injection

When a Buffer Item resumes from `blocked` to `in_progress`, NNOS automatically constructs recovery context:

```yaml
pnodes:
  - id: resume_work
    type: llm
    storyline: "refactor_user_module"
    buffer_item: "BUF-003"

    context:
      explicit:
        user: "Continue executing BUF-003 per confirmed approach"

      implicit_sources:
        - type: buffer_context
          item_id: "BUF-003"
          include:
            - original_description
            - previous_attempts
            - issue_history

        - type: dependency_context
          dependencies_of: "BUF-003"
          include_resolutions: true

        - type: phase_context
          phase: "phase_2_design"

        - type: decision_context
          related_to: "BUF-003"

    process:
      prompt:
        user: "{{workflow.input.user_instruction}}"
```

### 6. Complete Example: Complex Task Execution Flow

```yaml
pml:
  version: "1.16"

storyline:
  id: "refactor_user_module"
  title: "Refactor User Module"
  phases:
    - { id: "phase_1_analysis", title: "Current State Analysis", order: 1 }
    - { id: "phase_2_design", title: "Solution Design", order: 2, requires_human_approval: true }
    - { id: "phase_3_implementation", title: "Implementation", order: 3 }
    - { id: "phase_4_verification", title: "Verification & Cleanup", order: 4 }

  buffer:
    items:
      - { id: "BUF-001", title: "Analyze User references", priority: 10, phase: "phase_1_analysis" }
      - { id: "BUF-002", title: "Analyze API endpoints", priority: 9, phase: "phase_1_analysis", dependencies: ["BUF-001"] }
      - { id: "BUF-003", title: "Determine split approach", priority: 8, phase: "phase_2_design", dependencies: ["BUF-001","BUF-002"], requires_issue_trigger: true }
      - { id: "BUF-004", title: "Execute field migration", priority: 7, phase: "phase_3_implementation", dependencies: ["BUF-003"] }
      - { id: "BUF-005", title: "Update tests", priority: 6, phase: "phase_4_verification", dependencies: ["BUF-004"] }

workflow:
  input:
    user_instruction: string
  nodes: [dispatcher, execute_item, check_blocked, trigger_issue, receive_response]
  edges:
    - dispatcher -> execute_item
    - execute_item -> check_blocked
    - check_blocked -> trigger_issue
    - trigger_issue -> receive_response
    - receive_response -> dispatcher

pnodes:
  - id: dispatcher
    type: buffer
    operation: query
    storyline: "refactor_user_module"
    query:
      status: "pending"
      dependencies_resolved: true
    order_by: "priority desc"
    limit: 1
    output:
      next_item: "{{result.items[0]}}"

  - id: execute_item
    type: llm
    control:
      loop:
        condition: "{{output.status == 'in_progress'}}"
    process:
      prompt:
        system: |
          Current phase: {{storyline.current_phase.title}}
          Current task: {{dispatcher.output.next_item.title}}
          Task description: {{dispatcher.output.next_item.description}}
          Dependency results: {{dependency_context}}

  - id: check_blocked
    type: script
    script: |
      async function run(input, ctx) {
          let item = ctx.state.get("dispatcher.output.next_item");
          let needsDecision = ctx.state.get("execute_item.output.needs_human_decision");
          if (needsDecision || item.requires_issue_trigger) {
              return { status: "blocked", reason: "needs_human_decision" };
          }
          await ctx.storyline.buffer.update(item.id, { status: "resolved" });
          return { status: "resolved" };
      }

  - id: trigger_issue
    type: issue_trigger
    condition: "{{check_blocked.output.status == 'blocked'}}"
    storyline: "refactor_user_module"
    buffer_item: "{{dispatcher.output.next_item.id}}"
    issue:
      title: "{{execute_item.output.decision_title}}"
      summary: "{{execute_item.output.decision_summary}}"
      options: "{{execute_item.output.decision_options}}"
      recommendation: "{{execute_item.output.recommendation}}"
    blocking:
      mode: "pause_buffer_item"
      timeout_hours: 48

  - id: receive_response
    type: issue_response
    storyline: "refactor_user_module"
    issue_id: "{{trigger_issue.output.issue_id}}"
    response:
      selected_option: "{{workflow.input.user_instruction}}"
    on_resolve:
      resume_buffer_item: true
```

### 7. Integration with Existing PML Features

| Integration Point | Approach | Value |
|-------------------|----------|-------|
| **Session / Inloop** | Storyline runs within an Inloop Session, sharing session context | Long-running tasks do not lose state |
| **PEM Engineering Memory** | `storyline.decisions` auto-written to `engineering_kb` | Cross-session decision knowledge accumulation |
| **Markdown Prompt Cards** | Issue Trigger can query `prompt_cards` domain for relevant conventions | Auto-inject project conventions during decisions |
| **SOF Self Ontology** | Agent queries `self.cognitive.planning_horizon` to determine Buffer chunk granularity | Adaptive task decomposition |
| **RBE Rule Engine** | Buffer scheduling policies can be implemented with `type: rule` nodes | Declarative scheduling rules |
| **Checkpoint** | Storyline full state can be checkpointed for persistence | Seamless continuation after failure recovery |

### 8. Design Principles

1. **Storyline is the "narrative container" of a task** — not a simple TODO list, but a complete story containing phases, decisions, and context evolution.
2. **Buffer is the Agent's "working memory"** — structured, reorderable, dependency-aware, stateful Issue List.
3. **Issue Trigger is the "human-machine handshake point"** — pause at Agent capability boundaries, structurally request human wisdom.
4. **State machine-driven, not polling-driven** — each Buffer Item's state transition is deterministic, auditable, and rollback-capable.
5. **Deep integration with Session/PEM/Markup** — not building isolated islands, reusing all existing infrastructure.
