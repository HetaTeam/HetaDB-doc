.. _vector_db_module:

向量数据库操作模块
===================

本模块提供完整的 **Milvus 向量数据库** 操作功能，专门用于 **HetaDB 知识库系统** 的向量数据存储和检索。支持高效的向量相似度搜索和大规模数据管理。

该模块是构建语义检索与图谱向量检索能力的核心组件，支持节点、关系和文本分块的三种向量类型。

功能特性
--------

- **多集合管理**：支持节点集合、关系集合和分块集合的自动创建和管理
- **向量相似度搜索**：基于 Milvus 的 ANN（近似最近邻）搜索，支持 Top-K 查询
- **批量数据操作**：高效的批量插入、删除和导出功能
- **自动索引优化**：根据数据规模自动选择最优索引类型
- **并发处理优化**：多线程处理提高数据导入效率
- **容错机制**：完善的异常处理和重试逻辑
- **内存管理**：控制批量大小避免内存溢出

数据格式说明
------------

模块处理三种主要的数据类型，每种都有特定的数据结构要求：

1. **节点数据格式**

   用于存储知识图谱的实体节点：

   * ``id``：节点唯一标识（主键）
   * ``nodename``：节点名称
   * ``description``：节点描述文本
   * ``type``：节点类型
   * ``subtype``：节点子类型（可选）
   * ``attr``：额外属性（JSON字符串）
   * ``embedding``：向量表示（``list[float]``）

2. **关系数据格式**

   用于存储知识图谱的实体关系：

   * ``id``：关系唯一标识（主键）
   * ``node1``：起始节点名称
   * ``node2``：目标节点名称
   * ``relation``：关系类型
   * ``type``：关系分类
   * ``description``：关系描述文本
   * ``embedding``：向量表示（``list[float]``）

3. **分块数据格式**

   用于存储文本分块：

   * ``chunk_id``：分块唯一标识（主键）
   * ``text``：原始文本内容
   * ``text_embedding``：文本向量表示
   * ``source_id``：来源文档标识
   * ``source_chunk``：来源分块信息（JSON字符串）

**重要说明**：所有向量维度必须与集合配置的 ``embedding_dim`` 参数一致，否则插入操作会失败。

核心函数
--------

连接与集合管理
~~~~~~~~~~~~~~

.. autofunction:: src.db_build.vector_db.vector_db.connect_milvus

.. autofunction:: src.db_build.vector_db.vector_db.ensure_nodes_collection

.. autofunction:: src.db_build.vector_db.vector_db.ensure_rel_collection

.. autofunction:: src.db_build.vector_db.vector_db.ensure_chunk_collection

数据操作函数
~~~~~~~~~~~~

.. autofunction:: src.db_build.vector_db.vector_db.insert_nodes_records_to_milvus

.. autofunction:: src.db_build.vector_db.vector_db.insert_relations_to_milvus

.. autofunction:: src.db_build.vector_db.vector_db.insert_chunk_batch_milvus

.. autofunction:: src.db_build.vector_db.vector_db.delete_nodes_records_from_milvus

.. autofunction:: src.db_build.vector_db.vector_db.delete_relations_from_milvus

搜索与查询
~~~~~~~~~~

.. autofunction:: src.db_build.vector_db.vector_db.search_similar_entities

.. autofunction:: src.db_build.vector_db.vector_db.search_similar_relations

.. autofunction:: src.db_build.vector_db.vector_db.get_chunk_text_by_id

集合（Collection）结构
~~~~~~~~~~~~~~~~~~~~~~~~

节点集合（Nodes Collection）
~~~~~~~~~~~~~~~~~~~~~~~~~~~

存储知识图谱实体节点的向量数据：

.. list-table::
   :header-rows: 1
   :widths: 20 15 50

   * - 字段名
     - 类型
     - 说明
   * - ``id``
     - VARCHAR(255)
     - 主键，节点唯一标识
   * - ``nodename``
     - VARCHAR(512)
     - 节点名称
   * - ``description``
     - VARCHAR(2048)
     - 节点描述文本
   * - ``type``
     - VARCHAR(64)
     - 节点类型
   * - ``subtype``
     - VARCHAR(64)
     - 节点子类型
   * - ``attr``
     - VARCHAR(65535)
     - 额外属性（JSON字符串）
   * - ``embedding``
     - FLOAT_VECTOR
     - 节点向量表示

**索引**：在 ``embedding`` 字段创建 ``IVF_FLAT`` 索引，``metric_type="IP"``。

关系集合（Relations Collection）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

存储知识图谱实体关系的向量数据：

.. list-table::
   :header-rows: 1
   :widths: 20 15 50

   * - 字段名
     - 类型
     - 说明
   * - ``id``
     - VARCHAR(255)
     - 主键，关系唯一标识
   * - ``node1``
     - VARCHAR(512)
     - 起始节点名称
   * - ``node2``
     - VARCHAR(512)
     - 目标节点名称
   * - ``relation``
     - VARCHAR(256)
     - 关系类型
   * - ``type``
     - VARCHAR(64)
     - 关系分类
   * - ``description``
     - VARCHAR(2048)
     - 关系描述文本
   * - ``embedding``
     - FLOAT_VECTOR
     - 关系向量表示

