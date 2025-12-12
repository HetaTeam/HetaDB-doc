.. _graph_db_module:

图谱与分块数据入库工具
====================================

本工具用于将结构化知识图谱数据（节点、关系）和文本分块（chunks）批量导入到 **PostgreSQL** 和 **Milvus**，支持四种独立模式：

- ``chunk2pg``：分块文本 → PostgreSQL
- ``chunk2milvus``：分块向量 → Milvus
- ``graph2pg``：图谱节点/关系 → PostgreSQL
- ``graph2milvus``：图谱向量 → Milvus

同时支持在图谱向量入库后自动插入一条**专利知识库描述记录**（用于全局检索引导）。

该工具是构建多模态知识库的关键环节，通常在数据解析、嵌入生成之后运行。

使用方式
--------

通过命令行调用，支持多模式组合与配置灵活切换：

.. code-block:: bash

   # 基础用法：指定知识库 ID 和模式
   python graph_db.py --kb-id 1 --mode chunk2pg

   # 多模式同时执行
   python graph_db.py --kb-id 1 --mode chunk2pg --mode graph2milvus

   # 指定自定义数据目录
   python graph_db.py --kb-id 2 --data001 data/my_custom_data --mode graph2pg

   # 插入专利描述（仅在 graph2milvus 时生效）
   python graph_db.py --kb-id 1 --mode graph2milvus --insert-patent

命令行参数
----------

.. program:: graph_db.py

.. option:: --kb-id <ID>

   **必填**。知识库 ID，对应 ``config/api_config.yaml`` 中 ``knowledge_bases`` 的键。用于加载 PostgreSQL、Milvus、表名等配置。

.. option:: --mode <模式>

   **至少指定一个**。可选值：
   - ``chunk2pg``
   - ``chunk2milvus``
   - ``graph2pg``
   - ``graph2milvus``

   支持多次使用 ``--mode`` 指定多个任务。

