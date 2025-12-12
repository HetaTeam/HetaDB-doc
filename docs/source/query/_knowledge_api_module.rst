.. _knowledge_api_module:

知识库查询服务 API
=================

本模块提供基于 FastAPI 的 RESTful 接口，支持多种策略的自然语言知识库查询，包括：

- **基础检索**（mode=0）：直接向量+图谱检索；
- **查询改写**（mode=1）：生成多个查询变体，融合结果；
- **多跳推理**（mode=2）：使用 Multi-Hop Agent 进行链式推理。

服务整合了 **Milvus 向量库**、**PostgreSQL 图谱表** 与 **文本到 SQL 引擎**，实现端到端的智能问答。

API 概览
--------

- **POST /api/knowledge/query**：核心查询接口（支持三种模式）
- **GET /api/knowledge/status**：健康检查接口

请求与响应均使用 JSON 格式，支持结构化错误处理与详细性能统计。

核心接口
--------

.. http:post:: /api/knowledge/query

   **功能**：执行知识库查询

   **请求体**（JSON）：

   .. autoclass:: src.api.get_ans.QueryRequest
      :members:

   **成功响应**（200 OK）：

   .. autoclass:: src.api.get_ans.QueryResponse
      :members:

   **错误响应**：
   - ``400``：缺失必要参数（``query``、``user_id``、``kb_id``）
   - ``500``：服务内部错误

   **支持的查询模式（mode）**：

   .. list-table::
      :header-rows: 1

      * - 模式值
        - 名称
        - 说明
      * - ``0``（默认）
        - 基础检索
        - 并行执行向量预过滤 + 知识图谱检索，融合结果
      * - ``1``
        - 查询改写
        - 自动生成 3 个查询变体，汇总所有结果
      * - ``2``
        - 多跳推理
        - 调用 ``MultiHopAgent`` 进行链式推理

使用示例
--------

.. code-block:: bash

   # 基础查询
   curl -X POST http://localhost:8000/api/knowledge/query \
        -H "Content-Type: application/json" \
        -d '{
              "query": "华为2023年有多少专利？",
              "kb_id": 1,
              "user_id": "user123",
              "mode": 0
            }'

   # 多跳推理
   curl -X POST http://localhost:8000/api/knowledge/query \
        -H "Content-Type: application/json" \
        -d '{
              "query": "华为在5G领域的主要竞争对手有哪些？",
              "kb_id": 1,
              "user_id": "user123",
              "mode": 2
            }'

查询流程（mode=0）
----------------

1. **输入校验**：检查 ``query``、``user_id``、``kb_id`` 是否非空；
2. **嵌入计算**：调用嵌入模型生成查询向量（仅一次）；
3. **并发检索**：
   - **Chunk 预过滤**：从 Milvus 获取 Top 1000 相似文本块；
   - **图谱检索**：从 Milvus 图谱集合获取 Top K 实体/关系；
4. **数据库查询**：根据图谱结果，从 PostgreSQL 批量查询关联文本块（已用 Chunk ID 过滤）；
5. **结果融合**：结合相似度与出现频次，综合评分排序；
6. **专利数据库查询**（可选）：若图谱结果触发专利描述，则并行调用 **Text2SQL 引擎** 生成结构化答案；
7. **生成最终回答**：将检索到的文本内容交由 LLM 生成自然语言摘要；
8. **返回结果**：包含结构化片段与自然语言回答。

> **性能优化**：关键路径均采用并发执行，总耗时 ≈ max(图谱检索, 专利查询) + 后处理。

核心函数
--------

.. autofunction:: src.api.get_ans.perform_knowledge_query

.. autofunction:: src.api.get_ans.perform_naive_kb_response

.. autofunction:: src.api.get_ans.perform_query_rewriter_kb_response

.. autofunction:: src.api.get_ans.perform_multi_hop_kb_response

配置依赖
--------

本服务依赖以下配置项（来自 ``config.load_config``）：

- ``POSTGRES_CONFIG``：PostgreSQL 连接池配置；
- ``QUERY_DEFAULTS``：查询默认参数（``top_k``, ``threshold``, 权重等）；
- ``get_kb_tables(kb_id)``：获取指定知识库的实体/关系表名。

数据库连接池
------------

- **驱动**：``postgresql+asyncpg``
- **连接池参数**：
  - ``pool_size=5``
  - ``max_overflow=10``
  - ``pool_recycle=3600``（1小时）
  - ``pool_pre_ping=True``（确保连接有效）

日志系统
--------

- **日志目录**：
  - 开发：``logs/dev/knowledge_server/``
  - 生产：``logs/prod/knowledge_server/``
- **轮转策略**：每日轮转，保留最近 7 天；
- **格式**：``YYYY-MM-DD HH:MM:SS.mmm - logger - LEVEL - [request_id] 消息``
- **自动清理**：启动时删除 7 天前的日志文件。

错误处理
--------

- 所有异常被捕获并记录；
- 返回结构化错误响应，包含 ``code``、``message``、``request_id``；
- 不泄露内部堆栈信息给客户端（生产环境安全）。

健康检查
--------

.. http:get:: /api/knowledge/status

   **功能**：检查服务是否正常运行

   **响应**：
   .. code-block:: json

      {
        "status": "healthy",
        "message": "知识库服务运行正常",
        "timestamp": "2025-12-12T10:30:45.123456",
        "environment": "dev"
      }

性能监控
--------

每个成功查询的 ``query_info`` 字段包含详细耗时统计：

.. code-block:: json

   "timing": {
     "parallel_phase": 0.45,
     "db_query": 0.12,
     "parse_results": 0.03,
     "generate_response": 1.2,
     "total_time": 1.85
   }

单位：秒，精确到毫秒。

注意事项
--------

- **必填参数**：``kb_id`` 必须对应配置中存在的知识库；
- **LLM 依赖**：``generate_answer_from_content`` 与专利查询均需可用的 LLM 服务；
- **超时控制**：SQL 查询有 30 秒超时，嵌入计算在独立线程中执行；
- **请求追踪**：每个请求分配唯一 ``request_id``，便于日志追踪；
- **模式回退**：未知 ``mode`` 值自动回退到 mode=0。
