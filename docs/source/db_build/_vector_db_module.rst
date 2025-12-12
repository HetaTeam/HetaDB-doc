.. _vector_db_module:

向量数据库操作模块
================

本模块专注于将 **分块文本（chunks）** 和 **知识图谱（graph）** 的向量嵌入数据高效批量插入到 **Milvus 向量数据库**，支持多线程并发加载与自动集合（Collection）管理。

该模块是构建语义检索与图谱向量检索能力的核心组件，通常在嵌入模型生成向量文件后调用。

功能概览
--------

- **自动连接 Milvus**：支持指定 URI、端口及数据库（Database）；
- **自动建集（Collection）**：
  - **图谱集合（Graph Collection）**：存储节点/关系描述及其向量；
  - **分块集合（Chunk Collection）**：存储原始文本分块及其向量；
- **向量索引自动创建**：默认使用 ``IVF_FLAT``，相似度度量为 **内积（IP）**；
- **高并发加载**：多线程读取多个 JSONL 文件并批量插入（默认 8 线程，每批 1000 条）；
- **健壮性处理**：跳过格式错误或字段缺失的记录，记录警告日志；
- **进度可视化**：使用 ``tqdm`` 显示插入进度。

数据输入要求
------------

模块从指定数据目录（``data_dir``）读取以下嵌入文件（均为 JSONL 格式）：

1. **图谱向量目录**：``<data_dir>/emb/graph_embedding/*.jsonl``  
   每行需包含字段（大小写不敏感，支持 ``id``/``Id``）：
   - ``id`` 或 ``Id``：记录唯一标识（主键）
   - ``chunk_id`` 或 ``ChunkId``：关联的分块 ID
   - ``type`` 或 ``Type``：类型（如 ``"attr"`` 表示属性节点，``"triple"`` 表示关系）
   - ``description`` 或 ``Description``：文本描述（用于生成向量）
   - ``embedding``：向量列表（``list[float]``）
   - 其他字段（如 ``NodeName``, ``Node1``, ``Node2``, ``Relation``）根据 ``type`` 动态填充

2. **分块向量目录**：``<data_dir>/emb/chunk_embedding/*.jsonl``  
   每行需包含：
   - ``Id``：唯一 ID
   - ``ChunkId``：分块 ID
   - ``Text``：原始文本内容
   - ``embedding``：文本嵌入向量

> **注意**：所有嵌入向量维度必须与配置中的 ``EMB_DIM`` 一致。

核心函数
--------

.. autofunction:: vector_db.connect_milvus

.. autofunction:: vector_db.ensure_collection

.. autofunction:: vector_db.ensure_chunk_collection

.. autofunction:: vector_db.process_graph_to_milvus

.. autofunction:: vector_db.process_chunks_to_milvus

集合（Collection）结构
--------------------

### 图谱集合（Graph Collection）

| 字段名        | 类型          | 说明                     |
|---------------|---------------|--------------------------|
| ``id``        | VARCHAR(255)  | 主键，记录唯一 ID        |
| ``chunkid``   | VARCHAR(255)  | 关联的分块 ID            |
| ``type``      | VARCHAR(64)   | 记录类型（attr/triple）  |
| ``description``| VARCHAR(2048) | 文本描述                 |
| ``nodename``  | VARCHAR(512)  | 属性节点名称（仅 type=attr） |
| ``node1``     | VARCHAR(512)  | 关系起点（仅 type=triple） |
| ``node2``     | VARCHAR(512)  | 关系终点（仅 type=triple） |
| ``relation``  | VARCHAR(512)  | 关系类型（仅 type=triple） |
| ``embedding`` | FLOAT_VECTOR  | 向量（维度：``EMB_DIM``） |

**索引**：在 ``embedding`` 上创建 ``IVF_FLAT`` 索引，``metric_type="IP"``。

### 分块集合（Chunk Collection）

| 字段名           | 类型           | 说明               |
|------------------|----------------|--------------------|
| ``id``           | VARCHAR(255)   | 主键               |
| ``chunk_id``     | VARCHAR(255)   | 分块 ID            |
| ``text``         | VARCHAR(65535) | 原始文本内容       |
| ``text_embedding``| FLOAT_VECTOR  | 文本嵌入向量       |

**索引**：在 ``text_embedding`` 上创建 ``IVF_FLAT`` 索引，``metric_type="IP"``。

配置依赖
--------

本模块依赖以下配置项（通过 ``config.load_config`` 加载）：

- ``MILVUS_URI``：Milvus 服务地址（如 ``"localhost"``）
- ``MILVUS_PORT``：Milvus 端口（如 ``19530``）
- ``EMB_DIM``：嵌入向量维度（如 ``1024``）
- 知识库配置（通过 ``get_kb_config(kb_id)`` 获取）：
  - ``milvus_database``（可选）：Milvus 数据库名
  - ``kg_collection``：图谱集合名称
  - ``chunk_collection``：分块集合名称

使用示例
--------

在您的主流程中调用：

.. code-block:: python

   from src.db.vector_db import process_graph_to_milvus, process_chunks_to_milvus

   kb_id = 1
   data_dir = "data/processed/my_dataset"

   # 插入图谱向量
   process_graph_to_milvus(kb_id, data_dir)

   # 插入分块向量
   process_chunks_to_milvus(kb_id, data_dir)

或作为独立脚本集成到 ETL 流程中。

性能与调优
----------

- **线程数**：通过 ``NUM_THREADS = 8`` 控制并发读取线程数，可根据 I/O 能力调整；
- **批大小**：``BATCH_SIZE = 1000`` 控制每批插入记录数，平衡内存与吞吐；
- **索引参数**：当前 ``nlist=1024``，适用于百万级数据；超大规模可考虑 ``HNSW`` 或调整 ``nlist``；
- **向量维度**：确保所有嵌入向量维度与 ``EMB_DIM`` 一致，否则插入会失败。

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

- **主键唯一性**：``id`` 字段必须全局唯一，否则 Milvus 插入会失败；
- **向量类型**：必须为 Python ``list[float]``，不支持 NumPy 数组；
- **文本长度**：``description`` 和 ``text`` 字段有最大长度限制（2048 / 65535），超长内容需预处理截断；
- **Milvus 版本**：建议使用 Milvus 2.3+，以确保 Database 功能支持。