.. option:: --data001 <路径>

   数据目录标识符（默认：``data/inserted_file``）。工具会在此目录下查找以下子路径：
   - ``graph/node/nodes.jsonl``
   - ``graph/relation/relations.jsonl``
   - ``graph/chunk/chunks.jsonl``
   - ``emb/graph_embedding/*.jsonl``
   - ``emb/chunk_embeding/*.jsonl``

.. option:: --insert-patent

   在 ``graph2milvus`` 完成后，自动插入一条专利数据库的全局描述向量（用于语义检索引导）。

.. option:: --patent-desc <文本>

   自定义专利描述文本。若未提供，则使用内置默认描述。

输入数据格式
------------

所有输入均为 **JSONL**（每行一个 JSON 对象）：

1. **Chunks**（``chunks.jsonl``）  
   字段：``ChunkId``, ``Text``, ``SourceTitle``, ``SourceUrl``

2. **Nodes**（``nodes.jsonl``）  
   字段：``NodeName``, ``Type``, ``SubType``, ``Description``, ``ChunkId``, ``Id``

3. **Relations**（``relations.jsonl``）  
   字段：``Node1``, ``Node2``, ``Type``, ``Relation``, ``Description``, ``ChunkId``, ``Id``

4. **Embedding 文件**（``*.jsonl``）  
   必须包含字段：``Id``, ``ChunkId``, ``embedding``（向量列表），以及对应内容字段（如 ``Text`` 或 ``Description``）。

> **注意**：所有嵌入文件必须已在前序步骤中生成（通常由嵌入模型批量计算并保存）。

功能模块说明
------------

PostgreSQL 部分
~~~~~~~~~~~~~~~

- **自动建库**：若目标数据库不存在，尝试在 ``postgres`` 库下创建（需用户有 CREATEDB 权限）；
- **自动建表**：
  - ``chunk_table``：存储分块文本；
  - ``entities_table`` 和 ``relations_table``：存储图谱节点与关系；
- **索引优化**：为 ``chunk_id`` 和 ``node_id`` 创建 B-tree 索引；
- **批量插入**：使用 ``psycopg2.extras.execute_values``，每批 1000 行，支持并发（4 线程）。

Milvus 部分
~~~~~~~~~~~

- **自动连接**：支持指定 Milvus 数据库（从配置加载）；
- **自动建 Collection**：
  - **Graph Collection**：包含节点/关系字段 + FLOAT_VECTOR（维度由 ``EMB_DIM`` 配置）；
  - **Chunk Collection**：包含文本字段 + FLOAT_VECTOR；
- **索引类型**：默认使用 ``IVF_FLAT``，相似度度量为 **内积（IP）**；
- **并发加载**：多线程读取多个 JSONL 文件并插入，带进度条。

专利描述记录（可选）
~~~~~~~~~~~~~~~~~~~

当启用 ``--insert-patent`` 且运行 ``graph2milvus`` 时，工具会：

1. 使用配置的嵌入模型（``EMB_MODEL``、``EMB_BASE_URL``、``EMB_API_KEY``）对专利描述文本生成向量；
2. 将以下记录插入图谱 Collection：
   - ``Id``: 描述文本的 SHA256
   - ``Type``: ``"database"``
   - ``Description``: 专利库说明
   - ``ChunkId``: ``"0"``
   - 其余字段留空

该记录可用于“全局语义检索”，例如用户问“这个数据库包含什么？”，可直接召回此条目。

配置依赖
--------

本工具依赖以下配置项（来自 ``config.load_config``）：

- **PostgreSQL**：``POSTGRES_CONN_CONFIG`` + 知识库专属 ``postgres_config``
- **Milvus**：``MILVUS_URI``, ``MILVUS_PORT``, 知识库专属 ``milvus_database``
- **嵌入模型**：``EMB_MODEL``, ``EMB_BASE_URL``, ``EMB_API_KEY``, ``EMB_DIM``, ``EMB_TIMEOUT``
- **表/Collection 名称**：每个知识库配置中需包含：
  - ``chunk_table``
  - ``entities_table``
  - ``relations_table``
  - ``kg_collection``
  - ``chunk_collection``

性能与容错
----------

- **断行容错**：跳过 JSON 解析失败或关键字段缺失的行，记录警告；
- **空值处理**：字段为空时插入空字符串；
- **并发控制**：使用 ``ThreadPoolExecutor``（4 线程）加速 I/O 密集型任务；
- **进度反馈**：Milvus 插入使用 ``tqdm`` 显示进度；
- **日志记录**：同时输出到控制台和 ``logs/graph_db.log``。

典型工作流
----------

1. 解析原始文件 → 生成 ``nodes.jsonl`` / ``relations.jsonl`` / ``chunks.jsonl``；
2. 调用嵌入模型 → 生成 ``graph_embedding/*.jsonl`` 和 ``chunk_embeding/*.jsonl``；
3. 运行本工具：
   .. code-block:: bash
      python graph_db.py --kb-id 1 --mode graph2pg --mode graph2milvus --mode chunk2pg --mode chunk2milvus

> **提示**：可根据资源情况分步执行，无需一次性全导入。

注意事项
--------

- 若要创建 PostgreSQL 数据库，运行用户需具备 ``CREATEDB`` 权限；
- Milvus Collection 一旦创建，**不会自动更新 schema**，若需修改字段，需手动删除重建；
- 当前所有文本字段在 PostgreSQL 中使用 **TEXT** 类型，未做长度截断（除了极个别字段做了安全截断）；
- 嵌入向量必须为 **list[float]**，且维度与 ``EMB_DIM`` 一致；
- 专利描述插入依赖 OpenAI 兼容 API，若未配置嵌入模型参数，将跳过。

日志与调试
----------

- 日志级别：``INFO``
- 日志文件：``<项目根目录>/logs/graph_db.log``
- 异常会打印完整堆栈，便于排查连接、权限、格式等问题。
