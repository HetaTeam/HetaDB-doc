.. _optimized_kb_query_module:

优化版知识库查询服务
=================

本模块提供高性能、可扩展的知识库查询能力，整合 **Milvus 向量数据库** 与 **PostgreSQL 图谱存储**，支持：

- **预初始化连接管理**：避免重复连接开销；
- **向量预计算复用**：减少嵌入模型调用；
- **Chunk 预过滤**：利用向量相似度提前缩小图谱查询范围；
- **批量数据库查询**：减少 SQL 交互次数；
- **全局 Chunk ID 收集**：支持后续内容融合与答案生成。

适用于高并发、低延迟的智能问答系统。

核心组件
--------

.. autoclass:: optimized_kb_query.ConnectionManager
   :members:
   :undoc-members:

.. autoclass:: optimized_kb_query.OptimizedKbQuery
   :members:
   :undoc-members:

全局实例
~~~~~~~~

- ``connection_manager``：全局连接管理器（单例）
- ``optimized_kb_query``：全局查询服务实例

连接管理器（ConnectionManager）
------------------------------

负责统一管理所有外部依赖连接，确保资源高效复用。

**支持的连接类型**：

- **Milvus**：按知识库 ID（``kb_id``）自动切换 Database；
- **PostgreSQL**：按 ``kb_id`` 动态选择目标数据库；
- **嵌入模型**：OpenAI 兼容 API（vLLM），单例客户端。

**关键方法**：

- ``get_kg_collection(kb_id)``：获取知识图谱 Collection（自动加载）；
- ``get_chunk_collection(kb_id)``：获取文本块 Collection；
- ``get_embedding(prompt)``：获取文本嵌入向量；
- ``get_postgres_connection(kb_id)``：上下文管理器，安全获取 DB 连接。

> **设计亮点**：Collection 实例缓存（以 ``(kb_id, name)`` 为键），避免重复加载。

查询服务（OptimizedKbQuery）
---------------------------

### 核心查询流程

1. **向量检索阶段**：
   - 并行执行：
     - ``get_top_similar_chunks()``：获取 Top 10000 相似文本块 ID（用于后续过滤）；
     - ``query_kg_source()``：检索 Top K 图谱实体/关系；
2. **数据库查询阶段**：
   - ``query_by_res_batch_optimized()``：批量查询图谱关联的实体、关系及 Chunk ID；
3. **结果后处理**：
   - ``parse_query_results_optimized()``：结构化解析；
   - ``get_top_k_items()``：综合评分排序；
4. **答案生成**（外部）：
   - ``generate_answer_from_content()``：基于检索内容调用 LLM 生成自然语言答案。

### 核心方法说明

.. automethod:: optimized_kb_query.OptimizedKbQuery.get_top_similar_chunks

.. automethod:: optimized_kb_query.OptimizedKbQuery.query_kg_source

.. automethod:: optimized_kb_query.OptimizedKbQuery.query_by_res_batch_optimized

.. automethod:: optimized_kb_query.OptimizedKbQuery.query_chunks_by_ids_batch

### 专利数据库触发机制

``query_kg_source()`` 在返回结果时，会检查是否存在类型为 ``"database"`` 的图谱节点：

- **若存在**：直接设置 ``use_db = True``；
- **若不存在但配置了专利描述节点**：计算查询向量与专利描述向量的内积，若加权后相似度高于图谱最低分，则启用专利数据库查询（Text2SQL）。

此机制实现“语义触发”，无需硬编码关键词。

批量查询优化（query_by_res_batch_optimized）
------------------------------------------

传统方式需对每个图谱节点发起独立 SQL 查询，本模块通过 **批量查询 + 关系展开** 优化：

