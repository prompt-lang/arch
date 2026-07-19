# PML World Markup Extension

## Scene-Markup
 
设计原则（新增）
- 职责单一：标记的 CRUD、查询、渲染、变换、流处理使用独立节点类型。
- 实时优先：支持标记流订阅与窗口聚合，适配传感器数据。
- 语法统一：标记查询统一使用 JSONPath 语法，与数据绑定表达式一致。
- 事件驱动：标记变更可触发工作流节点，避免轮询。
- 可组合：标记可作为记忆持久化，可与脚本、LLM 深度集成。

---
标记域定义（Markup Domain）

```
markup_domains:
  - id: "scene_3d"
    type: "spatial"                     # spatial, temporal, spatiotemporal
    coordinate_system: "world"
    bounds: [-100, -100, -50, 100, 100, 50]
    versioning: true                    # 启用乐观锁版本控制（v1.4 新增）
    conflict_policy: "last_write_wins"  # last_write_wins, merge, raise_error
    storage:
      backend: "memory"                 # memory, postgis, redis
      ttl_sec: 3600                     # 标记自动过期时间
```
- `versioning`：开启后每个标记包含 `version` 字段，更新时需匹配版本号。
- `conflict_policy`：并发冲突时的解决策略。


## 标记节点类型
### type: markup_crud（创建、读取、更新、删除）
```
pnodes:
  - id: manage_car
    type: markup_crud
    domain: "scene_3d"
    operation: "create"                 # create, read, update, delete
    markup:
      id: "car_001"                     # 若 create 时未提供，自动生成
      type: "vehicle"
      region: { type: "bbox", coordinates: [1,2,3,4,5,6] }
      attributes: { speed: 10 }
      # 可选：version (仅 update/delete 时需要)
    # 输出: { success: bool, markup: {...}, version: number }
```
- update 操作需在 markup 中包含 id 和 version，若版本不匹配则失败。
- delete 可仅提供 id 或 query（见下）。
  
### type: markup_query（查询）
```
pnodes:
  - id: find_cars
    type: markup_query
    domain: "scene_3d"
    query: "$..[?(@.type=='vehicle')]"   # JSONPath 语法（统一）
    # 可选：limit, offset, sort
    limit: 10
    output:
      schema:
        results: { type: array }
```
查询语法：使用 `JSONPath`（例如 `$.markups[*]` 或 `$..[?(@.type=='vehicle')]）`，与 PML 数据绑定表达式中的 `{{}}` 内语法保持一致。

### type: markup_render（渲染）
```
pnodes:
  - id: render_view
    type: markup_render
    domain: "scene_3d"
    query: "$..[?(@.type=='vehicle')]"   # 可选，不提供则渲染整个域
    render:
      mode: "image"                      # image, pointcloud, mesh
      camera: { position: [0,0,10], look_at: [0,0,0] }
      style: "colored_bbox"
      output_format: "png"
    resources:
      timeout_sec: 30
    output: { image_data: "{{result.base64}}" }
```

### type: markup_transform（坐标变换）
```
pnodes:
  - id: world_to_pixel
    type: markup_transform
    domain: "scene_3d"
    query: "$..[?(@.type=='vehicle')]"
    transform:
      source_coordinate_system: "world"
      target_coordinate_system: "pixel"
      method: "projection"
      parameters:
        intrinsic: [fx, fy, cx, cy]
        extrinsic: [...]
    output: { transformed_markups: "{{result}}" }
```
### type: markup_stream（实时标记流 - v1.4 新增）
订阅标记域的变更事件或从外部流摄入标记。

```
protocols:
  - id: "lidar_stream"
    type: mq
    provider: "kafka"
    topic: "lidar_markups"

pnodes:
  - id: track_vehicles
    type: markup_stream
    domain: "scene_3d"
    source:
      protocol: "lidar_stream"           # 引用 protocols
      format: "json"                     # 消息格式
    window:
      type: "sliding"
      duration_sec: 5.0
      step_sec: 1.0
    output:
      schema:
        aggregated: { type: array }
```

- 该节点持续运行（类似 timer），每个窗口输出聚合结果。
- 支持 window 聚合（计数、平均位置等）。

### 标记变更事件（Markup Events）

标记域支持变更事件，其他节点可订阅。

```
markup_domains:
  - id: "scene_3d"
    events:
      on_create: "markup_created_channel"
      on_update: "markup_updated_channel"
      on_delete: "markup_deleted_channel"
```
在工作流中订阅：

