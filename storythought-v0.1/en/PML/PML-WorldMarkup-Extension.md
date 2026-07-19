# PML World Markup Extension

## Scene-Markup

Design Principles (New)
- **Single Responsibility**: Markup CRUD, query, render, transform, and stream processing use independent node types.
- **Real-Time Priority**: Support markup stream subscription and window aggregation, adapting to sensor data.
- **Unified Syntax**: Markup queries uniformly use JSONPath syntax, consistent with data binding expressions.
- **Event-Driven**: Markup changes can trigger workflow nodes, avoiding polling.
- **Composable**: Markup can be persisted as memory and deeply integrated with scripts and LLMs.

---
### Markup Domain Definition

```
markup_domains:
  - id: "scene_3d"
    type: "spatial"                     # spatial, temporal, spatiotemporal
    coordinate_system: "world"
    bounds: [-100, -100, -50, 100, 100, 50]
    versioning: true                    # Enable optimistic locking version control (v1.4+)
    conflict_policy: "last_write_wins"  # last_write_wins, merge, raise_error
    storage:
      backend: "memory"                 # memory, postgis, redis
      ttl_sec: 3600                     # Auto-expiry time for markups
```

## Markup Node Types

### type: markup_crud (Create, Read, Update, Delete)
```
pnodes:
  - id: manage_car
    type: markup_crud
    domain: "scene_3d"
    operation: "create"                 # create, read, update, delete
    markup:
      id: "car_001"
      type: "vehicle"
      region: { type: "bbox", coordinates: [1,2,3,4,5,6] }
      attributes: { speed: 10 }
```

### type: markup_query
```
pnodes:
  - id: find_cars
    type: markup_query
    domain: "scene_3d"
    query: "$..[?(@.type=='vehicle')]"   # JSONPath syntax (unified)
    limit: 10
    output:
      schema:
        results: { type: array }
```

### type: markup_render
```
pnodes:
  - id: render_view
    type: markup_render
    domain: "scene_3d"
    query: "$..[?(@.type=='vehicle')]"
    render:
      mode: "image"                      # image, pointcloud, mesh
      camera: { position: [0,0,10], look_at: [0,0,0] }
      style: "colored_bbox"
      output_format: "png"
```

### type: markup_transform (Coordinate Transform)
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
```

### type: markup_stream (Real-Time Markup Stream - v1.4+)
Subscribe to markup domain change events or ingest markups from external streams.

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
      protocol: "lidar_stream"
      format: "json"
    window:
      type: "sliding"
      duration_sec: 5.0
      step_sec: 1.0
    output:
      schema:
        aggregated: { type: array }
```

### Markup Change Events

Markup domains support change events; other nodes can subscribe.

```
markup_domains:
  - id: "scene_3d"
    events:
      on_create: "markup_created_channel"
      on_update: "markup_updated_channel"
      on_delete: "markup_deleted_channel"
```

### Markup Attention Enhancement

#### Attention Heatmap Output
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

### Markup & Memory Integration

Allow persisting markup domains as long-term memory or restoring from memory.
```
memory_backends:
  - id: "markup_store"
    type: markup_adapter
    backend: "qdrant"
    collection: "scene_markups"
```

### Script API Integration (PMLScript)
Inject `markup` object into `ctx` in PMLScript.
```
// In type: script node
async function run(input, ctx) {
    let cars = await ctx.markup.query("scene_3d", "$..[?(@.type=='vehicle')]");
    let car = await ctx.markup.read("scene_3d", "car_001");
    let newId = await ctx.markup.create("scene_3d", {
        type: "pedestrian",
        region: { type: "point", coordinates: [1,2,0] }
    });
    await ctx.markup.update("scene_3d", "car_001", { speed: 20 }, version);
    await ctx.markup.delete("scene_3d", "car_001");
    return { car_count: cars.length };
}
```

## Q-Markup

### Q-Value Definition

Q-values represent the quality/confidence of markup data, enabling filtering and attention-weighting based on data reliability.

### Q-Value & Attention Mechanism Integration

Q-values integrate with the attention mechanism, allowing LLM nodes to focus on high-Q markup regions.

## Q-Value & Memory Integration

High-Q markups can be persisted to memory for long-term reliable scene understanding.

### Complete Example: Q-Value Based LiDAR Markup Processing

A LiDAR tracking pipeline using Q-values for filtering and weighted aggregation.

```
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
          // Only use high-Q markups for path planning
          return { command: compute_path(objects) };
      }
```


## Markdown Prompt Card Extension (Markdown Prompt Card Domain)

### Design Motivation