1. **批量获取实体/关系**：使用 ``WHERE node_id = ANY(?)`` 一次查出所有；
2. **一跳关系扩展**：对属性节点，额外查询其关联关系（限制 20 条）；
3. **全局 Chunk ID 收集**：聚合所有结果涉及的 ``chunk_id``，避免重复；
4. **结果结构**：返回列表，每项包含：
   - ``entities``：关联实体列表；
   - ``relations``：关联关系列表；
   - ``chunk_ids``：该节点涉及的 Chunk ID；
   - ``_all_chunk_ids``：**全局**所有涉及的 Chunk ID（用于后续内容查询）。

> **性能提升**：将 N 次查询降至 2–3 次，显著降低数据库负载。

辅助函数
--------

.. autofunction:: optimized_kb_query.parse_query_results_optimized

.. autofunction:: optimized_kb_query.get_top_k_items

.. autofunction:: optimized_kb_query.generate_answer_from_content

**说明**：
- ``generate_answer_from_content`` 调用 LLM 生成最终答案，自动移除 ``<think>`` 标签；
- 支持长文本截断（当前注释，可启用）；
- 使用 ``asyncio.to_thread`` 包装同步 HTTP 请求，适配异步环境。

配置依赖
--------

模块从 ``config.load_config`` 加载以下配置：

- **数据库**：
  - ``POSTGRES_CONN_CONFIG``：基础连接参数；
  - ``KNOWLEDGE_BASES[kb_id]["postgres"]["database"]``：知识库专属数据库名；
  - ``DB_TABLES``：实体/关系/Chunk 表名；
- **Milvus**：
  - ``MILVUS_CONFIG``：连接参数；
  - ``KNOWLEDGE_BASES[kb_id]["milvus"]["database"]``：Milvus Database；
  - ``KNOWLEDGE_BASES[kb_id]["milvus"]["kg_collection"]``：图谱集合名；
- **模型**：
  - ``VLLM_EMBEDDING_CONFIG``：嵌入模型 API；
  - ``CHAT_CONFIG``：回答生成 LLM API；
- **查询参数**：
  - ``QUERY_DEFAULTS``：默认 top_k、阈值、权重；
  - ``SEARCH_PARAMS``：Milvus 搜索参数（如 ``ef``）。

安全与容错
----------

- **SQL 超时**：所有 PostgreSQL 查询设置 ``statement_timeout = '30s'``；
- **空输入处理**：对空 ``chunk_ids``、``res`` 等安全返回；
- **异常捕获**：各方法独立捕获异常，避免级联失败；
- **结果状态标记**：每条结果包含 ``status`` 字段（``"success"``, ``"not_found"``, ``"error"``）。

日志与监控
----------

- 使用统一 logger：``knowledge_server``；
- 关键路径记录耗时（如批量查询）；
- 记录检索数量（实体、关系、Chunk ID）；
- 请求级别日志标记（通过 ``request_id``）。

典型调用示例
------------

.. code-block:: python

   # 1. 嵌入计算（一次）
   embedding = connection_manager.get_embedding("华为专利")

   # 2. 并行检索
   allowed_chunk_ids, _ = optimized_kb_query.get_top_similar_chunks(
       embedding=embedding, kb_id=1
   )
   kg_res, use_db = optimized_kb_query.query_kg_source(
       embedding=embedding, kb_id=1
   )

   # 3. 批量查询
   results = optimized_kb_query.query_by_res_batch_optimized(
       kg_res,
       entity_table="entities",
       relation_table="relations",
       allowed_chunk_ids=allowed_chunk_ids,
       kb_id=1
   )

   # 4. 解析与排序
   entities, relations, chunk_ids = parse_query_results_optimized(results)
   # ... 继续处理

注意事项
--------

- **知识库 ID 必须有效**：所有方法依赖 ``kb_id`` 从配置中查找数据库/集合名；
- **向量维度一致**：确保嵌入模型输出维度与 Milvus 集合定义一致；
- **Text2SQL 依赖**：专利数据库查询需额外实现 ``generate_answer()`` 函数（当前代码未包含）；
- **内存控制**：全局 Chunk ID 限制为 10,000 个，防止单次查询爆炸。
