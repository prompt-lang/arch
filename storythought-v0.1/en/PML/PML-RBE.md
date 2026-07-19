# PML_RBE Design Specification: PNode RuleBased Engine

## 1. Overview

### 1.1 Background & Motivation

In complex agent workflows, a large amount of business logic exists in "condition → action" patterns. If such rule-type logic is hardcoded one-by-one through `type: script` nodes, rules become difficult to maintain, cannot be hot-reloaded, and lack a unified rule execution model. To fill this gap, this document proposes PML_RBE (RuleBased PNode) — a rule-driven PNode type that enables PML workflows to define, manage, and execute rules declaratively.

### 1.2 Design Goals
- **Declarative Rule Definition**: Users only describe "condition → action" without writing execution flow code
- **Dual-Mode Implementation**: Rules can be inline PML Scripts or external executable system programs
- **Seamless PNode Integration**: As a new PNode type (`type: rule`), maintaining interoperability with existing workflow, memory, and environment systems
- **Hot-Reload Support**: Rules can be dynamically loaded and updated at runtime via PNode Protocol or executor interfaces
- **High-Performance Execution**: Pattern sharing and caching mechanisms in condition matching phase to avoid redundant evaluation

## 2. Core Concepts

### 2.1 Rule Engine Fundamentals
The rule engine follows the "condition matching → action execution" paradigm. Core components:
- **Rule**: Declarative statement composed of conditions and actions, mapping to each entry in the `rules` field
- **Condition**: true/false boolean expression containing one or more predicates on facts, corresponding to the `condition` field
- **Action**: Operation executed when condition is met, corresponding to the `action` field
- **Fact**: Input data for rule evaluation, i.e., workflow state bound to the `input` field
- **RuleSet**: Collection of related rules, i.e., the overall `rules` array
- **Agenda**: Priority-sorted queue of pending actions, managed internally by the engine

### 2.2 ECA Pattern
Industry-standard rule systems widely adopt the Event-Condition-Action (ECA) pattern. PML_RBE inherits this pattern:
- **Event**: Implicitly defined by upstream node outputs or external system events in PML workflows, declared via the `trigger` field
- **Condition**: Declarative predicate expressions supporting logical combinations (AND/OR)
- **Action**: Executable action body (PML Script, system command, HTTP call, etc.)

### 2.3 PML_RBE vs. Existing PNodes
Compared to regular PNodes (e.g., `script`, `llm`), RuleBased PNode (`type: rule`) has a multi-rule parallel matching, agenda-driven execution model rather than single-path one-shot execution; its reasoning capability is limited to deterministic condition matching but achieves complete separation of logic and engine, supports hot-reloading, and allows priority and conflict resolution strategies between rules.

## 3. PML_RBE Syntax Specification

### 3.1 Top-Level Structure
Rule nodes are defined in the `pnodes` list with node type `rule`. Core fields include:
- `id`, `description`: Node identifier and description
- `implementation`: `pml_script` or `system_program`, specifying rule implementation mode
- `trigger`: Trigger mode, can include `mode` (`on_input`, `on_event`, or `periodic`), with corresponding `event_filter` or `interval_sec`
- `rules`: Rule list, each rule containing `name`, `priority` (integer, larger = higher priority), `condition` (condition expression), and `action` (action definition, type can be `execute`, `set_var`, `emit_event`, `call_node`, etc.)
- `conflict_resolution`: Conflict resolution strategy, options: `first_match`, `priority`, or `all_match`
- `control`: Control parameters including `timeout` (seconds), `max_rules_per_run`, `fallback_action`

### 3.2 Implementation Mode 1: PML Script Inline
When `implementation` is `pml_script`, rule action bodies are defined by inline PML Script. For example, a credit approval rule engine:

Node id `credit_evaluator`, implementation `pml_script`, trigger `on_input`. Rule set contains three rules:
- Rule `high_value_approve`: priority 100, condition `input.credit_score > 750 and input.income > 50000`, action sets workflow variable `approval_result`, status `approved`, amount = 90% of requested.
- Rule `medium_review`: priority 50, condition credit score 600-750, action sets status `manual_review`.
- Rule `low_deny`: priority 10, condition credit score ≤ 600, action status `denied`.