PML's declarative structure provides deterministic, verifiable execution models — but in LLM-Loop systems, rapid closed-loop interaction between Agent and model often relies more on **natural language flexibility and immediacy**. Claude Code's success with its Markdown system (CLAUDE.md, skills/*.md, rules/*.md) proves the value of Markdown as an Agent interaction medium.

The **Markdown Prompt Card Extension** adds `type: markdown` markup domain to PML's Markup system, giving PML the flexibility of Markdown prompt cards while maintaining its structural advantages. It is positioned as a Markup domain type — on par with spatial/temporal domains — but specifically oriented toward natural language content storage, querying, version management, and LLM context injection.

### Comparison with Anthropic Claude Code

```
Claude Code:
  CLAUDE.md ──→ Static text file, manual editing
  skills/*.md ──→ Static text file, on-demand loading
  rules/*.md ──→ Static text file, path-matched loading
  Common flaw: No version control, no structured query, no programmatic CRUD, no auto-evolution

PML Markdown Markup Domain:
  markup_domains(type: markdown) ──→ Structured markup domain
  markup_crud ──→ Programmatic CRUD (Agent can autonomously create/update cards)
  markup_query ──→ Structured query + full-text search
  markup_render ──→ Auto-format for LLM context injection
  Core advantages: Version control, programmatic management, auto-evolution, PEM memory linkage
```

### Syntax Extension

#### Markup Domain Definition

```yaml
markup_domains:
  - id: "prompt_cards"
    type: "markdown"                    # New domain type
    description: "Agent prompt cards for LLM-Loop interaction"
    versioning: true
    conflict_policy: "last_write_wins"
    storage:
      backend: "memory"
      ttl_sec: 86400
    metadata_schema:
      author: string
      tags: [string]
      scope: "global" | "workflow" | "node" | "turn"
      priority: 1..10
      status: "draft" | "active" | "deprecated"
    indexing:
      fulltext: true
      embedding:
        enabled: true
        model: "text-embedding-3-small"
```

#### Basic CRUD Operations

```yaml
pnodes:
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
        # Go Code Conventions
        ## Naming
        - Use camelCase, export capitalized symbols
        - Interface names end with `er` (e.g., `Reader`, `Writer`)
        ## Error Handling
        - Never ignore error return values
        - Use `fmt.Errorf` to wrap error context
        ## Testing
        - Test files in same package, suffix `_test.go`
        - Use table-driven tests

  - id: find_coding_conventions
    type: markup_query
    domain: "prompt_cards"
    query: "$..[?(@.metadata.tags contains 'coding')]"
    fulltext: "error handling testing conventions"
    semantic: "Go language best practices"
    limit: 5
```

#### Auto-Inject into LLM Context

```yaml
pnodes:
  - id: code_review_agent
    type: llm
    context:
      markup_sources:
        - domain: "prompt_cards"
          query: "$..[?(@.metadata.scope == 'workflow')]"
          priority_min: 5
          status: "active"
          format: "system_message"
          max_tokens: 2000
          order_by: "priority desc"
```

At runtime, NNOS automatically: queries matching Markdown cards, sorts by priority and truncates to `max_tokens`, formats as system message for LLM context injection, and logs which cards were injected (observability).

### LLM-Loop Rapid Closed-Loop Interaction Pattern

A typical use case: Agent autonomously manages prompt cards across multiple iterations:

```yaml
workflow:
  name: "self_improving_agent"
  session_mode: inloop

  nodes:
    - id: load_context      # Query active prompt cards
      type: markup_query
      domain: "prompt_cards"
      query: "$..[?(@.metadata.status == 'active')]"

    - id: execute            # Execute task (auto-inject matching cards)
      type: llm
      context:
        markup_sources:
          - domain: "prompt_cards"
            format: "system_message"
            max_tokens: 2000

    - id: reflect_and_update # Agent self-reflects → decides create/update/none
      type: llm
      # Outputs: { action: "create"|"update"|"none", card: {...} }

    - id: apply_card_changes # Execute card changes
      type: script
      # ctx.markup.create() or ctx.markup.update()
```

### Integration with PEM

Markdown prompt cards serve as the "unstructured frontend" for PEM:

```yaml
markup_domains:
  - id: "prompt_cards"
    type: "markdown"
    storage:
      backend: "pem_backend"
      pem_domain: "engineering_kb"
    sync:
      direction: "bidirectional"         # PEM records ↔ Markdown cards
      mapping:
        pem_to_card:
          source_view: "view_decisions"
          transform: "summarize_as_convention"
        card_to_pem:
          target_category: "context"
          auto_tag: true
```

### Design Principles

1. **Markdown is a Markup domain type** — not a new system; reuses all existing node types.
2. **Structured management of unstructured content** — every card has structured metadata (author, tags, scope, priority, status); content is Markdown.
3. **LLM context auto-injection** — `markup_sources` declarative config; NNOS auto query → sort → truncate → inject.
4. **Agent autonomously manages cards** — Agent can create, update, deprecate cards in loops for self-evolution.
5. **Version control & audit** — each modification records version number, supports rollback and audit.
6. **Bidirectional PEM linkage** — structured engineering memory ↔ unstructured prompt cards.
