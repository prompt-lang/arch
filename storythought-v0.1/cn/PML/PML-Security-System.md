## PML安全框架与安全规则


## 模型的安全机制与安全设计框架

### 人为设计的安全机制与测试系统

PML要求在开发和部署阶段内置以下人工安全保障：

- **沙箱测试环境**：所有工作流在正式上线前必须在隔离的测试环境中运行，该环境与生产资源（密钥、物理设备、外部 API）完全隔离。
- **静态分析工具**：提供 `pml security audit` 命令，检查工作流中的潜在风险（如未授权的外部调用、敏感数据明文传输、无限循环风险）。
- **人工审批门禁**：对于涉及高权限操作（如 `actuator` 控制物理设备、`tool` 调用删除接口）的工作流版本变更，需要经过人工审核签名。
- **版本回滚机制**：安全补丁发布后，可快速回退到上一个已知安全的工作流版本。

### 安全评估系统及边界

PML 定义一套安全评估框架，用于量化工作流的风险等级并设置执行边界。

```
security_boundary:
  assets: ["user_data", "robot_arm"]
  allowed_operations: ["read", "infer"]
  forbidden_operations: ["delete", "format"]
```

- **安全边界声明**：在工作流顶层通过 `security_boundary` 字段声明所涉及的资产和危险操作。  
  示例中声明了资产包括 user_data 和 robot_arm，允许的操作有 read 和 infer，禁止的操作包括 delete 和 format。
- **动态安全评分**：NNOS 运行时根据历史执行记录评估工作流的异常行为（如高频失败、越权尝试），低于阈值时自动熔断。
- **安全审计日志**：所有安全相关事件（权限拒绝、敏感数据访问、跨边界调用）记录到不可变审计日志，可输出到外部 SIEM 系统。

## 任务执行的安全准则

### 客观的安全准则

PML 强制执行以下客观安全准则，违反时工作流编译失败或运行时拒绝执行：

- **最小权限原则**：工作流只能访问 `permissions` 中声明的资源（工具、密钥、网络、文件系统）。
- **数据最小化**：节点只能读取其输入声明的数据，不能随意遍历状态中的所有变量。
- **操作可逆性**：对于可产生副作用的节点（如 `actuator` 移动机器人），必须同时定义 `compensate` 补偿节点，以便失败时回滚。
- **超时与熔断**：所有网络调用和外部执行必须设置超时（`control.timeout`），并启用熔断器（`control.circuit_breaker`）。

### 安全风险分类

PML 识别并针对以下安全风险提供防护措施：

| 风险类别 | 风险示例 | PML 防护 |
|----------|----------|----------|
| **提示注入** | 用户输入中包含恶意指令，绕过系统 prompt | Safe Prompts（见第 3 节） |
| **模型逃逸** | 模型输出意外代码或脚本 | 输出 schema 严格校验 + 沙箱执行 |
| **权限提升** | 节点尝试调用未授权的工具 | `permissions` 白名单 |
| **资源耗尽** | 无限循环或超大数据消耗 | `control.loop` 的 `max_iterations` + 资源配额 |
| **数据泄露** | 密钥、用户隐私写入日志 | 自动脱敏 (secrets 不打印) |
| **供应链攻击** | 使用带后门的 Skill 或子工作流 | 依赖完整性哈希校验 (`meta.integrity`) |


## Safe Prompts（防注入）

PML 提供多层机制防止 Prompt Injection 攻击。

### 输入净化

- 对所有用户提供的输入（`workflow.input` 中的字符串字段）自动进行 HTML 实体编码，移除控制字符。
- 在 `config` 中可配置自定义净化规则，例如禁止模板注入模式 `{{.*?}}` 以及限制换行符。

```
config:
  sanitize:
    - pattern: "{{.*?}}"       # 禁止模板注入
      replacement: ""
    - pattern: "[\\n\\r]+"     # 限制换行符
      replacement: " "
```

### 结构化的 Prompt 构建

