.. _system_config_module:

系统配置说明
============

本配置文件定义了知识库构建与查询系统的全局参数，涵盖 **文本分块**、 **知识图谱构建**、 **向量嵌入处理** 和 **查询默认行为** 四大模块，以及数据库连接与执行模式设置。

所有配置项均采用分层结构，便于按需调整。

参数配置（parameter）
--------------------

.. _chunk_config:

文本分块配置（chunk_config）
~~~~~~~~~~~~~~~~~~~~~~~~~~~

用于控制原始文本的分块策略与并行处理能力。

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 默认值
     - 说明
   * - ``chunk_size``
     - ``1024``
     - 单个文本块的最大字符数（含 overlap）
   * - ``overlap``
     - ``50``
     - 相邻文本块的重叠字符数，用于保持上下文连贯性
   * - ``max_batch_bytes``
     - ``3221225472`` 
     - 单次分块任务的最大输入数据量（字节），超限将自动分批
   * - ``max_workers``
     - ``16``
     - 分块处理的最大并发线程数

.. _graph_config:

知识图谱构建配置（graph_config）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

控制从结构化数据（如 CSV）构建知识图谱的处理参数。

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 默认值
     - 说明
   * - ``entity_schema_csv_path``
     - ``null``
     - 实体 Schema 文件路径（当前未使用，预留扩展）
   * - ``relation_schema_csv_path``
     - ``null``
     - 关系 Schema 文件路径（当前未使用，预留扩展）
   * - ``batch_size``
     - ``2000``
     - 单次图谱节点/关系处理的批量大小
   * - ``max_workers``
     - ``16``
     - 图谱构建的最大并发线程数
   * - ``max_file_size_bytes``
     - ``3221225472`` 
     - 单个输出 JSONL 文件的最大大小，超过后自动切分新文件

.. _vector_config:

向量嵌入配置（vector_config）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

用于控制向量生成与写入 Milvus 的性能与容错策略。

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 默认值
     - 说明
   * - ``batch_size``
     - ``2000``
     - 单次向量计算或插入的批量大小
   * - ``max_file_size_bytes``
     - ``3221225472`` 
     - 向量 JSONL 文件的最大大小，用于分片存储
   * - ``num_threads``
     - ``8``
     - 向量处理的并发线程数（用于文件读写与插入）
   * - ``max_retries``
     - ``5``
     - 向量服务调用失败时的最大重试次数
   * - ``retry_delay``
     - ``2`` 
     - 重试之间的延迟时间（秒）

.. _query_defaults:

查询默认参数（query_defaults）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

为知识库检索提供默认行为，可被 API 请求覆盖。

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 默认值
     - 说明
   * - ``top_k``
     - ``10``
     - 向量检索返回的 top-K 结果数
   * - ``threshold``
     - ``0.7``
     - 向量相似度阈值，低于此值的结果将被过滤
   * - ``max_chunks``
     - ``100``
     - 最终返回的最大文本块数量
   * - ``similarity_weight``
     - ``1.5``
     - 相似度分数在综合评分中的权重
   * - ``occur_weight``
     - ``1.0``
     - 实体/关系出现频次在综合评分中的权重

API配置（api_config）
--------------------

.. _llm_config:

LLM配置（llm）
~~~~~~~~~~~~~

大语言模型API配置：

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 默认值
     - 说明
   * - ``base_url``
     - ``http://localhost:8000/v1``
     - LLM API基础URL
   * - ``api_key``
     - ``EMPTY``
     - API密钥
   * - ``model``
     - ``qwen3_32b``
     - 模型名称
   * - ``timeout``
     - ``120``
     - 请求超时时间(秒)
   * - ``max_concurrent_requests``
     - ``10``
     - 最大并发请求数

.. _vlm_config:

VLM配置（vlm）
~~~~~~~~~~~~~

视觉语言模型API配置：

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 默认值
     - 说明
   * - ``base_url``
     - ``http://localhost:8000/v1``
     - VLM API基础URL
   * - ``api_key``
     - ``EMPTY``
     - API密钥
   * - ``model``
     - ``qwen-vl-plus``
     - 模型名称

.. _embedding_config:

嵌入模型配置（embedding_api）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

向量嵌入模型API配置：

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 默认值
     - 说明
   * - ``base_url``
     - ``http://localhost:8000/v1``
     - 嵌入API基础URL
   * - ``model``
     - ``text-embedding-3-small``
     - 嵌入模型名称
   * - ``api_key``
     - ``EMPTY``
     - API密钥
   * - ``dim``
     - ``1536``
     - 向量维度
   * - ``batch_size``
     - ``100``
     - 批处理大小
   * - ``num_threads``
     - ``4``
     - 处理线程数
   * - ``timeout``
     - ``60``
     - 请求超时时间(秒)

FastAPI配置（fastapi_config）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Web服务配置：

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 默认值
     - 说明
   * - ``env``
     - ``dev``
     - 运行环境(dev/prod)
   * - ``host``
     - ``127.0.0.1``
     - 服务绑定地址
   * - ``port``
     - ``8000``
     - 服务端口
   * - ``reload``
     - ``true``
     - 开发模式自动重载
   * - ``log_level``
     - ``info``
     - 日志级别

数据库配置（db_config）
----------------------


.. _csv_mode:

表格节点插入（csv_mode）
~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: yaml

   csv_mode: true

- 若为 ``true``，在执行 ``graph2milvus`` 模式后，**自动插入专利数据库描述记录**（用于 Text2SQL 触发）；
- 若为 ``false``，则跳过此步骤。

> **注意**：实际代码中该字段名为 ``csv_mode``，但功能与“专利描述”相关，命名略有歧义。

数据库连接配置
~~~~~~~~~~~~~~

.. _postgresql_config:

PostgreSQL
^^^^^^^^^^

.. code-block:: yaml

   PostgreSQL:
     host: localhost
     port: 5432
     user: postgres
     password: postgres
     options: "-c search_path=public"

用于连接 PostgreSQL 数据库，支持多知识库通过不同 database 隔离。

.. _milvus_config:

Milvus
^^^^^^

.. code-block:: yaml

   Milvus:
     uri: localhost
     port: 19530
     sentence_mode: false

- ``uri``：Milvus 服务地址；
- ``port``：gRPC 端口（默认 19530）；
- ``sentence_mode``：是否启用句子级向量存储（当前未启用）。

注释项说明
----------

配置文件中包含以下被注释的可选项，可根据需要取消注释：

- ``data_dir``：指定数据输入目录（默认由调用方传入）；
- ``patent_desc``：自定义专利描述文本（若为 ``null``，则使用内置默认描述）。

配置最佳实践
------------

- **生产环境**：建议降低 ``max_workers`` 和 ``num_threads`` 避免资源耗尽；
- **大文件处理**：确保 ``max_file_size_bytes`` 与磁盘 I/O 能力匹配；
- **查询调优**：根据业务需求调整 ``similarity_weight`` 与 ``occur_weight`` 的平衡；
- **安全**：生产环境中应通过环境变量或密钥管理服务注入数据库密码，而非明文写入配置。
