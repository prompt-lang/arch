# PML：权限系统完善与架构角色增强

> 针对架构角色设计进行全面完善。新增**动态角色切换**、**角色与能力范围对齐**、**Master 角色再分层**、**跨工作流角色传播**、**角色委派**、**角色冲突策略**、**细粒度记忆/标记访问控制**、**Master 通道强化**以及**最小权限静态分析工具**。这些改进使 PML 的权限系统达到企业级身份与访问管理标准，并更好地支撑多租户、多层级智能体架构。


## 设计目标

- **动态权限调整**：支持在运行时临时提升或降低角色，满足维护与应急场景。
- **能力对齐**：将角色权限与智能体规划层级（`capability_scope`）关联，避免越权调用。
- **管理角色细分**：区分管理员、审计员、操作员，实现职责分离。
- **跨边界安全**：子工作流角色继承与映射，委派权限给其它 Agent，安全的多角色并集策略。
- **细粒度资源控制**：对记忆分层（tier）和标记域（domain）进行独立权限设置。
- **强认证 Master 通道**：防止 Master 指令伪造，提供 mTLS/JWT 等认证方式。
- **辅助工具**：`pml security audit` 自动生成最小权限建议，`pml security minify-roles` 裁剪过度授权。


## 语法与功能扩展

### 动态角色切换

新增运行时 API `assumeRole(role, duration)`，允许当前工作流在有限时间内切换角色。


- 仅当当前角色的 `allow_assume` 权限为 `true` 时可用（默认仅 `admin` 或 `master` 角色有此权限）。
- 切换后的角色权限必须小于等于原角色的权限（不可越权）。
- 所有切换记录写入审计日志。

在角色定义中可增加 `allow_assume` 字段，例如 `admin` 角色允许切换，`operator` 角色不允许。

```
roles:
  - name: admin
    allow_assume: true
  - name: operator
    allow_assume: false
```


在脚本中，当遇到紧急情况时，可以通过 `await ctx.assumeRole("emergency", 300)` 临时提升权限 5 分钟，执行高特权操作，之后自动恢复原角色。

### 角色与能力范围对齐

在角色权限中增加 `allowed_capability_levels` 字段，限制角色可调用的智能体规划层级（`level`）。

```
roles:
  - name: viewer
    allowed_capability_levels: ["local"]
  - name: operator
    allowed_capability_levels: ["local", "global"]
```

例如，`viewer` 角色只允许使用 `local` 层级的规划器，`operator` 角色可以使用 `local` 和 `global` 层级。

在 `type: planning` 节点中，运行时检查当前角色是否允许使用要求的 `capability_level`。

### Master 角色再分层

预定义三个管理角色，取代单一的 `master` 角色：

- `admin`：系统管理员，可修改工作流配置、管理角色、执行 Master Prompt。
- `auditor`：审计员，只能查看审计日志和指标，不能执行任何写操作。
- `operator`：操作员，可暂停/恢复工作流、重新加载配置，但无权修改权限。

```
roles:
  - name: admin
    permissions: { all: true }
    allow_assume: true
  - name: auditor
    permissions: { audit_read: true }
  - name: operator
    permissions:
      control: ["pause", "resume", "reload"]
```

例如，`admin` 角色拥有所有权限且允许切换；`auditor` 只有 `audit_read` 权限；`operator` 拥有控制操作（暂停、恢复、重载）以及特定工具和记忆的权限。

Master Prompt 通道仅对 `admin` 角色开放。

### 跨工作流角色传播

`subworkflow` 节点新增 `inherit_role` 和 `role_mapping` 字段。

- `inherit_role: true`：子工作流使用父工作流的角色。
- `role_mapping`：将父角色映射到子工作流中的另一角色，例如 `operator -> viewer`。

```
pnodes:
  - id: call_child
    type: subworkflow
    workflow: "./child.pml"
    inherit_role: false
    role_mapping:
      operator: viewer
      admin: admin
```

例如，调用子工作流时，可以设置 `inherit_role: false`，并定义映射关系（如 operator 映射为 viewer，admin 映射为 admin）。缺省情况下，子工作流以最小权限角色（`viewer`）执行，除非显式映射或继承。

### 角色委派（Delegation）

允许当前角色临时将自身权限的子集授予另一个 Agent。

