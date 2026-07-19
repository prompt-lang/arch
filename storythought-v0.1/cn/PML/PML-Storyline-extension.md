# PML：故事情节扩展（Storyline Extension）

> 针对 LLM-Loop 复杂任务执行场景，新增**故事情节（Storyline）**容器、**Prompt-Buffer 提示缓冲**、**Issue Trigger 议题触发器**以及**缓冲状态机**。这些扩展使 Agent 能够以 Issue List 的方式管理复杂任务的执行流程，在关键决策点暂停并请求用户讨论，形成结构化的人机协作闭环。


## 设计目标

- **故事情节容器化**：将一次复杂任务的完整执行过程封装为 Storyline，包含目标、阶段、缓冲区和决策历史。
- **Prompt-Buffer 提示缓冲**：Agent 以 Issue List 方式管理待处理提示——排序、依赖、状态追踪。
- **议题触发器**：Agent 在关键决策点暂停执行，发起结构化议题供用户讨论，收到反馈后继续。
- **缓冲状态机**：每个 Buffer Item 遵循 `pending → in_progress → blocked → resolved/closed` 的确定性状态流转。
- **人类协作闭环**：用户讨论意见被结构化为 Issue Response，Agent 据此更新策略后恢复执行。


## 核心概念

| 概念 | 说明 | PML 实现 |
|------|------|----------|
| **Storyline** | 一次复杂任务的完整叙事容器。包含目标、阶段图、缓冲池、议题记录、决策历史。一次 Inloop Session 可以包含多个 Storyline，Storyline 之间可以串行或嵌套。 | 顶层 `storyline` 字段 + `type: storyline` 节点 |
| **Prompt-Buffer** | Agent 的"待办事项列表"。每个条目是一个结构化 Prompt Issue，包含标题、描述、优先级、依赖、状态、指派人（Agent 或 Human）。 | `storyline.buffer` + `type: buffer` 节点 |
| **Buffer Item** | 缓冲池中的单个待处理项。包含 `id`、`title`、`description`、`priority`、`status`、`dependencies`、`assignee`。 | `storyline.buffer.items[]` |
| **Issue Trigger** | 当 Agent 无法自行决策时，生成一个结构化议题，暂停当前 Buffer Item，等待用户输入后再继续。议题可附带选项、推荐方案和影响分析。 | `type: issue_trigger` 节点 |
| **Issue Response** | 用户对议题的结构化回复。包含决策选择、理由、约束条件。 | `type: issue_response` 节点 |
| **Buffer State Machine** | 每个 Buffer Item 的状态机：`pending` → `in_progress` → (`blocked` → `in_progress`) → `resolved` / `closed` | `item.status` 枚举 |


## 语法与功能扩展

### 1. Storyline 顶层声明

