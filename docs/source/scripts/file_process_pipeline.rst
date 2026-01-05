.. _file_process_pipeline_module:

完整数据处理流水线
==================

``scripts/file_process.py`` 提供了一套完整的端到端数据处理流水线，支持从原始文件自动构建知识库。该脚本整合了文件解析、文本分块、图谱抽取、向量化处理和数据库存储等全流程。

核心特性
--------

* **配置统一管理**：通过 ``ConfigManager`` 和 ``PipelineConfig`` 统一管理所有配置
* **模块化流水线**：支持独立运行各个处理阶段
* **自动化路径管理**：通过 ``DatasetPaths`` 自动管理文件路径
* **完整端到端处理**：一键执行从文件到知识库的全流程
* **并发处理优化**：支持多线程和异步处理提升效率
* **智能去重合并**：基于LLM的语义去重和向量聚类合并
* **多数据库集成**：同时支持PostgreSQL和Milvus向量数据库

配置管理
--------

ConfigManager 类
~~~~~~~~~~~~~~~~~

``ConfigManager`` 负责统一加载和管理所有配置文件：

* ``config/db_config.yaml``：数据库配置
* ``config/api_config.yaml``：API配置
* ``config/prompt/kg_prompt.yaml``：知识图谱提示模板配置

配置数据类
~~~~~~~~~~

脚本定义了多个配置数据类来结构化管理配置：

LLMConfig
^^^^^^^^^

大语言模型配置：

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 类型
     - 说明
   * - ``base_url``
     - str
     - LLM API 基础URL
   * - ``api_key``
     - str
     - API 密钥
   * - ``model``
     - str
     - 模型名称
   * - ``timeout``
     - int
     - 请求超时时间(秒)
   * - ``max_concurrent_requests``
     - int
     - 最大并发请求数

VLMConfig
^^^^^^^^^

视觉语言模型配置：

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 类型
     - 说明
   * - ``base_url``
     - str
     - VLM API 基础URL
   * - ``api_key``
     - str
     - API 密钥
   * - ``model``
     - str
     - 模型名称

EmbeddingConfig
^^^^^^^^^^^^^^^

嵌入模型配置：

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 类型
     - 说明
   * - ``base_url``
     - str
     - 嵌入API基础URL
   * - ``model``
     - str
     - 嵌入模型名称
   * - ``api_key``
     - str
     - API 密钥
   * - ``dim``
     - int
     - 向量维度
   * - ``batch_size``
     - int
     - 批处理大小
   * - ``num_threads``
     - int
     - 处理线程数
   * - ``timeout``
     - int
     - 请求超时时间(秒)

DatabaseConfig
^^^^^^^^^^^^^^

数据库配置：

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 类型
     - 说明
   * - ``postgres_config``
     - Dict[str, Any]
     - PostgreSQL连接配置
   * - ``postgres_batch_size``
     - int
     - PostgreSQL批处理大小
   * - ``milvus_config``
     - Dict[str, Any]
     - Milvus连接配置
   * - ``milvus_host``
     - str
     - Milvus主机地址
   * - ``milvus_port``
     - int
     - Milvus端口

GraphConfig
^^^^^^^^^^^

图谱处理配置：

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 类型
     - 说明
   * - ``batch_size``
     - int
     - 图谱处理批大小
   * - ``max_workers``
     - int
     - 最大工作线程数
   * - ``max_file_size_bytes``
     - int
     - 最大文件大小(字节)
   * - ``chunk_size``
     - int
     - 文本分块大小
   * - ``overlap``
     - int
     - 分块重叠大小
   * - ``max_batch_bytes``
     - int
     - 最大批处理字节数
   * - ``entity_schema_csv_path``
     - Optional[str]
     - 实体Schema CSV路径
   * - ``relation_schema_csv_path``
     - Optional[str]
     - 关系Schema CSV路径
   * - ``entity_schema_str``
     - str
     - 实体Schema字符串
   * - ``relation_schema_str``
     - str
     - 关系Schema字符串

ProcessingConfig
^^^^^^^^^^^^^^^^

处理参数配置：

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 类型
     - 说明
   * - ``top_k``
     - int
     - 检索返回Top-K结果
   * - ``merge_threshold``
     - float
     - 合并相似度阈值
   * - ``max_rounds``
     - int
     - 最大合并轮数
   * - ``num_topk_param``
     - int
     - Top-K参数
   * - ``nprobe``
     - int
     - Milvus搜索参数

PromptConfig
^^^^^^^^^^^^

提示模板配置：

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 类型
     - 说明
   * - ``merge_and_refine_prompt``
     - str
     - 合并精炼提示模板
   * - ``merge_prompt``
     - str
     - 合并提示模板
   * - ``dedup_template``
     - str
     - 去重提示模板
   * - ``merge_cluster_prompt``
     - str
     - 聚类合并提示模板
   * - ``dedup_rel_template``
     - str
     - 关系去重提示模板
   * - ``merge_rel_prompt``
     - str
     - 关系合并提示模板
   * - ``entity_template``
     - str
     - 实体抽取提示模板
   * - ``relation_template``
     - str
     - 关系抽取提示模板
   * - ``node_prompt``
     - str
     - 节点处理提示模板
   * - ``rel_prompt``
     - str
     - 关系处理提示模板

