# Prompt-Lang Prompt Pipeline Extension

## 1. 设计原则
- 表示与执行分离：Prompt 内容（ppl/ippl）独立于工作流逻辑，通过 ppcli 工具完成双向转换。
- 管线化编排：mppl/mippl 管线支持顺序、条件、复用的 Prompt 组合。
- 循环原生支持：control.loop 使 Prompt In Loop 成为一等公民，无需借用 timer + 脚本。
- 上下文注入：Intrinsic Prompts 自动注入 Agent 本体状态；Shared Prompts 支持公共片段复用。
- 安全与元数据：Header 协议声明安全规则、压缩、加密、模态等。
- 数据加速：pptensor/ppdata 为记忆和标记提供张量化存储与加速。

## 表示层增强：标准化 ppl/ippl 文件协议
### ppl 与 ippl 文件
- ppl（Public Presentation Language）：人类可读的 Prompt 片段，后缀 .ppl，不能被模型直接加载运行。
- ippl（Internal Processing Language）：模型可读的内容格式，后缀 .ippl，可以被模型直接加载运行。
- i 空间：由 ippl 对应数据组成的空间，非人可读，但模型可读。

### 引用外部 ppl/ippl 文件
在 PNode 的 process.prompt 中支持 source 字段引用外部文件：
```
pnodes:
  - id: assistant
    type: llm
    process:
      prompt:
        system:
          source: "./prompts/system.ppl"
          format: "ppl"          # ppl 或 ippl
        user: "{{workflow.input.query}}"
```
若引用 .ippl 文件，运行时直接使用；若引用 .ppl，NNOS 会自动调用 pplc 将其转换为 ippl 后再执行。

## pplc 编译工具
提供命令行工具 ppcli中集成PPLC进行 ppl ↔ ippl 转换：

```
# 基本转换
ppcli convert input.ppl --to ippl --output input.ippl
ppcli convert input.ippl --to ppl --output input.ppl

# 自动检测输入格式
ppcli convert input.ppl --to ppdata --output data.ppdata

# 批量转换目录
ppcli convert ./prompts/ --pattern "*.ppl" --to ippl --out-dir ./ippl_cache/

# 使用映射表
ppcli convert input.ppl --to ippl --mapping sentiment_labels --output output.ippl
```

## 查看支持的转换格式
```
ppcli convert --help-formats
```
- pplc 内置默认转换规则，也支持用户提供 Mapping List。

### Mapping List
在 config 中声明映射规则，用于转换过程中的词汇替换。
```
config:
  mapping_list:
    - id: "sentiment_labels"
      mapping:
        positive: 1
        negative: -1
        neutral: 0
    - id: "action_tokens"
      mapping:
        search: "[SEARCH]"
        summarize: "[SUM]"
```
转换时可通过 --mapping 参数指定使用哪组映射表。


## 编排层增强：mppl / mippl 管线节点
type: pipeline 节点，用于按顺序组合多个 ppl/ippl 片段。
```
pnodes:
  - id: reasoning_pipeline
    type: pipeline
    mode: "mppl"                # "2ppl" 或 "2ippl"
    sequences:
      - name: "preprocess"
        source: "./prompts/preprocess.ppl"
      - name: "reasoning"
        source: "./prompts/reasoning.ppl"
      - name: "format"
        source: "./prompts/format.ppl"
    variable_mapping:           # 片段间变量传递
      - input: "{{preprocess.output}}"
        output: "{{reasoning.input.user_message}}"
    conditioned:                # 条件执行
      - condition: "{{workflow.vars.needs_reasoning}}"
        execute: ["reasoning"]
    output:
      schema:
        final_prompt: { type: string }
```

- sequences 按顺序执行，每个片段可以是文件引用或内联内容。
- variable_mapping 实现片段间数据传递。
- conditioned 允许跳过某些片段。
- 最终输出组合后的完整 prompt，可供 LLM 节点使用。