```yaml
storyline:
  id: "refactor_user_module"
  title: "重构用户模块"
  description: "将 User 模型拆分为 Account + Profile，迁移所有引用并保持 API 兼容"
  phases:
    - id: "phase_1_analysis"
      title: "现状分析"
      order: 1
      goal: "梳理所有 User 模型的引用点和依赖关系"
    - id: "phase_2_design"
      title: "方案设计"
      order: 2
      goal: "确定拆分方案和迁移步骤"
      requires_human_approval: true       # 此阶段完成后需人工审批
    - id: "phase_3_implementation"
      title: "编码实现"
      order: 3
      goal: "执行拆分和迁移"
    - id: "phase_4_verification"
      title: "验证收尾"
      order: 4
      goal: "测试覆盖、清理旧代码、更新文档"

  # ── Prompt-Buffer 初始定义 ──
  buffer:
    items:
      - id: "BUF-001"
        title: "分析 User 模型的所有引用"
        description: "搜索代码库中所有引用 User 结构体的位置，按文件分组统计"
        priority: 10
        dependencies: []
        assignee: "agent"
        phase: "phase_1_analysis"

      - id: "BUF-002"
        title: "分析 User 相关的 API 端点"
        description: "列出所有暴露 User 模型的 API，标注哪些是公开接口"
        priority: 9
        dependencies: ["BUF-001"]
        assignee: "agent"
        phase: "phase_1_analysis"

      - id: "BUF-003"
        title: "确定拆分方案"
        description: "基于分析结果，提出 Account/Profile 拆分方案，标注影响范围"
        priority: 8
        dependencies: ["BUF-001", "BUF-002"]
        assignee: "agent"
        phase: "phase_2_design"
        # 此议题需要触发用户讨论
        requires_issue_trigger: true

      - id: "BUF-004"
        title: "执行字段迁移"
        description: "按照通过审批的方案执行 User → Account/Profile 字段迁移"
        priority: 7
        dependencies: ["BUF-003"]
        assignee: "agent"
        phase: "phase_3_implementation"

      - id: "BUF-005"
        title: "更新测试用例"
        description: "修改所有引用 User 模型的测试用例"
        priority: 6
        dependencies: ["BUF-004"]
        assignee: "agent"
        phase: "phase_4_verification"

  # ── 议题记录（Issue Log）──
  issues:
    - id: "ISS-001"
      buffer_item: "BUF-003"
      title: "拆分方案需要确认"
      description: "两种方案：1) 激进拆分（立即移除 User）2) 渐进拆分（保留 User 别名 3 个月）"
      options:
        - id: "aggressive"
          label: "激进拆分"
          impact: "高风险，但代码清理彻底"
        - id: "gradual"
          label: "渐进拆分"
          impact: "低风险，需要更多维护窗口"
      recommendation: "gradual"
      status: "open"

  # ── 决策历史 ──
  decisions: []
```

### 2. Prompt-Buffer 状态机

每个 Buffer Item 遵循 **5 状态确定性格流转**：

```
                    ┌──────────┐
                    │ pending  │  ← 初始状态
                    └────┬─────┘
                         │ Agent 开始处理
                         ▼
                    ┌──────────┐
              ┌────→│in_progress│
              │     └────┬─────┘
              │          │ 遇到阻塞（需人类决策 / 依赖未完成）
              │          ▼
              │     ┌──────────┐
              │     │ blocked  │  ← 等待外部输入
              │     └────┬─────┘
              │          │ Issue Trigger 获得用户回复
              │          │ 或依赖项完成
              │          ▼
              │     ┌──────────┐
              │     │in_progress│ (恢复)
              │     └────┬─────┘
              │          │ 任务完成
              │          ▼
              │     ┌──────────┐     ┌──────────┐
              └─────│ resolved │     │  closed  │  ← 被取消 / 不再需要
                    └──────────┘     └──────────┘
```

状态转移规则：
- `pending → in_progress`：Agent 开始处理（自动或由 dispatcher 节点触发）
- `in_progress → blocked`：触发 Issue Trigger（需人类决策）/ 依赖项未 resolved
- `blocked → in_progress`：收到 Issue Response / 依赖项 resolved
- `in_progress → resolved`：任务完成，结果写入 buffer item 的 `resolution` 字段
- `in_progress → closed`：任务被取消或不再需要

### 3. Buffer 节点类型

#### `type: buffer` — 缓冲池管理

```yaml
pnodes:
  # 查询缓冲池状态
  - id: get_pending_items
    type: buffer
    operation: query
    storyline: "refactor_user_module"
    query:
      status: ["pending", "blocked"]      # 筛选条件
      priority_min: 5                     # 最低优先级
      phase: "phase_1_analysis"           # 限定阶段
      dependencies_resolved: true         # 仅返回依赖已满足的项
    order_by: "priority desc"
    limit: 5
    output:
      schema:
        items: { type: array }

  # 更新缓冲项状态
  - id: start_item
    type: buffer
    operation: update
    storyline: "refactor_user_module"
    item_id: "BUF-001"
    updates:
      status: "in_progress"
      started_at: "{{workflow.now}}"
    control:
      # 状态转移验证：仅允许合法状态转移
      validate_transition: true

  # 完成缓冲项
  - id: resolve_item
    type: buffer
    operation: update
    storyline: "refactor_user_module"
    item_id: "BUF-001"
    updates:
      status: "resolved"
      resolution:
        summary: "发现 47 处 User 引用，分布在 12 个文件中"
        artifact_ref: "{{analysis_result.output.report}}"
      completed_at: "{{workflow.now}}"

  # 动态创建新的缓冲项（Agent 自主发现的新任务）
  - id: add_new_item
    type: buffer
    operation: create
    storyline: "refactor_user_module"
    item:
      title: "修复发现的循环依赖"
      description: "在迁移过程中发现 account.go 和 profile.go 存在循环引用"
      priority: 10
      dependencies: ["BUF-004"]
      assignee: "agent"
      phase: "phase_3_implementation"
```