```
pnodes:
  - id: log_new_cars
    type: script
    communication:
      inbound:
        - protocol: "event_bus"
          event: "markup_created_channel"
          handler: "onNewCar"
```

### 标记注意力增强（Markup Attention）

#### 注意力热图输出

在 `control.attention` 中可指定输出注意力图。

```
pnodes:
  - id: focus_on_cars
    type: llm
    control:
      attention:
        domain: "scene_3d"
        query: "$..[?(@.type=='vehicle')]"
        mode: "heatmap"                  # boost, suppress, heatmap
        output_heatmap_to: "{{workflow.vars.attention_map}}"
```
- `mode: heatmap` 生成与输入图像/空间同尺寸的热图，保存到指定状态变量。

### 独立注意力节点

使用 `type: markup_query + type: function` 实现自定义注意力权重。
```
- id: compute_attention
  type: markup_query
  domain: "scene_3d"
  query: "$..[?(@.attention_score > 0.5)]"
  output: { important_regions: "{{result}}" }
```

### 标记与记忆集成

允许将标记域持久化为长期记忆，或从记忆恢复。
```
memory_backends:
  - id: "markup_store"
    type: markup_adapter           # 新增适配器
    backend: "qdrant"
    collection: "scene_markups"

pnodes:
  - id: save_markups
    type: memory
    operation: write
    backend: "markup_store"
    value: "{{markups.scene_3d}}"   # 整个域作为记忆存储

  - id: load_markups
    type: memory
    operation: read
    key: "scene_3d_snapshot"
    output: { restored_domain: "{{result}}" }
```
- type: markup_adapter 自动将标记域序列化为向量或键值对，支持语义检索。

### 脚本 API 集成（PMLScript）
在 PMLScript 的 ctx 中注入 markup 对象。
```
// 在 type: script 节点中
async function run(input, ctx) {
    // 查询标记
    let cars = await ctx.markup.query("scene_3d", "$..[?(@.type=='vehicle')]");
    // 读取单个标记
    let car = await ctx.markup.read("scene_3d", "car_001");
    // 创建标记
    let newId = await ctx.markup.create("scene_3d", {
        type: "pedestrian",
        region: { type: "point", coordinates: [1,2,0] }
    });
    // 更新标记（带版本控制）
    await ctx.markup.update("scene_3d", "car_001", { speed: 20 }, version);
    // 删除标记
    await ctx.markup.delete("scene_3d", "car_001");
    return { car_count: cars.length };
}
```

- 所有方法返回 Promise，支持异步。
- 版本冲突时抛出异常，可由脚本捕获并重试。

### 层级标记聚合操作
在 markup_crud 或 markup_query 中增加 aggregate 选项。

```
pnodes:
  - id: aggregate_room
    type: markup_query
    domain: "building"
    query: "$..[?(@.type=='room')]"
    aggregate:
      group_by: "$.parent_id"           # 按父标记 ID 分组
      functions:
        - name: "avg_temperature"
          field: "attributes.temperature"
          op: "avg"
        - name: "count_devices"
          field: "children"
          op: "count"
    output: { statistics: "{{result}}" }
- 支持 avg, sum, min, max, count。
- 结果可直接用于下游决策。
存储与性能配置（细化）
config:
  markup_storage:
    default_backend: "memory"
    backends:
      - id: "postgis"
        type: "postgresql"
        dsn: "postgres://user:pass@localhost/markups"
        pool_size: 10
        auto_index: true
    index:
      spatial: "r_tree"
      temporal: "interval_tree"
    cache:
      enabled: true
      max_size_mb: 512
      entry_ttl_sec: 60
```

- 支持分片：shard_by: "type" 或 shard_by: "time_bucket"。
  
完整示例：实时标记流

