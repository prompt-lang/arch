
# PML_RBE 设计规范：PNode RuleBased Engine

## 1. 概述

### 1.1 背景与动机
在复杂智能体工作流中，大量业务逻辑以“条件→动作”的模式存在。这类规则型逻辑若通过 `type: script` 节点逐条硬编码，将导致规则难以维护、无法热更新，且缺乏统一的规则执行模型。为填补这一空白，本文提出 PML_RBE（RuleBased PNode）——一种基于规则驱动的 PNode 类型，使 PML 工作流能够以声明式方式定义、管理和执行规则。

### 1.2 设计目标
- 声明式规则定义：用户仅需描述“条件→动作”，无需编写执行流程代码
- 双模实现：规则可以是内联的 PML Script，也可以是外部可执行系统程序
- 与 PNode 体系无缝集成：作为一种新的 PNode 类型（`type: rule`），保持与现有工作流、记忆、环境系统的互操作性
- 支持热更新：规则可通过 PNode Protocol 或执行器接口动态加载和更新
- 高性能执行：在条件匹配阶段引入模式共享与缓存机制，避免重复评估

## 2. 核心概念

### 2.1 规则引擎基本原理
规则引擎遵循“条件匹配 → 动作执行”的基本范式。其核心组件包括：
- **规则（Rule）**：由条件和动作组成的声明式语句，映射至 `rules` 字段中的每个条目
- **条件（Condition）**：true/false 布尔表达式，包含作用于事实的一个或多个谓词，对应 `condition` 字段
- **动作（Action）**：条件满足时执行的操作，对应 `action` 字段
- **事实（Fact）**：规则评估的输入数据，即 `input` 字段绑定的工作流状态
- **规则集（RuleSet）**：一组相关规则的集合，即 `rules` 数组的整体
- **议程（Agenda）**：按优先级排序的待执行动作队列，由引擎内部管理

### 2.2 ECA 模式
业界标准的规则系统广泛采用 Event-Condition-Action (ECA) 模式。PML_RBE 继承此模式：
- **Event**：由 PML 工作流中的上游节点输出或外部系统事件隐式定义，通过 `trigger` 字段声明
- **Condition**：声明式的谓词表达式，支持逻辑组合（AND/OR）
- **Action**：可执行的动作体（PML Script、系统命令、HTTP 调用等）

### 2.3 PML_RBE 与现有 PNode 的关系
与普通 PNode（如 `script`、`llm`）相比，RuleBased PNode（`type: rule`）的执行模型为多规则并行匹配、议程驱动，而非单一路径一次执行；其推理能力仅限于确定性条件匹配，但实现了逻辑与引擎的完全分离，支持热更新，且规则间可定义优先级与冲突解决策略。

## 3. PML_RBE 语法规范

### 3.1 顶层结构
规则节点的定义位于 `pnodes` 列表中，节点类型为 `rule`，核心字段包括：
- `id`、`description`：节点标识与描述
- `implementation`：取值为 `pml_script` 或 `system_program`，指定规则的实现方式
- `trigger`：触发模式，可包含 `mode`（`on_input`、`on_event` 或 `periodic`），以及对应的 `event_filter` 或 `interval_sec`
- `rules`：规则列表，每条规则包含 `name`、`priority`（整数，越大越优先）、`condition`（条件表达式）和 `action`（动作定义，类型可为 `execute`、`set_var`、`emit_event`、`call_node` 等）
- `conflict_resolution`：冲突解决策略，可选 `first_match`、`priority` 或 `all_match`
- `control`：控制参数，包括 `timeout`（秒）、`max_rules_per_run`、`fallback_action`（无规则匹配时的兜底动作）

### 3.2 实现方式一：PML Script 内嵌
当 `implementation` 为 `pml_script` 时，规则的动作体由内联的 PML Script 定义。以下描述一个信贷审批规则引擎的典型定义：

节点标识为 `credit_evaluator`，实现方式为 `pml_script`，触发模式为 `on_input`。规则集包含三条规则：
- 规则 `high_value_approve`：优先级 100，条件为 `input.credit_score > 750 and input.income > 50000`，动作为设置工作流变量 `approval_result`，状态 `approved`，额度为申请金额的 90%。
- 规则 `medium_review`：优先级 50，条件为信用分在 600 到 750 之间，动作为设置状态为 `manual_review`，需人工审核。
- 规则 `low_deny`：优先级 10，条件为信用分小于等于 600，动作为状态 `denied`。

冲突解决策略设为 `first_match`，超时 5 秒，兜底动作返回 `error` 状态。节点定义了输入字段 `credit_score`、`income`、`requested_amount`（均为数字类型），输出对象包含 `approval_result`。

### 3.3 实现方式二：可执行系统程序
当 `implementation` 为 `system_program` 时，规则引擎由外部可执行程序承载，PML 通过标准输入/输出接口与之交互。例如一个安全威胁检测引擎，节点配置如下：
- `trigger` 为 `on_input`
- `system` 字段指定可执行文件路径、启动参数（如 `./bin/threat_rules_engine --ruleset ./rules/threat_rules.json --format json`），通信协议可为 `stdio`、`http` 或 `grpc`，并可启用沙箱限制内存与 CPU
- 输入为包含 `sensor_data` 和 `timestamp` 的对象，输出为 `alerts` 和 `actions_taken` 数组
- 控制参数中可设置超时（如 30 秒）和失败重启策略