**索引**：在 ``embedding`` 字段创建 ``IVF_FLAT`` 索引，``metric_type="IP"``。

分块集合（Chunks Collection）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

存储文本分块的向量数据：

.. list-table::
   :header-rows: 1
   :widths: 20 15 50

   * - 字段名
     - 类型
     - 说明
   * - ``chunk_id``
     - VARCHAR(255)
     - 主键，分块唯一标识
   * - ``text``
     - VARCHAR(65535)
     - 原始文本内容
   * - ``text_embedding``
     - FLOAT_VECTOR
     - 文本向量表示
   * - ``source_id``
     - VARCHAR(1000)
     - 来源文档标识
   * - ``source_chunk``
     - VARCHAR(65535)
     - 来源分块信息（JSON字符串）

**索引**：在 ``text_embedding`` 字段创建 ``IVF_FLAT`` 索引，``metric_type="IP"``。

配置依赖
--------

本模块依赖以下配置项（通过 ``src.utils.load_config`` 加载）：

**Milvus 配置** （通过 ``get_milvus_config()`` 获取）：
- ``host``：Milvus 服务地址（如 ``"localhost"``）
- ``port``：Milvus 端口（如 ``19530``）

**向量维度配置**：
- ``embedding_dim``：嵌入向量维度（如 ``1024``），在创建集合时指定

**其他配置**：
- ``BATCH_SIZE``：批量插入大小（默认 1000）
- ``NUM_THREADS``：并发线程数（默认 8）

使用示例
--------

连接 Milvus 数据库：

.. code-block:: python

   from src.db_build.vector_db.vector_db import connect_milvus

   # 连接到 Milvus
   connect_milvus()

创建集合并插入数据：

.. code-block:: python

   from src.db_build.vector_db.vector_db import (
       ensure_nodes_collection,
       ensure_rel_collection,
       ensure_chunk_collection,
       insert_nodes_records_to_milvus,
       insert_relations_to_milvus,
       insert_chunk_batch_milvus
   )

   # 创建节点集合（维度为 1024）
   nodes_collection = ensure_nodes_collection("my_nodes", dim=1024)

   # 创建关系集合
   rel_collection = ensure_rel_collection("my_relations", dim=1024)

   # 创建分块集合
   chunk_collection = ensure_chunk_collection("my_chunks", embedding_dim=1024)

   # 准备节点数据
   node_records = [
       {
           "id": "node_1",
           "nodename": "人工智能",
           "description": "人工智能技术领域",
           "type": "concept",
           "embedding": [0.1, 0.2, 0.3, ...]  # 1024维向量
       }
   ]

   # 插入节点数据
   insert_nodes_records_to_milvus(nodes_collection, node_records)

向量相似度搜索：

.. code-block:: python

   from src.db_build.vector_db.vector_db import search_similar_entities

   # 搜索相似实体
   query_vector = [0.1, 0.2, 0.3, ...]  # 查询向量
   similar_entities = search_similar_entities(
       nodes_collection,
       query_vector,
       top_k=10
   )

   for entity in similar_entities:
       print(f"ID: {entity['id']}, Score: {entity['score']:.3f}")

性能与调优
----------

- **并发线程数**：``NUM_THREADS = 8`` 控制并发处理的线程数量，可根据系统性能调整
- **批量大小**：``BATCH_SIZE = 1000`` 控制每次批量插入的记录数量，平衡内存使用和吞吐量
- **向量维度**：所有向量必须与集合定义的维度完全一致，否则插入操作会失败
- **索引参数**：默认使用 ``IVF_FLAT`` 索引，``nlist=1024``，适用于中等规模数据集
- **内存管理**：通过控制批量大小避免内存溢出，支持大规模数据处理

日志与监控
----------

- **日志级别**：INFO
- **日志输出**：同时写入控制台和 ``logs/vector_db.log``
- **关键事件记录**：
  - Milvus 连接成功
  - 集合创建/复用
  - 文件加载进度
  - 插入完成与 flush

容错机制
--------

- 自动跳过空行、JSON 解析失败或关键字段缺失的记录；
- 非致命异常仅记录警告，不影响整体流程；
- 集合若已存在则直接复用，避免重复创建。

注意事项
--------

- **主键唯一性**：各集合的 ``id``/``chunk_id`` 字段必须全局唯一，否则会导致插入失败
- **向量格式**：向量必须为 Python ``list[float]`` 类型，不支持 NumPy 数组
- **维度一致性**：所有向量维度必须与集合 schema 中定义的维度完全匹配
- **字段长度限制**：
  - VARCHAR 字段有最大长度限制，超长内容会被截断
  - ``text`` 和 ``attr`` 字段支持较长内容（65535字符）
- **数据完整性**：缺失关键字段（如 ``id``、``embedding``）的记录会被跳过
- **集合重用**：已存在的集合会被直接复用，不会重复创建
- **异常处理**：非致命异常只记录警告，不中断整体处理流程