```
pml:
  version: "1.4"

markup_domains:
  - id: "perception"
    type: "spatiotemporal"
    versioning: true

protocols:
  - id: "lidar_input"
    type: mq
    provider: "kafka"
    topic: "lidar_markups"

workflow:
  input: {}
  output: { steering: "{{plan.output.angle}}" }
  nodes: [stream_consumer, query_vehicles, plan_path]

pnodes:
  - id: stream_consumer
    type: markup_stream
    domain: "perception"
    source: { protocol: "lidar_input" }
    window: { type: "sliding", duration_sec: 3.0, step_sec: 0.5 }
    output: { vehicles: "{{result}}" }

  - id: query_vehicles
    type: markup_query
    domain: "perception"
    query: "$..[?(@.type=='vehicle')]"
    output: { vehicle_list: "{{result}}" }

  - id: plan_path
    type: neuro_task
    # 宏：通过脚本使用 markup API
    script: |
      async function run(input, ctx) {
          let vehicles = await ctx.markup.query("perception", "$..[?(@.type=='vehicle')]");
          let nearest = vehicles.reduce((a,b) => a.dist < b.dist ? a : b);
          let angle = Math.atan2(nearest.y, nearest.x);
          return { angle };
      }
```

## Q-Markup

### Q 值定义
Q 值（Quality Score）是每个标记可携带的一个浮点数，用于量化标记的置信度、重要性、可靠性或任何自定义指标。

- 范围：通常为 [0.0, 1.0]，但允许任意实数（可由业务定义）。
- 默认值：若未显式指定，默认为 1.0（最高质量）。
- 语义：Q 值越高表示标记越“好”或越“重要”。在压缩或选择时，可依据 Q 值阈值或比例进行操作。

##### 标记结构中的 Q 值

```
markup:
  id: "car_001"
  type: "vehicle"
  region: { type: "bbox", coordinates: [...] }
  attributes: { speed: 15 }
  q: 0.92                     # Q 值，可选
  version: 1
```


#### Q 值在标记节点中的应用
##### markup_crud：创建与更新时设置 Q 值

创建标记时可指定初始 Q 值：

```
- id: create_car
  type: markup_crud
  operation: create
  domain: "scene_3d"
  markup:
    type: "vehicle"
    region: {...}
    q: 0.85                     # 初始 Q 值
```
更新时可修改 Q 值：

```
- id: update_car_q
  type: markup_crud
  operation: update
  markup:
    id: "car_001"
    version: 2
    q: 0.95
markup_query：基于 Q 值的过滤、排序与限制
新增 q_filter 和 q_sort 字段。
- id: query_high_quality
  type: markup_query
  domain: "scene_3d"
  query: "$..[?(@.type=='vehicle')]"
  q_filter:
    min: 0.8                   # 只返回 Q >= 0.8 的标记
    max: 1.0                   # 可选上限
  q_sort:
    order: "desc"              # 按 Q 值降序排列
  limit: 5                     # 取前 5 个最高 Q 的标记
```

- q_filter.min / max 直接筛选。
- q_sort 与 limit 结合可实现 Top-K 选择。
- 也支持与普通 JSONPath 条件组合（AND 语义）。


##### markup_render：按 Q 值控制渲染细节

可根据 Q 值阈值决定是否渲染某些标记，或调整渲染样式。
```
- id: render_best
  type: markup_render
  domain: "scene_3d"
  query: "$..[?(@.type=='vehicle')]"
  q_filter: { min: 0.9 }        # 只渲染高 Q 车辆
  render:
    mode: "image"
    style: "colored_bbox"
    # 低 Q 标记可渲染为半透明或忽略
    low_q_style: "transparent"
```

##### markup_stream：窗口内按 Q 值加权聚合

在流式聚合中，可依据 Q 值进行加权平均。
```
- id: aggregate_tracks
  type: markup_stream
  domain: "scene_3d"
  source: { protocol: "lidar_stream" }
  window: { type: "sliding", duration_sec: 3.0 }
  aggregate:
    group_by: "$.id"                 # 按标记 ID 分组
    functions:
      - name: "weighted_position"
        field: "region.bbox.center"
        op: "weighted_avg"
        weight_field: "q"            # 使用 Q 值作为权重
      - name: "average_q"
        field: "q"
        op: "avg"
    output: { result: "{{aggregated}}" }
```
- 支持 weighted_avg、weighted_sum 等操作。
- 可用于融合多帧检测结果，给置信度高的检测更大权重。

#### 新增节点：type: markup_compress
专门用于根据 Q 值压缩标记集，减少存储或传输开销。
```
- id: compress_by_q
  type: markup_compress
  domain: "scene_3d"
  method: "threshold"                 # threshold, top_k, percentile
  threshold:
    min_q: 0.95                       # 删除 Q < 0.95 的标记
  # 或者保留 Top-K
  # top_k: 100
  # 或者按百分位
  # percentile: 80                    # 保留 Q 值前 80% 的标记
  output:
    deleted_count: "{{result.deleted}}"
    remaining_count: "{{result.remaining}}"
```
- 该节点会永久删除 domain 中不符合 Q 条件的标记（受版本控制保护）。
- 也可输出删除的标记列表供审计。

