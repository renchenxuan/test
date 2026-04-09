# Dify工作流导入规范（基于 `test_end.yml`）

> 目标：把你后续编排的“无关业务工作流”也做成**可直接导入 Dify** 的 YAML。  
> 依据文件：`test_end.yml`

---

## 1. 顶层结构（必须有）

```yaml
app:
  mode: advanced-chat
  name: <workflow_name>
kind: app
version: 0.5.0
workflow:
  conversation_variables: []
  environment_variables: []
  features: ...
  graph:
    edges: [...]
    nodes: [...]
```

### 必要字段说明
- `kind: app`
- `version: 0.5.0`
- `app.mode`（示例里是 `advanced-chat`）
- `workflow.graph.nodes`、`workflow.graph.edges` 必须同时存在

---

## 2. 依赖与模型插件（建议固定写法）

`test_end.yml` 使用了 `dependencies`（marketplace 插件）。  
如果你的流程用到对应 provider，导入时应保留这段依赖声明：

- `yangyaofei/vllm`
- `langgenius/openai_api_compatible`
- `langgenius/tongyi`

---

## 3. graph 结构规范（高频踩坑点）

### 3.1 nodes 通用字段
每个 node 需要有：
- `id`（全局唯一）
- `data`（节点配置本体）
- `type: custom`（外层）
- `position` / `positionAbsolute`
- `width` / `height`

并且 `data.type` 才是业务节点类型（如 `llm` / `code` / `knowledge-retrieval`）。

### 3.2 edges 通用字段
每条 edge 建议包含：
- `id`
- `source` / `target`
- `sourceHandle` / `targetHandle`
- `type: custom`
- `data.sourceType` / `data.targetType`

---

## 4. 常用节点类型与必要参数

### 4.1 start 节点
- `data.type: start`
- `data.variables`：入参定义数组（`variable`、`type`、`required`、`default`）

适合统一接接口参数（如 token、部门、用户、知识库映射等）。

### 4.2 llm 节点
- `data.type: llm`
- `data.model.provider` / `data.model.name` / `data.model.mode`
- `data.model.completion_params`（如 `temperature`、`enable_thinking`）
- `data.prompt_template`（`role` + `text`）
- `data.context`、`data.memory`（按需）

### 4.3 code 节点
- `data.type: code`
- `data.code`
- `data.code_language`（示例为 `python3`）
- `data.variables`（输入映射：`value_selector` + `variable`）
- `data.outputs`（输出 schema）

### 4.4 if-else 节点
- `data.type: if-else`
- `data.cases[].conditions[]`
- 每个条件要有 `variable_selector`、`comparison_operator`、`value`

### 4.5 knowledge-retrieval 节点
- `data.type: knowledge-retrieval`
- `data.dataset_ids`（必填）
- `data.query_variable_selector`（必填，指定检索 query 来源）
- `data.retrieval_mode`（示例为 `multiple`）
- `data.multiple_retrieval_config`（见下一节）

### 4.6 variable-aggregator 节点
- `data.type: variable-aggregator`
- `data.variables`（聚合来源变量列表）
- `data.output_type`

### 4.7 iteration 节点（循环）
- `data.type: iteration`
- `data.start_node_id`（指向 `iteration-start` 节点 id）
- `data.iterator_selector`（迭代输入数组来源）
- `data.output_selector`（循环产出收集来源）
- `data.output_type`
- `data.is_parallel`、`data.parallel_nums`
- `data.error_handle_mode`

---

## 5. 检索节点关键参数（建议作为模板复用）

`test_end.yml` 的典型配置：
- `multiple_retrieval_config.reranking_enable: true`
- `multiple_retrieval_config.reranking_mode: weighted_score`
- `multiple_retrieval_config.reranking_model.model: gte-rerank`
- `multiple_retrieval_config.reranking_model.provider: langgenius/tongyi/tongyi`
- `weights.keyword_setting.keyword_weight: 0.3`
- `weights.vector_setting.vector_weight: 0.7`
- `weights.vector_setting.embedding_model_name: Qwen-Embedding`
- `weights.vector_setting.embedding_provider_name: langgenius/openai_api_compatible/openai_api_compatible`
- `top_k: 3`
- `score_threshold`: 有的节点是 `0.7`，有的为 `null`（按你的召回策略决定）

---

## 6. 本仓库可直接继承的“格式约束”

1. 节点数据结构统一是 **外层 `type: custom` + 内层 `data.type: 业务类型`**。  
2. 变量传递统一走 `value_selector`（code/llm/检索节点都这样映射）。  
3. 多路检索统一先路由后聚合（`if-else -> knowledge-retrieval -> variable-aggregator`）。  
4. 多意图场景统一用 `iteration`，且当前样例是：
   - `is_parallel: false`
   - `parallel_nums: 10`

---

## 7. 导入前检查清单（推荐你每次都过一遍）

- [ ] 顶层 `kind/version/app/workflow` 完整  
- [ ] `workflow.graph.nodes` 与 `edges` 都非空  
- [ ] 每个节点 `id` 唯一  
- [ ] 每条 edge 的 `source/target` 都能在 nodes 里找到  
- [ ] 每个 `knowledge-retrieval` 节点都有 `dataset_ids`  
- [ ] 每个 `knowledge-retrieval` 节点都有 `query_variable_selector`  
- [ ] 每个 `code` 节点的 `outputs` 与下游引用字段一致  
- [ ] 每个 `llm` 节点的 provider/model 在 `dependencies` 可解析  
- [ ] 若使用迭代：`start_node_id`、`iterator_selector`、`output_selector` 已配置  

---

## 8. 最小可导入骨架（可复制改名）

```yaml
app:
  description: ''
  icon: 🤖
  icon_background: '#FFEAD5'
  mode: advanced-chat
  name: demo_workflow
  use_icon_as_answer_icon: false
kind: app
version: 0.5.0
workflow:
  conversation_variables: []
  environment_variables: []
  features:
    opening_statement: ''
    retriever_resource:
      enabled: true
    suggested_questions: []
    suggested_questions_after_answer:
      enabled: true
  graph:
    nodes: []
    edges: []
```

---

如果你愿意，我下一步可以基于这份规范，直接给你再产出一个“空白但可导入”的工作流模板文件（含 start/llm/code/retrieval/answer 的最小闭环）。
