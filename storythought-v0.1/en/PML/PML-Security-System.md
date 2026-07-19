## PML Security Framework & Security Rules


## Model Security Mechanisms & Security Design Framework

### Human-Designed Security Mechanisms & Testing Systems

PML requires the following human-designed security safeguards to be built-in during development and deployment:

- **Sandbox Test Environment**: All workflows must run in an isolated test environment before production deployment, completely isolated from production resources (secrets, physical devices, external APIs).
- **Static Analysis Tools**: Provides `pml security audit` command to inspect workflows for potential risks (e.g., unauthorized external calls, plaintext sensitive data transmission, infinite loop risks).
- **Human Approval Gate**: Workflow version changes involving high-privilege operations (e.g., `actuator` controlling physical devices, `tool` calling delete interfaces) require human review and signature.
- **Version Rollback Mechanism**: After security patches are released, can quickly roll back to the last known secure workflow version.

### Security Assessment System & Boundaries

PML defines a security assessment framework for quantifying workflow risk levels and setting execution boundaries.

```
security_boundary:
  assets: ["user_data", "robot_arm"]
  allowed_operations: ["read", "infer"]
  forbidden_operations: ["delete", "format"]
```

- **Security Boundary Declaration**: Declare involved assets and dangerous operations at the workflow top level via the `security_boundary` field.  
  The example declares assets including user_data and robot_arm, allowed operations including read and infer, and forbidden operations including delete and format.
- **Dynamic Security Scoring**: NNOS runtime evaluates workflow anomalous behavior (e.g., high-frequency failures, privilege escalation attempts) based on historical execution records, automatically circuit-breaking when below threshold.
- **Security Audit Logs**: All security-related events (permission denials, sensitive data access, cross-boundary calls) are recorded to immutable audit logs, exportable to external SIEM systems.

## Task Execution Security Guidelines

### Objective Security Guidelines

PML enforces the following objective security guidelines — violations cause workflow compilation failure or runtime execution refusal:

- **Principle of Least Privilege**: Workflows can only access resources declared in `permissions` (tools, secrets, network, filesystem).
- **Data Minimization**: Nodes can only read data declared in their input bindings, cannot arbitrarily traverse all state variables.
- **Operation Reversibility**: For nodes that produce side effects (e.g., `actuator` moving a robot), a `compensate` compensation node must be defined for rollback on failure.
- **Timeout & Circuit Breaking**: All network calls and external executions must set timeouts (`control.timeout`) and enable circuit breakers (`control.circuit_breaker`).

### Security Risk Classification

PML identifies and provides protections against the following security risks:

| Risk Category | Risk Example | PML Protection |
|---------------|--------------|----------------|
| **Prompt Injection** | User input contains malicious instructions bypassing system prompt | Safe Prompts (see Section 3) |
| **Model Escape** | Model outputs unexpected code or scripts | Output schema strict validation + sandbox execution |
| **Privilege Escalation** | Node attempts to invoke unauthorized tools | `permissions` whitelist |
| **Resource Exhaustion** | Infinite loops or excessive data consumption | `control.loop` `max_iterations` + resource quotas |
| **Data Leakage** | Secrets, user privacy written to logs | Automatic redaction (secrets not printed) |
| **Supply Chain Attack** | Using backdoored Skills or sub-workflows | Dependency integrity hash verification (`meta.integrity`) |


## Safe Prompts (Injection Prevention)

PML provides multiple layers of mechanism to prevent Prompt Injection attacks.

### Input Sanitization

- All user-provided inputs (string fields in `workflow.input`) are automatically HTML-entity encoded and control characters removed.
- Custom sanitization rules can be configured in `config`, e.g., blocking template injection patterns `{{.*?}}` and limiting newlines.

```
config:
  sanitize:
    - pattern: "{{.*?}}"       # Block template injection
      replacement: ""
    - pattern: "[\\n\\r]+"     # Limit newlines
      replacement: " "
```

### Structured Prompt Construction

- Recommend using `type: pipeline` nodes to compose system instructions and user input, treating user input as an independent fragment not concatenated with system instructions in the same string.
- System instruction portions should be set to `role: system` and non-overridable by user input.

### Output Filtering

- For LLM node outputs, enforce output format constraints via `output.schema` (e.g., JSON only), rejecting responses containing executable code.
- Optionally enable content safety classifier (e.g., `control.safety_filter: true`), NNOS calls external service to detect toxic content.