### Q 值与注意力机制的集成
在 control.attention 中，Q 值可作为注意力权重的来源。
```
- id: attention_by_q
  type: llm
  control:
    attention:
      domain: "scene_3d"
      query: "$..[?(@.type=='vehicle')]"
      mode: "boost"
      weight_source: "q"             # 使用标记的 Q 值作为权重
      weight_scale: 2.0
```
- weight_source 可以是 q 或固定数值或另一属性路径。
- 高 Q 标记获得更高注意力权重，低 Q 标记可能被忽略。

## Q 值与记忆的集成
在存储标记到长期记忆时，可依据 Q 值决定是否存储或设置 TTL。

```
- id: store_high_q_only
  type: memory
  operation: write
  backend: "markup_store"
  value: "{{markups.scene_3d}}"
  filter:
    q_min: 0.7                       # 仅存储 Q >= 0.7 的标记
  ttl_sec: 86400                     # 低 Q 记忆更快过期
- 记忆后端可支持按 Q 值进行老化淘汰。
脚本 API 支持 Q 值
在 PMLScript 中，ctx.markup 方法支持读写 Q 值。
// 创建带 Q 值的标记
let id = await ctx.markup.create("scene_3d", {
    type: "pedestrian",
    region: {...},
    q: 0.88
});

// 查询时按 Q 值过滤和排序
let cars = await ctx.markup.query("scene_3d", "$..[?(@.type=='vehicle')]", {
    q_min: 0.6,
    sort_by: "q",
    order: "desc",
    limit: 10
});

// 更新 Q 值
await ctx.markup.update("scene_3d", id, { q: 0.95 }, version);
```

##### 性能与存储建议
- 索引：建议对 Q 值建立 B-Tree 索引，以加速 q_filter 查询。
- 压缩：定期执行 markup_compress 节点，可大幅减少长期存储中的低质量标记。
- 动态阈值：Q 值阈值可根据当前域中标记数量动态调整（例如通过 planning 节点）。

### 完整示例：基于 Q 值的 LiDAR 标记处理

```
pml:
  version: "1.5"

markup_domains:
  - id: "lidar_tracking"
    versioning: true

protocols:
  - id: "lidar_stream"
    type: mq
    topic: "lidar_detections"

workflow:
  nodes: [consumer, filter_high_q, plan]

pnodes:
  - id: consumer
    type: markup_stream
    domain: "lidar_tracking"
    source: { protocol: "lidar_stream" }
    window: { type: "sliding", duration_sec: 2.0 }
    aggregate:
      group_by: "$.id"
      functions:
        - name: "weighted_pos"
          field: "region.center"
          op: "weighted_avg"
          weight_field: "q"
    output: { tracked_objects: "{{aggregated}}" }

  - id: filter_high_q
    type: markup_query
    domain: "lidar_tracking"
    query: "$..[*]"
    q_filter: { min: 0.9 }
    output: { reliable_objects: "{{result}}" }

  - id: plan
    type: neuro_task
    script: |
      async function run(input, ctx) {
          let objects = await ctx.markup.query("lidar_tracking", "$..[*]", { q_min: 0.9 });
          // 仅使用高 Q 标记进行路径规划
          return { command: compute_path(objects) };
      }
```


## Markdown 提示扩展（Markdown Prompt Card Domain）

### 设计动机