PipelineConfig
^^^^^^^^^^^^^^

完整的流水线配置类，封装了所有上述配置并创建相应的客户端实例。

路径管理
--------

DatasetPaths 类
~~~~~~~~~~~~~~~

``DatasetPaths`` 自动管理数据集相关的文件路径：

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 类型
     - 说明
   * - ``data_dir``
     - str
     - 数据根目录
   * - ``dataset``
     - str
     - 数据集名称

自动生成的路径包括：

* ``text_json_out``：解析后的文本JSON输出目录
* ``chunk_dir``：原始分块目录
* ``rechunk_output_dir``：重新分块输出目录
* ``graph_dir``：图谱数据目录
* 节点处理相关路径（去重、嵌入、合并等）
* 关系处理相关路径（去重、嵌入、合并等）
* 集合相关路径（chunk表、集合等）

流水线函数
----------

文件解析流水线
~~~~~~~~~~~~~~

``run_file_parse_pipeline()`` 负责文件解析：

1. 创建 ``ParserAssignment`` 实例
2. 清理旧数据
3. 执行文件分配
4. 批量解析文件（支持LLM和VLM）

Chunk处理流水线
~~~~~~~~~~~~~~~

``run_chunk_processing_pipeline()`` 执行分块处理：

1. **分块切分**：使用 ``chunk_directory()`` 将文本切分为块
2. **分块合并**：通过 ``chunks_merge_func()`` 合并相似分块
3. **重新切分**：使用 ``rechunk_by_source()`` 重新切分

图谱提取流水线
~~~~~~~~~~~~~

``run_graph_extraction_pipeline()`` 进行知识图谱提取：

1. 从JSONL加载分块数据
2. 使用 ``batch_extract_kg_from_chunks()`` 批量提取节点和关系
3. 支持实体和关系Schema配置

节点处理流水线
~~~~~~~~~~~~~

``run_node_processing_pipeline()`` 处理节点数据：

1. **节点去重**：使用LLM进行语义去重
2. **节点嵌入**：生成节点向量表示
3. **节点合并**：
   - 运行批处理合并流水线
   - 执行Milvus去重
   - 自适应合并映射

关系处理流水线
~~~~~~~~~~~~~

``run_relation_processing_pipeline()`` 处理关系数据：

1. **关系去重**：基于映射进行关系去重
2. **关系嵌入**：生成关系向量表示
3. **关系合并**：
   - 运行关系合并流水线
   - 执行关系Milvus去重

完整流水线
~~~~~~~~~~

``run_complete_pipeline()`` 提供端到端完整处理：

1. **阶段1**：文件解析
2. **阶段2**：Chunk处理
3. **阶段3**：图谱提取
4. **阶段4**：节点处理
5. **阶段5**：关系处理

支持自定义数据目录、数据集名称和原始文件路径。

知识库映射管理
--------------

``update_kb_ds_mapping()`` 函数维护知识库-数据集映射关系，保存在 ``kb_ds_mapping.json`` 文件中。

使用示例
--------

基本使用
~~~~~~~~

.. code-block:: python

   from scripts.file_process import run_complete_pipeline

   # 运行完整流水线
   run_complete_pipeline(
       data_dir="data",
       dataset="my_dataset",
       raw_file_dir=["raw_files"]
   )

自定义配置
~~~~~~~~~~

.. code-block:: python

   from scripts.file_process import create_pipeline_config, run_complete_pipeline

   # 创建自定义配置
   config = create_pipeline_config()

   # 使用自定义配置运行
   run_complete_pipeline(
       data_dir="data",
       dataset="my_dataset",
       raw_file_dir=["raw_files"],
       config=config
   )

单独运行阶段
~~~~~~~~~~~~

.. code-block:: python

   from scripts.file_process import (
       create_pipeline_config,
       run_file_parse_pipeline,
       run_chunk_processing_pipeline
   )

   config = create_pipeline_config()
   paths = DatasetPaths(data_dir="data", dataset="my_dataset")

   # 仅运行文件解析
   run_file_parse_pipeline(
       data_dir="data",
       dataset="my_dataset",
       raw_file_dir=["raw_files"],
       config=config
   )

   # 仅运行chunk处理
   run_chunk_processing_pipeline(paths, config)

命令行使用
~~~~~~~~~~

.. code-block:: bash

   # 直接运行脚本（使用默认配置）
   python scripts/file_process.py

   # 脚本会自动处理预定义的知识库和数据集

注意事项
--------

* 确保所有配置文件正确设置
* 建议在处理大规模数据时监控系统资源
* 支持断点续传，可根据需要单独运行各个阶段
* 日志输出到控制台，便于监控处理进度

依赖模块
--------

该脚本依赖以下核心模块：

* ``src.file_parsing.parser_assignment``：文件解析分配器
* ``src.db_build.graph_db.text_chunker``：文本分块器
* ``src.db_build.graph_db.chunks_merge``：分块合并器
* ``src.db_build.graph_db.graph_extraction``：图谱提取器
* ``src.db_build.graph_db.node_dedup_merge``：节点去重合并器
* ``src.db_build.graph_db.merge_mappings``：映射合并器
* ``src.db_build.graph_db.rel_dedup_merge``：关系去重合并器
* ``src.utils.llm_client``：LLM客户端
* ``src.utils.vlm_client``：VLM客户端