#### 系统程序接口规范
外部程序通过 JSON-Lines 协议与 PML 运行时交互。请求格式包含 `request_id`、操作类型 `evaluate` 以及 `facts` 对象（如传感器数据）。响应格式返回 `status`、`matched_rules` 数组（内含规则名、优先级、动作列表）及执行耗时。此外，还支持管理指令，如 `reload_rules` 用于热更新规则集，`status` 用于查询引擎状态。

## 4. 条件表达式规范

### 4.1 基本语法
条件表达式采用类 SQL 的声明式语法，支持的操作符包括：比较（`==`、`!=`、`>` 等）、逻辑（`and`、`or`、`not`）、集合（`in`、`not in`）、空值判断（`is null`、`is not null`）、正则匹配（`matches`）、范围（`between`）以及存在性检查（`exists`）。

### 4.2 复合条件
条件可通过括号和逻辑操作符组合，例如“金融类型且金额大于 10000，或医疗类型且优先级为紧急”。

### 4.3 上下文函数
条件中可引用以下 PML 上下文函数，访问工作流状态、环境标记和记忆数据：
- `ctx.var("<path>")`：读取工作流变量
- `ctx.markup("<domain>", "<query>")`：查询环境标记
- `ctx.memory("<backend>", "<key>")`：读取记忆数据

示例条件：输入速度超过工作流变量中的限速值，且世界地图中距离小于 10 的障碍物数量大于 0。

## 5. 规则执行模型

### 5.1 执行流程
整体流程分为四个阶段：
1. **输入阶段**：PNode 接收来自上游节点或外部系统的 input 数据，构建事实集
2. **条件匹配阶段**：遍历规则集中的每条规则，评估其 condition 表达式；匹配条件在规则间共享，仅评估一次以提高效率
3. **议程构建阶段**：根据 conflict_resolution 策略，将匹配的规则按优先级排序
4. **动作执行阶段**：按议程顺序执行动作，动作可能修改工作流状态、触发事件或调用其他节点；若无规则匹配，则执行 fallback_action

### 5.2 冲突解决策略
- `first_match`：仅执行第一个匹配的规则
- `priority`：按优先级排序，执行所有匹配的规则（高优先级优先）
- `all_match`：执行所有匹配的规则，无特定顺序

### 5.3 与工作流图的集成
`type: rule` 节点与普通 PNode 一样，嵌入在 PML 工作流的 DAG 中。例如，一个包含数据预处理、规则评估、动作分发和通知的工作流，其边依次连接。规则评估节点可引用上游输出，并根据规则执行结果在后续的 switch 节点中进行分支处理（如阻断交易、发送警报等）。

## 6. 规则管理

### 6.1 规则集引用
大型规则集可抽取为独立文件，通过 `ruleset` 字段引用外部 YAML 文件路径。外部规则文件包含版本、领域和规则列表，每条规则的定义与内联规则一致。

### 6.2 热更新机制
通过 PNode Protocol 的 update 操作，可在运行时动态替换规则集，只需指定节点 ID 和新规则集路径。对于系统程序实现，也可通过 `reload_rules` 指令触发热更新。

## 7. 与 PML 体系集成

### 7.1 与执行器元模型的关系
在顶层 `executors` 声明中，可将规则引擎注册为一种执行器后端，描述其算法（如 Rete）、支持的操作符、最大规则容量和平均匹配延迟等能力。

### 7.2 与记忆系统的集成
规则引擎的结果可持久化到 PML 的记忆后端，支持跨工作流的规则执行审计，例如将规则执行输出写入长期记忆层，键中包含工作流运行 ID。

### 7.3 与标记环境的集成
规则条件可直接查询环境标记，实现感知型规则。例如避障规则的条件可描述为：从世界地图标记中查询距离小于 5 的障碍物数量大于 0，动作为调用紧急停止节点。

### 7.4 与智能体分层的关系
`type: rule` 节点可通过 `required_capability` 字段绑定到特定层级的智能体（如 `local` 规划器），以保证低延迟执行。

## 8. 实现注意事项

1. 条件缓存：跨规则共享的条件表达式应仅评估一次，结果缓存于工作内存中
2. 沙箱隔离：无论是内嵌脚本还是外部程序，都应运行在独立沙箱中，防止影响主调度器
3. 超时控制：规则集执行必须有超时限制，超时后触发 on_error 策略
4. 协议安全性：系统程序模式下的通信端点应支持认证（IPC 限定本地进程，网络端点启用 mTLS）
5. 状态持久化：规则引擎的执行状态应可序列化到 checkpoint，用于故障恢复
6. 可观测性：输出规则匹配数量、执行耗时、各规则命中率等指标，便于优化规则集