- 通过 `ctx.delegateRole(targetAgentId, permissions, ttl)` 创建委派令牌。
- 接收方 Agent 在执行需要特权的节点时，可附带令牌证明权限。
- 所有委派操作记录审计。
```
let token = await ctx.delegateRole("worker_bot", {
    tools: ["web_search"],
    memory: { tier: "fast", operations: ["read"] }
}, 600);
await ctx.callAgent("worker_bot", { task: "search", delegationToken: token });
```

例如，主脚本可以创建一个委派令牌，允许 `worker_bot` 在 600 秒内使用 `web_search` 工具以及读取 `fast` 记忆层，然后调用该 Agent 执行任务并附上令牌。

### 角色冲突策略

当一个调用者具有多个角色时（如通过身份系统返回角色列表），可配置策略合并权限。

- 在 `config` 中设置 `role_combining: union`（默认）或 `intersection`。
- `union`：取所有角色权限的并集。
- `intersection`：取交集（最严格）。

```
config:
  role_combining: union
```

例如，配置 `role_combining: union` 表示权限合并采用并集模式。

### 细粒度记忆与标记访问控制

在 `permissions` 中扩展 `memory` 和 `markup` 子项。

- 按记忆 `tier` 和操作类型限制。
- 按标记域 `domain` 和操作限制。

```
permissions:
  memory:
    - tier: "fast"
      operations: ["read", "write"]
    - tier: "long"
      operations: ["read"]
  markup:
    - domain: "public_map"
      operations: ["read"]
    - domain: "private_agent_data"
      operations: []
```

例如，权限声明中可以定义：对 `fast` 记忆层允许读写，对 `long` 记忆层只允许读；对 `public_map` 标记域允许读，对 `private_agent_data` 禁止任何操作。

### Master 通道强化

在 `config` 中增加 `master_auth` 配置，要求 Master Prompt 必须通过认证通道发送。

- 支持 `method: mtls`、`method: jwt`。
- 普通输入中的 `is_master` 字段将被忽略，只有通过认证连接的消息才被视为 Master 指令。

例如，配置 Master 认证使用 mTLS，并指定证书和 CA 文件路径。

```
config:
  master_auth:
    method: "mtls"
    cert_file: "/etc/pml/master.crt"
    ca_file: "/etc/pml/ca.crt"
```

### 最小权限静态分析

增强 CLI 子命令：

- `ppcli security audit --suggest-roles`：输出工作流实际使用的资源，并建议最小角色权限清单。
- `ppcli security minify-roles --out roles.yaml`：根据工作流分析结果生成裁剪后的角色定义。

---

## 完整示例：多角色、委派与 Master 控制

```
pml:
  version: "1.14"

roles:
  - name: admin
    permissions: { all: true }
    allow_assume: true
  - name: auditor
    permissions: { audit_read: true }
  - name: operator
    permissions:
      control: ["pause", "resume"]
      tools: ["search"]
      memory: [{ tier: "fast", operations: ["read", "write"] }]

config:
  role_combining: union
  master_auth:
    method: "jwt"
    public_key: "/secure/jwt.pub"

meta:
  name: "managed_agent"
  controlled_by: "master_console"

workflow:
  input:
    user_token: string
    command: string
  vars:
    caller_role: "viewer"        # 由认证网关填充
    delegation_token: null
  nodes: [auth, handle_master, normal_task, delegate_task]

pnodes:
  - id: auth
    type: script
    script: |
      // 从令牌解析角色，可能返回多个角色（如 ["operator", "auditor"]）
      let roles = parseJWT(input.user_token).roles;
      ctx.setCallerRoles(roles);   // 多角色支持

  - id: handle_master
    type: master_handler
    condition: "{{input.command == 'stop'}}"
    required_role: "admin"         # 只有 admin 可执行 Master 指令
    script: "ctx.terminate_workflow('master_stop')"

  - id: normal_task
    type: llm
    required_role: "operator"      # 至少 operator 角色
    process: { prompt: "{{workflow.input.command}}" }

  - id: delegate_task
    type: script
    script: |
      // 委派权限给子 Agent
      let token = await ctx.delegateRole("child_agent", {
          tools: ["search"],
          memory: { tier: "fast", operations: ["read"] }
      }, 300);
      await ctx.callAgent("child_agent", { query: "search documents", token });
```

以下示例展示了 PML 的一个完整工作流，集成了角色定义、Master 通道认证、动态角色切换、委派和多角色处理。