PML 的声明式结构提供了确定性、可验证的执行模型——但 LLM-Loop 系统中，Agent 与模型的快速闭环交互往往更依赖**自然语言的灵活性与即时性**。Claude Code 的 Markdown 体系（CLAUDE.md、skills/*.md、rules/*.md）的成功证明了 Markdown 作为 Agent 交互媒介的价值。

**Markdown 提示扩展**在 PML 的 Markup 系统中新增 `type: markdown` 标记域，使 PML 在保持结构化优势的同时，获得 Markdown 提示卡片的灵活性。它被定位为 Markup 的一个域类型——与 spatial/temporal 域同等地位，但专门面向自然语言内容的存储、查询、版本管理和 LLM 上下文注入。

### 与 Anthropic Claude Code 的定位对比

```
Claude Code:
  CLAUDE.md ──→ 静态文本文件，手动编辑
  skills/*.md ──→ 静态文本文件，按需加载
  rules/*.md ──→ 静态文本文件，路径匹配加载
  共同缺陷: 无版本控制、无结构化查询、无程序化 CRUD、无自动演化

PML Markdown Markup Domain:
  markup_domains(type: markdown) ──→ 结构化标记域
  markup_crud ──→ 程序化 CRUD（Agent 可自主创建/更新提示卡）
  markup_query ──→ 结构化查询 + 全文搜索
  markup_render ──→ 自动格式化注入 LLM 上下文
  核心优势: 版本控制、程序化管理、自动演化、与 PEM 记忆联动
```

### 语法扩展

#### 标记域定义

```yaml
markup_domains:
  - id: "prompt_cards"
    type: "markdown"                    # 新增域类型
    description: "Agent prompt cards for LLM-Loop interaction"
    versioning: true                    # 每次修改自动递增版本号
    conflict_policy: "last_write_wins"
    storage:
      backend: "memory"                 # memory, git, pem_backend
      ttl_sec: 86400                    # 卡片可设置自动过期
    metadata_schema:                    # 每张卡片的结构化元数据
      author: string
      tags: [string]
      scope: "global" | "workflow" | "node" | "turn"
      priority: 1..10
      status: "draft" | "active" | "deprecated"
    indexing:
      fulltext: true                    # 启用全文搜索索引
      embedding:                        # 启用语义向量索引
        enabled: true
        model: "text-embedding-3-small"
```

#### 基本 CRUD 操作

```yaml
pnodes:
  # 创建 Markdown 提示卡
  - id: create_card
    type: markup_crud
    domain: "prompt_cards"
    operation: "create"
    markup:
      type: "prompt_card"
      metadata:
        author: "refactor-agent"
        tags: ["coding", "convention", "golang"]
        scope: "workflow"
        priority: 8
        status: "active"
      content: |
        # Go 代码规范

        ## 命名约定
        - 使用驼峰命名，导出符号首字母大写
        - 接口名以 `er` 结尾（如 `Reader`, `Writer`）

        ## 错误处理
        - 永远不要忽略 error 返回值
        - 使用 `fmt.Errorf` 包装错误上下文

        ## 测试
        - 测试文件放在同一包内，以 `_test.go` 结尾
        - 使用 table-driven tests
    output:
      schema:
        card_id: { type: string }
        version: { type: number }

  # 查询 Markdown 提示卡
  - id: find_coding_conventions
    type: markup_query
    domain: "prompt_cards"
    query: "$..[?(@.metadata.tags contains 'coding')]"   # 结构化查询
    fulltext: "错误处理 测试约定"                          # 全文搜索
    semantic: "Go language best practices"                # 语义搜索
    limit: 5
    output:
      schema:
        cards: { type: array }

  # 更新 Markdown 提示卡
  - id: update_card
    type: markup_crud
    domain: "prompt_cards"
    operation: "update"
    markup:
      id: "card_golang_001"
      version: 3                          # 乐观锁版本控制
      content: |
        # Go 代码规范 (v2)
        ## 新增：并发模式
        - 使用 context 传递取消信号
        - goroutine 生命周期必须明确
```

#### 自动注入 LLM 上下文

Markdown 提示卡的核心价值在于**自动注入 LLM 上下文**。`type: llm` 节点通过 `context.markup_sources` 声明需要注入的标记域：

```yaml
pnodes:
  - id: code_review_agent
    type: llm
    context:
      # ── 显式上下文（原有）──
      explicit:
        system: "你是代码审查专家"
        user: "{{workflow.input.task}}"

      # ── 标记域上下文（新增）──
      markup_sources:
        - domain: "prompt_cards"
          query: "$..[?(@.metadata.scope == 'workflow')]"   # 范围过滤
          priority_min: 5                                    # 仅高优先级
          status: "active"                                   # 仅活跃卡片
          format: "system_message"                           # 注入方式
          max_tokens: 2000                                   # 上下文预算
          order_by: "priority desc"                          # 优先级排序

        - domain: "prompt_cards"
          query: "$..[?(@.metadata.tags contains 'security')]"
          format: "system_message"
          prefix: "## 安全规范\n以下安全规则必须遵守：\n"

    process:
      prompt:
        user: "{{workflow.input.task}}"
```

运行时 NNOS 自动：
1. 查询匹配的 Markdown 提示卡
2. 按优先级排序并截断至 `max_tokens`
3. 格式化为 system message 注入 LLM 上下文
4. 记录注入了哪些卡片（可观测性）

#### 渲染输出

```yaml
  - id: render_conventions
    type: markup_render
    domain: "prompt_cards"
    query: "$..[?(@.metadata.tags contains 'coding')]"
    render:
      mode: "markdown_merged"            # 合并多张卡片为单个 Markdown 文档
      format: "markdown"
      template: |
        # 项目约定与规范
        生成时间: {{now}}
        ---
        {{cards}}
      group_by: "$.metadata.tags"        # 按标签分组
      sort_by: "$.metadata.priority desc"
    output:
      schema:
        merged_markdown: { type: string }
```

### LLM-Loop 快速闭环交互模式

Markdown 提示扩展的典型场景是 **LLM-Loop 快速闭环**——Agent 在多轮迭代中自主管理提示卡：

```yaml
workflow:
  name: "self_improving_agent"
  session_mode: inloop

  nodes:
    # Step 1: 从提示卡域获取当前上下文
    - id: load_context
      type: markup_query
      domain: "prompt_cards"
      query: "$..[?(@.metadata.status == 'active')]"
      order_by: "metadata.priority desc"
      output:
        context_cards: "{{result}}"

    # Step 2: 执行任务（自动注入匹配的提示卡）
    - id: execute
      type: llm
      context:
        markup_sources:
          - domain: "prompt_cards"
            query: "$..[?(@.metadata.status == 'active')]"
            format: "system_message"
            max_tokens: 2000
      process:
        prompt:
          user: "{{workflow.input.task}}"

    # Step 3: Agent 自主决定是否创建/更新提示卡
    - id: reflect_and_update
      type: llm
      process:
        prompt:
          system: |
            你刚刚完成了任务。请评估以下内容并决定是否需要更新提示卡：

            1. 你是否发现了一个新的模式或约定需要记录？
            2. 你是否遇到了一个需要避免的错误？
            3. 现有提示卡中是否有过时或不准确的内容？

            如果需要创建或更新提示卡，输出格式为：
            {
              "action": "create" | "update" | "none",
              "card": { "metadata": {...}, "content": "..." }
            }

    # Step 4: 执行提示卡变更
    - id: apply_card_changes
      type: script
      script: |
        async function run(input, ctx) {
            let decision = input.reflection;
            if (decision.action === "create") {
                await ctx.markup.create("prompt_cards", decision.card);
            } else if (decision.action === "update") {
                await ctx.markup.update("prompt_cards", decision.card.id, decision.card);
            }
            return { applied: decision.action };
        }
```

### 与 PEM 的联动

Markdown 提示卡可以作为 PEM 的"非结构化前端"：

```yaml
# Markdown 提示卡 ↔ PEM 双向同步

memory_backends:
  - id: "engineering_kb"
    mode: "deterministic"
    format: "mem_pml"

markup_domains:
  - id: "prompt_cards"
    type: "markdown"
    storage:
      backend: "pem_backend"             # 持久化到 PEM
      pem_domain: "engineering_kb"
    sync:
      direction: "bidirectional"         # PEM 结构化记录 ↔ Markdown 卡片
      mapping:
        # PEM 决策记录 → Markdown 提示卡
        pem_to_card:
          source_view: "view_decisions"
          transform: "summarize_as_convention"
        # Markdown 卡片 → PEM 知识条目
        card_to_pem:
          target_category: "context"
          auto_tag: true
```

### 设计原则

1. **Markdown 是 Markup 的一个域类型** — 不是新建体系，而是复用 `markup_crud`、`markup_query`、`markup_render`、`markup_stream` 全部现有节点类型。
2. **结构化管理非结构化内容** — 每张 Markdown 卡片都有结构化元数据（author, tags, scope, priority, status），内容是 Markdown。
3. **LLM 上下文自动注入** — `markup_sources` 声明式配置，NNOS 自动查询 → 排序 → 截断 → 注入。
4. **Agent 自主管理提示卡** — Agent 可以在 Loop 中创建、更新、弃用提示卡，实现自我进化。
5. **版本控制与审计** — 每次修改记录版本号，支持回滚和审计追踪。
6. **与 PEM 双向联动** — 结构化工程记忆 ↔ 非结构化提示卡互转。