### Instruction Delimiters

- Automatically insert invisible delimiters in prompts, e.g., `[SYS]` and `[USER]`, enabling the model to distinguish system instructions from user input. NNOS automatically adds these when rendering prompts.


## Secure Execution Environment

PML requires NNOS runtime to provide isolated execution environments for all untrusted code.

### Script Sandbox

- `type: script` and `type: function` nodes run in restricted sandboxes:
  - Filesystem access, network connections, and child process creation are denied by default.
  - Additional capabilities can be explicitly granted via `permissions` (e.g., `filesystem: "/data"`).
  - Memory and CPU limits are configured via the `runtime` field.
- Wasm sandbox is recommended (`language: wasm`), providing the strictest isolation.

### Model Execution Isolation

- `type: predictor` and `type: llm` nodes should run in independent processes or containers, preventing model crashes from affecting the scheduler.
- Model file integrity hashes must be verified before loading (`model.integrity` field).

### Network Isolation

- `type: tool` or `type: mcp` nodes can only access domains/IPs declared in `permissions.network`.
- Access to sensitive internal addresses (e.g., 127.0.0.1, 169.254.169.254) is blocked by default.

### Secret Management

- All secrets reference environment variables or external Vault via the `secrets` field, never written in plaintext in PML files.
- The logging system automatically replaces `$SECRET{...}` with `[REDACTED]`.

### Execution Environment Isolation Policies

| Node Type | Recommended Isolation Level | Description |
|-----------|----------------------------|-------------|
| `llm`, `predictor` | Process-level container | Independent process, CPU/memory limits |
| `script`, `function` | Wasm or subprocess | Strict sandbox, no permissions by default |
| `tool` | Sandbox + network whitelist | Only permitted endpoints accessible |
| `neuro_task` (Wasm) | Wasm sandbox | Most secure, recommended |
| `actuator` | Dedicated driver proxy | Communicates via local socket with daemon, no direct hardware exposure |


## Security Configuration Example

A complete security configuration example:

```
pml:
  version: "1.10"

meta:
  name: "secure_workflow"
  integrity:
    subworkflows:
      "validate.pml": "sha256:abc123..."
    models:
      "yolov8n.onnx": "sha256:xyz789..."

permissions:
  tools: ["web_search", "calculator"]
  secrets: ["openai_key"]
  network: ["api.openai.com"]
  filesystem: ["./cache"]            # Read-only cache directory

config:
  sanitize:
    - pattern: "{{.*?}}"
      replacement: ""
  sandbox:
    default_memory_mb: 256
    default_timeout_sec: 30
    network: false
  audit:
    enabled: true
    webhook: "https://audit.example.com/event"

workflow:
  input: { user_query: string }
  # ... nodes ...
  security_boundary:
    assets: ["user_data"]
    allowed_operations: ["infer"]
```

- PML version is 1.10
- Metadata declares workflow name secure_workflow, includes integrity hashes for sub-workflows and model files.
- Permissions declare: allowed tools web_search and calculator, allowed secret openai_key, allowed network domain api.openai.com, allowed filesystem path ./cache (read-only).
- Config section defines: input sanitization rules (remove template injection and excessive newlines), sandbox default memory 256MB, default timeout 30s, network disabled, audit logging enabled with webhook address.
- Workflow input includes user_query, declares security boundary: asset user_data, allowed operation infer.


## Security Assessment & Audit Commands

PML toolchain provides the following security-related commands:

- **`ppcli security audit <workflow.pml>`**: Statically analyze workflow, output risk report and remediation suggestions.
- **`ppcli security test --injection`**: Run Prompt Injection test suite, simulating multiple attack vectors.
- **`ppcli security sandbox`**: Launch a temporary sandbox for testing untrusted scripts or models.
- **`ppcli security logs --since 1h`**: View recent audit logs.


## Defense-in-Depth Security System

Built through the following mechanisms:

- **Model Security**: Sandbox testing, static analysis, audit logs, version rollback.
- **Task Security**: Least privilege, data minimization, compensation transactions, timeout circuit-breaking.
- **Injection Prevention**: Input sanitization, structured prompts, output filtering, delimiter injection.
- **Execution Environment Isolation**: Sandbox, network whitelist, secret management, process isolation.