## 执行层增强：原生 Prompt In Loop
在 control 中增加 loop 字段，支持原生循环执行。
```
pnodes:
  - id: iterative_refinement
    type: llm
    process:
      prompt:
        user: |
          当前迭代：{{workflow.vars.iteration}}
          前一轮输出：{{workflow.vars.last_output}}
          用户问题：{{workflow.input.query}}
    control:
      loop:
        condition: "{{output.quality < 0.8 and workflow.vars.iteration < 5}}"
        max_iterations: 10
        update_vars:
          - iteration: "{{workflow.vars.iteration + 1}}"
          - last_output: "{{output}}"
    output:
      schema:
        answer: { type: string }
        quality: { type: number }
```
- condition：表达式为真时继续循环。
- max_iterations：防止无限循环。
- update_vars：每次迭代后更新状态变量（可写回 workflow.vars 或节点输出）。
与现有 timer 节点的区别：
- timer 用于定时触发，不依赖当前节点输出。
- loop 用于根据当前执行结果决定是否继续，是真正的 Prompt In Loop。

## 上下文增强：Shared Prompts 与 Intrinsic Prompts
#### Shared Prompts
在 workflow.vars 中定义可复用的公共 Prompt 片段。
```
workflow:
  vars:
    shared_prompts:
      system_context: |
        你是专业助手。用户偏好：简洁回答，避免术语。
      global_rules: |
        回答必须基于提供的上下文，不要编造信息。
```
在 pipeline 或 LLM 节点中引用：
```
- id: use_shared
  type: pipeline
  mode: "2ppl"
  sequences:
    - name: "global"
      content: "{{workflow.vars.shared_prompts.system_context}}"
    - name: "task"
      source: "./prompts/task.ppl"
```
#### Intrinsic Prompts
在 PNode 中增加 intrinsic 字段，用于自动注入 Agent 本体状态信息。
```
pnodes:
  - id: self_aware_agent
    type: llm
    intrinsic:
      - agent_name: "{{workflow.meta.name}}"
      - agent_version: "{{workflow.meta.version}}"
      - capabilities: "{{workflow.capabilities}}"
      - current_iteration: "{{workflow.vars.iteration}}"
    process:
      prompt:
        user: "{{workflow.input.query}}"
```
运行时，NNOS 将 intrinsic 中的键值对转换为系统消息或前缀，自动添加到 prompt 中（不影响用户输入）。

## Synthetic Prompts（合成 Prompt）
```
新增 type: synthetic 节点，用于生成合成 Prompt 数据，适用于数据增强、测试用例生成等场景。
pnodes:
  - id: augment_training
    type: synthetic
    method: "mix"               # mix, augment, transform
    sources:
      - source: "./prompts/base.ppl"
        weight: 0.6
      - source: "./prompts/variant.ppl"
        weight: 0.4
    augmentation:
      - type: "paraphrase"
        model: "gpt-3.5-turbo"
        temperature: 0.8
      - type: "noise_injection"
        rate: 0.05
    output:
      candidates: 10
      format: "ppl"
```

- method=mix：从多个源中按权重选择片段组合。
- method=augment：对单个源进行扩写、改写。
- method=transform：应用映射表转换。
- 输出为 .ppl 文件或直接返回字符串。

## Header 协议支持
在 config 中增加 header 配置，用于声明模型输入前的安全、压缩、加密、模态等元数据。
```
config:
  header:
    xml: |
      <security>
        <redact_pii>true</redact_pii>
        <max_tokens>4096</max_tokens>
      </security>
    compression: "gzip"          # 或 "snappy", "none"
    encryption:
      algorithm: "aes-256-gcm"
      key_source: "$SECRET{encryption_key}"
    modalities:
      - type: "text"
        format: "utf-8"
      - type: "image"
        format: "base64"
    inference_mode: "chat"       # chat, completion, stream
    time_format: "iso8601"
    decoding: "utf-8"
```
NNOS Runtime 在调用 LLM 前解析 header 配置，进行数据解压缩、解密、PII 脱敏等操作。