### 4. Issue Trigger — 议题触发器

#### `type: issue_trigger` — 暂停并请求人类决策

```yaml
pnodes:
  - id: request_split_decision
    type: issue_trigger
    storyline: "refactor_user_module"
    buffer_item: "BUF-003"                # 关联的缓冲项

    # ── 议题定义 ──
    issue:
      title: "User 模型拆分方案确认"
      summary: |
        已完成现状分析（BUF-001, BUF-002）。发现 User 模型在 47 处被引用。

        现有两种拆分方案，请确认选择：

        **方案 A：激进拆分**
        - 立即创建 Account + Profile 替代 User
        - 一次性移除所有 User 引用
        - 风险：高（可能遗漏隐蔽引用）
        - 工期：2-3 天

        **方案 B：渐进拆分**
        - 先创建 Account + Profile
        - 保留 User 作为类型别名，标记 deprecated
        - 3 个月后移除别名
        - 风险：低（向后兼容）
        - 工期：5-7 天（含过渡期维护）

      options:
        - id: "aggressive"
          label: "方案 A：激进拆分"
          risk: "high"
        - id: "gradual"
          label: "方案 B：渐进拆分（推荐）"
          risk: "low"
          recommended: true

      recommendation:
        option: "gradual"
        rationale: "考虑到 User 模型被广泛引用且部分引用来自第三方集成代码，渐进拆分降低回归风险。"
        confidence: 0.85

      # 需要用户提供的额外信息
      required_context:
        - question: "是否有即将发布的版本需要避免影响？"
          type: "boolean"
        - question: "渐进方案的 3 个月过渡期是否可以接受？"
          type: "text"

    # ── 阻塞策略 ──
    blocking:
      mode: "pause_storyline"             # pause_buffer_item | pause_phase | pause_storyline
      timeout_hours: 72                   # 超时自动提醒
      on_timeout: "remind"                # remind | auto_close | escalate

    output:
      schema:
        issue_id: { type: string }
        status: { type: string }
```

#### `type: issue_response` — 接收人类决策

```yaml
  - id: receive_decision
    type: issue_response
    storyline: "refactor_user_module"
    issue_id: "ISS-001"

    # ── 用户回复 ──
    response:
      selected_option: "gradual"
      rationale: "下周有 v2.3 发布，激进方案风险太高。渐进方案 3 个月过渡期可接受。"
      additional_context:
        next_release_date: "2026-08-01"
        accept_transition_period: true
      constraints:
        - "不能影响 v2.3 发布"
        - "API 兼容性不能破坏"

    # ── 后续动作 ──
    on_resolve:
      resume_buffer_item: true            # 自动恢复 BUF-003 为 in_progress
      update_storyline: true              # 记录决策到 storyline.decisions
      notify_phase: "phase_3_implementation"
```

### 5. 故事线恢复与上下文注入

当 Buffer Item 从 `blocked` 恢复为 `in_progress` 时，NNOS 自动构建恢复上下文：

```yaml
pnodes:
  - id: resume_work
    type: llm
    storyline: "refactor_user_module"
    buffer_item: "BUF-003"

    # ── 自动注入恢复上下文 ──
    context:
      # 显式部分
      explicit:
        user: "根据确认的方案继续执行 BUF-003"

      # 隐式部分（NNOS 自动组装）
      implicit_sources:
        - type: buffer_context          # 当前 buffer item 的完整历史
          item_id: "BUF-003"
          include:
            - original_description
            - previous_attempts
            - issue_history             # 所有相关的 issue trigger + response

        - type: dependency_context      # 已完成的依赖项结果
          dependencies_of: "BUF-003"
          include_resolutions: true

        - type: phase_context           # 当前阶段的所有已完成项
          phase: "phase_2_design"

        - type: decision_context        # 相关的人类决策记录
          related_to: "BUF-003"

    process:
      prompt:
        user: "{{workflow.input.user_instruction}}"
```

