.. _chat_controller_module:

聊天控制器
==========

聊天控制器提供了知识库查询和对话的核心API接口，支持多种查询策略和智能对话模式。基于FastAPI开发，提供异步处理和高并发支持。

功能特性
--------

* **多种查询模式**：支持朴素查询、查询重写、多跳推理等策略
* **并发处理**：支持多数据集并发查询，提升响应速度
* **智能回答生成**：基于检索结果生成准确回答
* **流式响应**：支持实时流式对话
* **历史记录**：聊天历史管理和检索

查询模式
--------

朴素知识库查询 (mode=0)
~~~~~~~~~~~~~~~~~~~~~~~~

直接基于用户查询进行知识库检索和回答生成：

1. 计算查询向量
2. 并行执行多数据集检索
3. 基于检索结果生成回答

查询重写器查询 (mode=1)
~~~~~~~~~~~~~~~~~~~~~~

使用查询重写技术生成多个查询变体：

1. 生成查询变体
2. 并行执行多个变体查询
3. 汇总所有结果生成回答

多跳推理查询 (mode=2)
~~~~~~~~~~~~~~~~~~~~

基于多跳推理的复杂问题解决：

1. 激活多跳推理代理
2. 逐步推理和知识检索
3. 生成最终答案

朴素回答生成 (mode=3)
~~~~~~~~~~~~~~~~~~~~

直接基于LLM生成回答，不进行知识库检索。

API接口
--------

POST /api/v1/chat
~~~~~~~~~~~~~~~~~

执行知识库查询和回答生成。

请求参数：

.. list-table::
   :header-rows: 1
   :widths: 20 20 60

   * - 参数
     - 类型
     - 说明
   * - ``query``
     - str
     - 查询内容（必需）
   * - ``top_k``
     - int
     - 返回结果数量（可选，默认10）
   * - ``kb_id``
     - str
     - 知识库ID（可选）
   * - ``user_id``
     - str
     - 用户ID（必需）
   * - ``max_results``
     - int
     - 最大结果数（可选，默认20）
   * - ``mode``
     - int
     - 查询模式：0-朴素查询，1-查询重写，2-多跳推理，3-朴素回答（可选，默认0）

请求示例：

.. code-block:: json

   {
     "query": "什么是机器学习？",
     "top_k": 10,
     "kb_id": "kb001",
     "user_id": "user123",
     "max_results": 20,
     "mode": 0
   }

响应示例：

.. code-block:: json

   {
     "success": true,
     "message": "查询成功",
     "data": [
       {
         "kb_id": "kb001",
         "kb_name": "技术文档",
         "score": 0.85,
         "content": "机器学习是人工智能的一个分支...",
         "text": "",
         "source_id": ["chunk_001"]
       }
     ],
     "total_count": 1,
     "request_id": "550e8400-e29b-41d4-a716-446655440000",
     "code": 200,
     "response": "机器学习是人工智能的一个分支，通过算法让计算机从数据中学习模式并做出预测。",
     "query_info": {
       "query": "什么是机器学习？",
       "kb_id": "kb001",
       "datasets": ["ds001"],
       "kb_name": "",
       "user_id": "user123",
       "top_k": 10,
       "threshold": 0.0,
       "similarity_weight": 1.5,
       "occur_weight": 1.0,
       "use_db": false,
       "allowed_chunks_count": 100,
       "timing": {
         "embedding": 0.123,
         "dataset_query": 0.456,
         "chunk_prefilter": 0.456,
         "kg_search": 0.456,
         "chunk_scoring": 0.089,
         "format_results": 0.012,
         "total_time": 0.591
       }
     }
   }

POST /api/v1/chat/stream
~~~~~~~~~~~~~~~~~~~~~~~

流式聊天接口（待实现）。

GET /api/v1/chat/history
~~~~~~~~~~~~~~~~~~~~~~~

获取聊天历史记录（待实现）。

数据模型
--------

QueryRequest
~~~~~~~~~~~~

查询请求模型：

.. code-block:: python

   class QueryRequest(BaseModel):
       query: str
       top_k: Optional[int] = None
       kb_id: Optional[str] = None
       user_id: Optional[str] = None
       max_results: Optional[int] = 20
       mode: int = 0

QueryResult
~~~~~~~~~~~

查询结果模型：

.. code-block:: python

   class QueryResult(BaseModel):
       kb_id: str
       kb_name: str
       score: float
       content: str
       text: str
       source_id: List[str]

QueryResponse
~~~~~~~~~~~~~

查询响应模型：

.. code-block:: python

   class QueryResponse(BaseModel):
       success: bool
       message: str
       data: List[QueryResult]
       total_count: int
       query_info: Dict
       request_id: str
       code: int
       response: Optional[str] = None

查询流程
--------

1. **参数验证**：验证必需参数和数据格式
2. **模式选择**：根据mode参数选择查询策略
3. **并发查询**：对多个数据集执行并行查询
4. **结果聚合**：合并和排序查询结果
5. **回答生成**：基于检索结果生成最终回答
6. **响应返回**：返回结构化查询结果

性能优化
--------

* **向量缓存**：缓存查询向量避免重复计算
* **并发执行**：使用asyncio和ThreadPoolExecutor提升并发性能
* **批量处理**：批量数据库查询优化
* **智能过滤**：预过滤和后排序优化结果质量

错误处理
--------

* **400 Bad Request**：参数错误（如缺少query或user_id）
* **500 Internal Server Error**：服务器内部错误（如数据库连接失败）

日志记录
--------

控制器会记录详细的请求日志，包括：

* 请求ID和时间戳
* 查询参数和执行时间
* 处理结果和错误信息
* 性能统计数据

配置依赖
--------

聊天控制器依赖以下配置：

* **数据库配置**：PostgreSQL和Milvus连接信息
* **API配置**：LLM、VLM、Embedding服务配置
* **查询配置**：默认查询参数和阈值设置