Conflict resolution `first_match`, timeout 5s, fallback returns `error` status.

### 3.3 Implementation Mode 2: Executable System Program
When `implementation` is `system_program`, the rule engine is hosted by an external executable, interacting with PML via standard I/O. For example, a security threat detection engine with communication protocol `stdio`, `http`, or `grpc`, sandbox-enabled with memory and CPU limits.

#### System Program Interface Specification
External programs interact with the PML runtime via JSON-Lines protocol. Request format includes `request_id`, operation type `evaluate`, and `facts` object. Response format returns `status`, `matched_rules` array, and execution time. Management commands like `reload_rules` for hot-reloading and `status` for engine state queries are also supported.

## 4. Condition Expression Specification

### 4.1 Basic Syntax
Condition expressions use SQL-like declarative syntax supporting operators: comparison (`==`, `!=`, `>`), logical (`and`, `or`, `not`), set (`in`, `not in`), null checks (`is null`, `is not null`), regex matching (`matches`), range (`between`), and existence checks (`exists`).

### 4.2 Compound Conditions
Conditions can be combined via parentheses and logical operators.

### 4.3 Context Functions
Conditions can reference PML context functions to access workflow state, environment markup, and memory data:
- `ctx.var("<path>")`: Read workflow variables
- `ctx.markup("<domain>", "<query>")`: Query environment markup
- `ctx.memory("<backend>", "<key>")`: Read memory data

## 5. Rule Execution Model

### 5.1 Execution Flow
Four phases:
1. **Input Phase**: PNode receives input data from upstream nodes or external systems, constructs fact set
2. **Condition Matching Phase**: Iterate each rule in the rule set, evaluate its condition expression; shared conditions evaluated once and cached for efficiency
3. **Agenda Construction Phase**: Sort matched rules by priority according to conflict_resolution strategy
4. **Action Execution Phase**: Execute actions in agenda order; actions may modify workflow state, trigger events, or invoke other nodes; if no rule matches, execute fallback_action

### 5.2 Conflict Resolution Strategies
- `first_match`: Execute only the first matching rule
- `priority`: Sort by priority, execute all matching rules (higher priority first)
- `all_match`: Execute all matching rules, no specific order

### 5.3 Workflow DAG Integration
`type: rule` nodes are embedded in the PML workflow DAG like regular PNodes. Rule evaluation nodes can reference upstream outputs, and subsequent switch nodes can branch based on rule execution results.

## 6. Rule Management

### 6.1 RuleSet References
Large rule sets can be extracted as independent files, referenced via the `ruleset` field pointing to an external YAML file path.

### 6.2 Hot-Reload Mechanism
Through PNode Protocol's update operation, rule sets can be dynamically replaced at runtime by specifying node ID and new rule set path. For system program implementations, `reload_rules` commands can also trigger hot-reloading.

## 7. PML System Integration

### 7.1 Executor Meta-Model Relationship
In top-level `executors` declarations, rule engines can be registered as executor backends describing their algorithm (e.g., Rete), supported operators, max rule capacity, and average matching latency.

### 7.2 Memory System Integration
Rule engine results can be persisted to PML memory backends, supporting cross-workflow rule execution auditing.

### 7.3 Markup Environment Integration
Rule conditions can directly query environment markup for perception-enabled rules. For example, obstacle avoidance rule conditions can query world map markup for nearby obstacles.

### 7.4 Agent Hierarchy Integration
`type: rule` nodes can bind to specific hierarchy-level agents via `required_capability` field (e.g., `local` planner) for low-latency execution.

## 8. Implementation Notes

1. **Condition Caching**: Cross-rule shared condition expressions should be evaluated once, results cached in working memory
2. **Sandbox Isolation**: Both inline scripts and external programs should run in independent sandboxes to prevent impact on the main scheduler
3. **Timeout Control**: Rule set execution must have timeout limits; on timeout, trigger on_error strategy
4. **Protocol Security**: Communication endpoints in system program mode should support authentication
5. **State Persistence**: Rule engine execution state should be serializable to checkpoint for failure recovery
6. **Observability**: Output rule match count, execution duration, per-rule hit rates for rule set optimization