- 版本声明为 1.14。
- 角色定义：`admin`（所有权限，允许切换）、`auditor`（仅审计读权限）、`operator`（控制操作、工具 search、fast 记忆读写）。
- 配置：角色合并策略为 `union`；Master 认证使用 JWT 并指定公钥。
- 元数据：工作流名称 `managed_agent`，受控于 `master_console`。
- 工作流输入：`user_token` 和 `command`；变量 `caller_role` 初始为 `viewer`，`delegation_token` 为空。
- 节点：
  - `auth`：解析 JWT 令牌得到角色列表（例如 `["operator", "auditor"]`），并通过 `ctx.setCallerRoles` 设置多角色。
  - `handle_master`：类型 `master_handler`，仅在输入 `command` 为 `stop` 时触发，要求角色为 `admin`，执行终止工作流的脚本。
  - `normal_task`：类型 `llm`，要求至少 `operator` 角色，处理用户命令。
  - `delegate_task`：类型 `script`，在其中通过 `ctx.delegateRole` 创建委派令牌授予 `child_agent` 搜索工具和 fast 记忆读权限（有效期 300 秒），然后调用 `child_agent` 并传入令牌。

该示例完整展示了多角色、委派以及 Master 控制机制。


## 核心特性


- **动态角色切换**与**临时委派**，提高灵活性。
- **角色与能力范围对齐**，防止权限滥用。
- **管理角色细分**（admin/auditor/operator），满足合规要求。
- **跨工作流角色传播**，保障子工作流安全。
- **多角色冲突策略**，明确权限合并逻辑。
- **细粒度记忆/标记控制**，实现数据分级保护。
- **Master 通道强认证**，防止伪造指令。
- **自动化最小权限分析工具**，辅助开发者配置。



# Master控制架构与系统受控模型

> 明确设计 **PML 系统受控于一个 Master** 的顶层控制模型。Master 作为最高权限实体，能够统一注册、监控、命令下发与紧急干预所有下属 Agent 和工作流。该设计使得 PML 可被部署为集中管控的智能体系统，满足企业级治理需求。


## 1. 设计目标

- **集中控制**：单个 Master（可以是一个 Agent、一个工作流或外部系统）拥有对整个 PML 运行时环境的全局管理权限。
- **强制受控**：每个下属 Agent 或工作流必须声明其受控于哪个 Master，未声明的实例默认拒绝运行（可配置）。
- **安全命令通道**：Master 通过专用的、强认证的命令通道（而非普通工作流输入）向下属发送指令，确保指令的真实性和完整性。
- **分级控制**：Master 可授予子 Master 有限的控制权，形成树状控制层次。
- **状态同步**：下属定期向 Master 报告心跳、资源使用、异常等状态。

## 核心概念

| 概念 | 说明 | PML 实现 |
|------|------|----------|
| **Master** | 拥有最高控制权限的实体，可以是 PML 工作流、外部进程或人类操作者 | `role: admin` + 专用 Master 通道 |
| **下属 Agent** | 受 Master 控制的工作流或智能体实例 | `meta.controlled_by` 字段声明 |
| **命令通道** | Master 向下属发送指令的专用连接 | `master_auth` 配置 + 独立 API 端点 |
| **控制指令集** | 预定义的命令：`pause`, `resume`, `terminate`, `upgrade`, `status`, `set_config` | `type: master_handler` 节点处理 |
| **控制层次** | 允许 Master 将部分控制权委派给子 Master | 通过 `delegateRole` + 角色继承实现 |


## 语法与配置扩展

### 全局 Master 配置

在 `config` 中增加 `master_control` 字段，声明整个 PML 运行时的 Master 身份和认证方式。

示例（自然语言描述）：
- 启用 Master 控制模式。
- Master 的标识为 `main-admin`。
- 认证方法为 mutual TLS，指定证书和 CA 文件。
- 允许子 Master 委派，最大深度为 2。

### 下属声明

每个被管控的工作流在其 `meta` 中必须包含 `controlled_by` 字段，指向一个 Master 标识。

示例：
- 工作流名称 `worker_agent`，受控于 `main-admin`。

如果省略该字段且全局 `master_control.enforced` 为 true，则工作流启动时被拒绝。

### Master 命令通道