### 6. 完整示例：复杂任务执行流程

```yaml
pml:
  version: "1.16"

storyline:
  id: "refactor_user_module"
  title: "重构用户模块"
  phases:
    - { id: "phase_1_analysis", title: "现状分析", order: 1 }
    - { id: "phase_2_design", title: "方案设计", order: 2, requires_human_approval: true }
    - { id: "phase_3_implementation", title: "编码实现", order: 3 }
    - { id: "phase_4_verification", title: "验证收尾", order: 4 }

  buffer:
    items:
      - { id: "BUF-001", title: "分析 User 引用", priority: 10, phase: "phase_1_analysis" }
      - { id: "BUF-002", title: "分析 API 端点", priority: 9, phase: "phase_1_analysis", dependencies: ["BUF-001"] }
      - { id: "BUF-003", title: "确定拆分方案", priority: 8, phase: "phase_2_design", dependencies: ["BUF-001","BUF-002"], requires_issue_trigger: true }
      - { id: "BUF-004", title: "执行字段迁移", priority: 7, phase: "phase_3_implementation", dependencies: ["BUF-003"] }
      - { id: "BUF-005", title: "更新测试", priority: 6, phase: "phase_4_verification", dependencies: ["BUF-004"] }

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
  # Step 1: 调度器 — 选取下一个可执行的 Buffer Item
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

  # Step 2: 执行当前 Item
  - id: execute_item
    type: llm
    control:
      loop:
        condition: "{{output.status == 'in_progress'}}"
    process:
      prompt:
        system: |
          当前阶段：{{storyline.current_phase.title}}
          当前任务：{{dispatcher.output.next_item.title}}
          任务描述：{{dispatcher.output.next_item.description}}
          依赖项结果：{{dependency_context}}

  # Step 3: 检查是否需要阻塞
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

  # Step 4: 触发议题
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

  # Step 5: 接收人类回复
  - id: receive_response
    type: issue_response
    storyline: "refactor_user_module"
    issue_id: "{{trigger_issue.output.issue_id}}"
    response:
      selected_option: "{{workflow.input.user_instruction}}"
    on_resolve:
      resume_buffer_item: true
```

### 7. 与现有 PML 特性的集成

| 集成点 | 方式 | 价值 |
|--------|------|------|
| **Session / Inloop** | Storyline 运行在 Inloop Session 内，共享会话上下文 | 长时间任务不丢失状态 |
| **PEM 工程记忆** | `storyline.decisions` 自动写入 `engineering_kb` | 跨会话的决策知识积累 |
| **Markdown 提示卡** | Issue Trigger 可查询 `prompt_cards` 域获取相关规范 | 决策时自动注入项目约定 |
| **SOF 自身本体** | Agent 查询 `self.cognitive.planning_horizon` 决定 Buffer 分块粒度 | 自适应任务分解 |
| **RBE 规则引擎** | Buffer 调度策略可用 `type: rule` 节点实现 | 声明式调度规则 |
| **Checkpoint** | Storyline 完整状态可检查点持久化 | 故障恢复后无缝继续 |

### 8. 设计原则

1. **Storyline 是任务的"叙事容器"** — 不是简单的 TODO list，而是包含阶段、决策、上下文演进的完整故事。
2. **Buffer 是 Agent 的"工作记忆"** — 结构化、可重排、有依赖、有状态的 Issue List。
3. **Issue Trigger 是"人机握手点"** — 在 Agent 能力边界处暂停，结构化地请求人类智慧。
4. **状态机驱动，非轮询驱动** — 每个 Buffer Item 的状态转移是确定性的、可审计的、可回滚的。
5. **与 Session/PEM/Markup 深度集成** — 不新建孤岛，复用全部现有基础设施。