- 推荐使用 `type: pipeline` 节点组合系统指令和用户输入，将用户输入作为独立片段，不与系统指令拼接在同一字符串中。
- 系统指令部分应设置为 `role: system` 且不可被用户输入覆盖。

### 输出过滤

- 对于 LLM 节点的输出，通过 `output.schema` 强制约束输出格式（如只允许 JSON），拒绝包含可执行代码的响应。
- 可选启用内容安全分类器（如 `control.safety_filter: true`），NNOS 调用外部服务检测有毒内容。

### 指令分隔符

- 在 Prompt 中自动插入不可见分隔符，例如 `[SYS]` 和 `[USER]`，使模型能够区分系统指令和用户输入。NNOS 在渲染 prompt 时自动添加。


## 安全执行环境

PML 要求 NNOS 运行时为所有不可信代码提供隔离的执行环境。

### 脚本沙箱

- `type: script` 和 `type: function` 节点运行在受限沙箱中：
  - 默认禁止文件系统访问、网络连接、子进程创建。
  - 可通过 `permissions` 显式授予额外能力（如 `filesystem: "/data"`）。
  - 内存和 CPU 限制通过 `runtime` 字段配置。
- 推荐使用 Wasm 沙箱（`language: wasm`），提供最严格的隔离。

###  模型执行隔离

- `type: predictor` 和 `type: llm` 节点应运行在独立进程或容器中，防止模型崩溃影响调度器。
- 模型文件加载前需校验完整性哈希（`model.integrity` 字段）。

### 网络隔离

- `type: tool` 或 `type: mcp` 节点只能访问 `permissions.network` 中声明的域名/IP。
- 默认阻止访问内网敏感地址（如 127.0.0.1, 169.254.169.254）。

### 密钥管理

- 所有密钥通过 `secrets` 字段引用环境变量或外部 Vault，禁止明文写在 PML 文件中。
- 日志系统自动将 `$SECRET{...}` 替换为 `[REDACTED]`。

### 执行环境隔离策略

| 节点类型 | 推荐隔离级别 | 说明 |
|----------|--------------|------|
| `llm`, `predictor` | 进程级容器 | 独立进程，可限制 CPU/内存 |
| `script`, `function` | Wasm 或子进程 | 严格沙箱，默认无权限 |
| `tool` | 沙箱 + 网络白名单 | 只能访问许可的 endpoint |
| `neuro_task` (Wasm) | Wasm 沙箱 | 最安全，推荐 |
| `actuator` | 专用驱动代理 | 通过本地 socket 与守护进程通信，不直接暴露硬件 |


## 安全配置示例

一份完整的安全配置示例如下：

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
  filesystem: ["./cache"]            # 只读缓存目录

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

- PML 版本为 1.10
- 元数据中声明工作流名称 secure_workflow，并包含子工作流和模型文件的完整性哈希。
- 权限声明：允许的工具为 web_search 和 calculator，允许的密钥为 openai_key，允许的网络域名为 api.openai.com，允许的文件系统路径为 ./cache（只读）。
- 配置段定义：输入净化规则（去除模板注入和多余换行符）、沙箱默认内存 256MB、默认超时 30 秒、网络禁止、审计日志启用并指定 webhook 地址。
- 工作流输入包含 user_query，并声明安全边界：资产为 user_data，允许操作为 infer。


## 安全评估与审计命令

PML工具链提供以下安全相关命令：

- **`ppcli security audit <workflow.pml>`**：静态分析工作流，输出风险报告和修复建议。
- **`ppcli security test --injection`**：运行 Prompt Injection 测试套件，模拟多种攻击向量。
- **`ppcli security sandbox`**：启动一个临时沙箱，用于测试不可信脚本或模型。
- **`ppcli security logs --since 1h`**：查看最近的审计日志。


## 通过以下机制构建了纵深安全体系：

- **模型安全**：沙箱测试、静态分析、审计日志、版本回滚。
- **任务安全**：最小权限、数据最小化、补偿事务、超时熔断。
- **防注入**：输入净化、结构化 prompt、输出过滤、分隔符注入。
- **执行环境隔离**：沙箱、网络白名单、密钥管理、进程隔离。