Master 通过专用的 HTTP 或 gRPC 端点向下属发送命令。命令格式为 JSON，包含 `command`、`params`、`token`（由 Master 签名的 JWT）。

PML 运行时需要监听此通道（默认端口 `pml-master-port`）。下属工作流内部通过 `type: master_handler` 节点接收并处理命令。

### 控制指令集

预定义以下标准命令（所有 `master_handler` 节点应能处理）：

- `pause`：暂停工作流执行，保存当前状态到检查点。
- `resume`：从上次检查点恢复执行。
- `terminate`：终止工作流，释放资源。
- `upgrade`：动态替换工作流定义（热升级）。
- `status`：返回当前执行状态、节点进度、资源消耗。
- `set_config`：动态修改工作流的部分配置（如日志级别、超时时间）。

### 控制层次扩展

通过角色委派（`delegateRole`）机制，Master 可以创建一个子 Master 角色，并授予其部分控制权限（例如只能暂停/恢复，不能终止或升级）。子 Master 再控制下一级 Agent，形成树状结构。


## Master 控制流程示例

以下描述一个完整的主从控制场景：

1. **启动 Master 控制台**：外部系统启动一个 PML 工作流作为 Master，配置 `master_control` 开启，监听端口 9100。
2. **注册下属**：下属工作流 `data_processor.pml` 在 `meta` 中声明 `controlled_by: "main-admin"`，启动时自动向 Master 注册（发送心跳）。
3. **正常执行**：下属独立执行其内部 DAG，同时定期向 Master 发送状态报告（默认每 30 秒）。
4. **Master 发送命令**：Master 控制台通过 HTTP POST 向下属的 `/master` 端点发送 `{"command": "pause", "token": "..."}`。
5. **下属处理**：下属的 `master_handler` 节点验证 token，执行 `ctx.pauseWorkflow()`，保存检查点，并进入暂停状态。
6. **恢复执行**：Master 发送 `resume` 命令，下属从检查点恢复。
7. **委派控制**：Master 创建一个子 Master 角色 `regional-admin`，通过 `delegateRole` 授予其对某组 Agent 的暂停/恢复权限。子 Master 随后可独立控制这些 Agent。


### 示例 1：Master 控制台工作流
Master 控制台负责管理多个下属 Agent，监控状态并发送控制指令。
```
pml:
  version: "1.14"

config:
  master_control:
    enabled: true
    master_id: "main-admin"
    auth:
      method: "mtls"
      cert_file: "/etc/pml/master.crt"
      ca_file: "/etc/pml/ca.crt"
    enforce_controlled: true
    listen_port: 9100
  audit:
    enabled: true
    webhook: "https://audit.example.com/event"

roles:
  - name: admin
    permissions: { all: true }
    allow_assume: true
  - name: operator
    permissions:
      control: ["pause", "resume", "status"]
      tools: ["agent_list"]

workflow:
  input:
    command: string        # 如 "pause", "resume", "status"
    target_agent: string
  vars:
    agent_registry: {}
  nodes: [register_agents, send_command, collect_status]

pnodes:
  - id: register_agents
    type: script
    script: |
      // 接收下属 Agent 的注册请求（通过独立注册端点）
      // 此处简化：手动注册两个示例 Agent
      ctx.state.set("agent_registry", {
          "data_processor": { endpoint: "https://dp.internal:9101/master" },
          "report_generator": { endpoint: "https://rg.internal:9102/master" }
      });

  - id: send_command
    type: tool
    tool:
      name: "http_client"
      endpoint: "{{agent_registry[target_agent].endpoint}}"
      method: "POST"
      headers:
        Authorization: "Bearer {{ctx.generateMasterToken()}}"
      body:
        command: "{{workflow.input.command}}"
        params: {}
    input:
      target_agent: "{{workflow.input.target_agent}}"
    output:
      schema:
        result: { type: object }
    required_role: "operator"

  - id: collect_status
    type: script
    script: |
      let statuses = {};
      for (let [agentId, info] of Object.entries(ctx.state.get("agent_registry"))) {
          let resp = await ctx.http.post(info.endpoint, { command: "status" });
          statuses[agentId] = resp.data;
      }
      return { statuses };
    required_role: "operator"
```