## Prompt的PMLscript处理
 
在脚本中支持动态 Prompt 生成。
```
scripts:
  - id: dynamic_prompt_gen
    language: "javascript"
    prompt_script: true                 # 标识为 prompt-script
    code: |
      function generate(input, ctx) {
          let prompt = `用户问题：${input.query}`;
          if (input.history && input.history.length) {
              prompt += `\n历史记录：${input.history.join("\n")}`;
          }
          // 可调用 ctx.markup, ctx.memory 等 API
          let context = ctx.memory.read("user_preferences", input.user_id);
          if (context) prompt += `\n用户偏好：${context}`;
          return { prompt };
      }
```
在 LLM 节点中通过 prompt_script 引用：
```
pnodes:
  - id: adaptive_llm
    type: llm
    process:
      prompt_script: "dynamic_prompt_gen"
      script_input:
        query: "{{workflow.input.user_query}}"
        history: "{{workflow.vars.chat_history}}"
        user_id: "{{workflow.input.user_id}}"
```

## 数据加速：pptensor 与 ppdata
为记忆后端和标记系统引入张量化存储与高效检索。

### pptensor 后端
pptensor 是一种将标记或记忆数据转换为张量的存储后端，支持高效的向量相似度检索。
```
config:
  memory:
    backends:
      - id: "tensor_cache"
        type: "pptensor"
        endpoint: "http://tensor-server:8080"
        default_dtype: "float32"
        index_type: "ivf_flat"
    data_format: "ppdata"               # 使用 ppdata 格式序列化
```
### ppdata 序列化格式
ppdata 是 PML 统一的二进制数据格式，用于加速存储 Shared Prompts、Synthetic Prompts 等数据。
```
workflow:
  vars:
    shared_prompts:
      source: "./data/shared.ppdata"
      format: "ppdata"
```

### 完整示例：StoryThought 集成工作流
```
pml:
  version: "1.6"

meta:
  name: "storythought_demo"

config:
  header:
    xml: |
      <security><redact_pii>true</redact_pii></security>
    inference_mode: "chat"

workflow:
  input: { query: string, user_id: string }
  vars:
    shared_prompts:
      system_prompt: "你是一个有用、诚实的助手。"
    iteration: 0
    last_output: ""

  nodes: [pipeline_builder, loop_agent, synthetic_generator]

pnodes:
  - id: pipeline_builder
    type: pipeline
    mode: "2ppl"
    sequences:
      - name: "system"
        content: "{{workflow.vars.shared_prompts.system_prompt}}"
      - name: "history"
        content: "历史：{{workflow.vars.last_output}}"
      - name: "user"
        content: "{{workflow.input.query}}"
    output: { final_prompt: "{{result.final}}" }

  - id: loop_agent
    type: llm
    process:
      prompt:
        user: "{{pipeline_builder.output.final_prompt}}"
    control:
      loop:
        condition: "{{output.quality < 0.85 and workflow.vars.iteration < 3}}"
        max_iterations: 5
        update_vars:
          - iteration: "{{workflow.vars.iteration + 1}}"
          - last_output: "{{output}}"
    output:
      schema:
        answer: { type: string }
        quality: { type: number }

  - id: synthetic_generator
    type: synthetic
    method: "augment"
    sources:
      - source: "./prompts/base.ppl"
    augmentation:
      - type: "paraphrase"
        model: "gpt-3.5-turbo"
    output:
      candidates: 3
```

运行：

```
# 将 ppl 编译为 ippl
ppcli convert ./prompts/base.ppl --output ./prompts/base.ippl

# 执行工作流
ppcli run storythought_demo.pml --input '{"query":"什么是AI","user_id":"u123"}'
```