### 示例 2：下属 Agent 工作流
下属 Agent 声明自己受控于 Master，并实现 master_handler 节点响应控制指令。
```
pml:
  version: "1.14"

meta:
  name: "data_processor"
  controlled_by: "main-admin"          # 声明受控于主 Master

config:
  master_control:
    enabled: true
    master_id: "main-admin"
    auth:
      method: "mtls"
      client_cert: "/etc/pml/agent.crt"
      ca_file: "/etc/pml/ca.crt"
    listen_port: 9101                  # 独立端口接收 Master 命令

workflow:
  input:
    data: string
  vars:
    paused: false
  nodes: [master_listener, process_data, health_reporter]

pnodes:
  - id: master_listener
    type: master_handler
    # 自动监听 Master 通道，收到命令后执行
    script: |
      // 该函数由运行时自动调用，传入 command 和 params
      async function onMasterCommand(command, params, ctx) {
          switch(command) {
              case "pause":
                  ctx.state.set("paused", true);
                  await ctx.checkpoint.save("paused_state");
                  return { status: "paused" };
              case "resume":
                  ctx.state.set("paused", false);
                  return { status: "resumed" };
              case "terminate":
                  await ctx.terminateWorkflow("master_terminate");
                  return { status: "terminated" };
              case "status":
                  return { status: "running", progress: ctx.getProgress() };
              default:
                  throw new Error("Unknown command");
          }
      }

  - id: process_data
    type: llm
    control:
      condition: "{{not workflow.vars.paused}}"
    process:
      prompt: { user: "处理数据: {{workflow.input.data}}" }
    output: { result: "{{result.content}}" }

  - id: health_reporter
    type: timer
    control:
      delay: 30
      repeat: true
    script: |
      // 定期向 Master 报告心跳
      let status = { agent: "data_processor", ts: Date.now(), memory_usage: process.memoryUsage() };
      await ctx.http.post("https://main-admin:9100/agent/heartbeat", status);
```

### 示例 3：子 Master 委派控制
Master 可以委派部分控制权给子 Master，子 Master 再管理其下属 Agent。

```
# 子 Master 工作流（具有有限控制权限）
pml:
  version: "1.14"

roles:
  - name: sub_admin
    permissions:
      control: ["pause", "resume"]      # 只能暂停/恢复，不能终止或升级
    allow_assume: false
  - name: operator
    permissions: { all: false }

workflow:
  input:
    target_agent: string
    action: string                      # pause 或 resume
  nodes: [get_token, call_agent]

pnodes:
  - id: get_token
    type: script
    script: |
      // 从 Master 请求委派令牌
      let token = await ctx.requestDelegation("main-admin", {
          target_agent: input.target_agent,
          permissions: ["control:pause", "control:resume"],
          ttl: 600
      });
      return { delegation_token: token };

  - id: call_agent
    type: tool
    tool:
      name: "http_client"
      endpoint: "{{agent_registry[target_agent].endpoint}}"
      method: "POST"
      headers:
        Authorization: "Bearer {{get_token.output.delegation_token}}"
      body:
        command: "{{workflow.input.action}}"
```

### 运行说明

启动下属 Agent：先运行下属工作流（示例 2），它会自动向 Master 注册并监听命令端口。

启动 Master 控制台：运行 Master 工作流（示例 1），确保其能够接收注册心跳。

发送命令：通过 Master 的输入或外部 HTTP 调用，执行 send_command 节点，指定 target_agent 和 command。

委派控制：运行子 Master 工作流（示例 3），它会向主 Master 请求委派令牌，然后使用令牌控制下属 Agent。


## 与现有权限系统的集成

- Master 本质是拥有 `admin` 角色的特殊实体，其命令通道独立于普通用户输入。
- `required_role` 检查在 Master 命令通道中依然生效：只有具备相应角色的调用者才能执行敏感命令。
- 通过 `master_auth` 配置的强认证机制，确保只有持有有效证书或令牌的调用者才能使用 Master 通道。
- 审计日志记录所有 Master 命令及其执行结果。

## 特性汇总

通过本补充设计，PML 明确支持 **系统受控于一个 Master** 的集中式管理模型：

- Master 作为最高权限实体，通过专用强认证通道下发命令。
- 每个下属工作流声明其归属，启动时自动注册并定期同步状态。
- 支持标准控制指令集（暂停、恢复、终止、升级、状态查询、配置修改）。
- 支持控制权委派，形成树状分层治理。
- 与已有 RBAC、角色委派、审计日志等安全机制无缝集成。
