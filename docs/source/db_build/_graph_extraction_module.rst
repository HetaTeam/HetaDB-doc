.. _graph_extraction_module:

图谱提取模块
============

``src/db_build/graph_db/graph_extraction.py`` 提供了从文本chunks中提取知识图谱的功能，支持批量提取实体节点和关系边，并支持自定义的实体和关系Schema。

核心功能
--------

* **批量图谱提取**：并发处理多个文本块
* **Schema支持**：支持自定义实体和关系模式
* **文件管理**：自动分片写入大文件
* **进度监控**：实时显示处理进度
* **错误处理**：完善的异常处理和日志记录

主要函数
--------

batch_extract_kg_from_chunks
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

从文本chunks中批量提取知识图谱：

.. code-block:: python

   def batch_extract_kg_from_chunks(
       text_chunks: list[tuple[str, str]],
       entity_schema_str: str,
       relation_schema_str: str,
       use_llm: Callable[[str], str],
       prompts: dict[str, str],
       parse_nodes: Callable[[str], list[dict]] = parse_nodes,
       get_sha256_hash: Callable[[str], str] = get_sha256_hash,
       output_dir: Path | str | None = None,
       batch_size: int = 2000,
       max_workers: int = 4,
       show_progress: bool = True,
       max_file_size_bytes: int = 3 * 1024 * 1024 * 1024
   ) -> tuple[list[dict], list[dict]]:

**参数说明：**

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 类型
     - 说明
   * - ``text_chunks``
     - list[tuple[str, str]]
     - 待处理的文本块列表，格式: [(chunk_id, text_content), ...]
   * - ``entity_schema_str``
     - str
     - 实体Schema定义字符串
   * - ``relation_schema_str``
     - str
     - 关系Schema定义字符串
   * - ``use_llm``
     - Callable[[str], str]
     - LLM调用函数，用于生成回答
   * - ``prompts``
     - dict[str, str]
     - 提示模板字典，包含: node_prompt, rel_prompt, entity_template, relation_template
   * - ``parse_nodes``
     - Callable[[str], list[dict]]
     - 节点解析函数，默认使用内置函数
   * - ``get_sha256_hash``
     - Callable[[str], str]
     - 哈希生成函数，用于ID生成
   * - ``output_dir``
     - Path | str | None
     - 输出目录路径，指定时自动保存结果，默认None
   * - ``batch_size``
     - int
     - 单批处理的数量，默认2000
   * - ``max_workers``
     - int
     - 最大并发线程数，默认4
   * - ``show_progress``
     - bool
     - 是否显示进度条，默认True
   * - ``max_file_size_bytes``
     - int
     - 单个输出文件的最大大小，默认3GB

**返回值：**

- ``all_relations``：提取的所有关系列表
- ``all_nodes``：提取的所有节点列表

load_chunks_from_jsonl
~~~~~~~~~~~~~~~~~~~~~~

从JSONL文件加载文本chunks：

.. code-block:: python

   def load_chunks_from_jsonl(input_dir: Path | str) -> list[tuple[str, str]]:

**参数说明：**

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 类型
     - 说明
   * - ``input_dir``
     - Path | str
     - 包含JSONL文件的目录路径

**返回值：** 文本chunks的列表，格式: [(chunk_id, text_content), ...]

extract_entity_from_csv
~~~~~~~~~~~~~~~~~~~~~~~

从CSV文件提取实体Schema：

.. code-block:: python

   def extract_entity_from_csv(csv_path: str) -> str:

extract_relation_from_csv
~~~~~~~~~~~~~~~~~~~~~~~~~

从CSV文件提取关系Schema：

.. code-block:: python

   def extract_relation_from_csv(csv_path: str) -> str:

Schema文件格式
--------------

实体Schema CSV格式
~~~~~~~~~~~~~~~~~~~

.. code-block:: text

   实体类型,描述,示例
   Person,自然人,张三、李四
   Organization,组织机构,清华大学、北京大学
   Location,地理位置,北京、上海

关系Schema CSV格式
~~~~~~~~~~~~~~~~~~~

.. code-block:: text

   关系类型,描述,示例
   属于,实体归属关系,张三属于清华大学
   位于,地理位置关系,清华大学位于北京
   合作,合作关系,张三和李四合作

数据格式
--------

输入数据格式
~~~~~~~~~~~~

文本chunks的格式：

.. code-block:: json

   {
     "source_chunk": "chunk_0001",
     "text": "张三在北京大学工作，是计算机专业的教授。",
     "source": "教师介绍",
     "source_id": "http://example.com/teacher"
   }

输出数据格式
~~~~~~~~~~~~

节点（nodes）格式：

.. code-block:: json

   {
     "NodeName": "张三",
     "Type": "Person",
     "SubType": "Professor",
     "Description": "计算机专业的教授",
     "ChunkId": "chunk_0001",
     "Id": "node_0001"
   }

关系（relations）格式：

.. code-block:: json

   {
     "Node1": "张三",
     "Node2": "北京大学",
     "Type": "works_at",
     "Relation": "工作于",
     "Description": "张三在北京大学工作",
     "ChunkId": "chunk_0001",
     "Id": "rel_0001"
   }

**字段说明：**

节点字段：

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 字段
     - 类型
     - 说明
   * - ``NodeName``
     - string
     - 节点名称
   * - ``Type``
     - string
     - 节点类型
   * - ``SubType``
     - string
     - 节点子类型
   * - ``Description``
     - string
     - 节点描述
   * - ``ChunkId``
     - string
     - 来源文本块ID
   * - ``Id``
     - string
     - 节点唯一标识符

关系字段：

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 字段
     - 类型
     - 说明
   * - ``Node1``
     - string
     - 起始节点名称
   * - ``Node2``
     - string
     - 目标节点名称
   * - ``Type``
     - string
     - 关系类型
   * - ``Relation``
     - string
     - 关系名称
   * - ``Description``
     - string
     - 关系描述
   * - ``ChunkId``
     - string
     - 来源文本块ID
   * - ``Id``
     - string
     - 关系唯一标识符

处理流程
--------

1. **准备阶段**：
   - 加载文本chunks
   - 准备Schema字符串
   - 初始化文件管理器

2. **批量处理**：
   - 将chunks按batch_size分组
   - 使用线程池并发处理每个batch
   - 对每个chunk调用LLM进行图谱提取

3. **结果收集**：
   - 收集所有提取的节点和关系
   - 写入JSONL文件（自动分片）
   - 返回汇总结果

性能优化
--------

* **并发处理**：使用 ``ThreadPoolExecutor`` 实现多线程并发
* **批处理**：将chunks分组处理，减少LLM调用次数
* **文件分片**：自动按文件大小切分，避免单个文件过大
* **内存管理**：分批处理，避免内存溢出
* **进度监控**：集成 ``tqdm`` 进度条

配置建议
--------

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 推荐值
     - 说明
   * - ``batch_size``
     - 10-50
     - 根据LLM上下文长度调整
   * - ``max_workers``
     - 1-8
     - 根据系统CPU核心数调整，默认4
   * - ``max_file_size_bytes``
     - 3GB
     - 根据磁盘空间和处理需求调整

使用示例
--------

基本使用
~~~~~~~~

.. code-block:: python

   from pathlib import Path
   from src.db_build.graph_db.graph_extraction import batch_extract_kg_from_chunks

   # 加载文本chunks
   text_chunks = load_chunks_from_jsonl(Path("data/dataset/kg_file/rechunked"))

   # 批量提取图谱
   all_relations, all_nodes = batch_extract_kg_from_chunks(
       text_chunks=text_chunks,
       entity_schema_str="Person: 自然人\\nOrganization: 组织机构",
       relation_schema_str="works_at: 工作关系\\nlocated_in: 位置关系",
       use_llm=my_llm_function,
       prompts=my_prompts_dict,
       output_dir=Path("data/dataset/kg_file"),
       batch_size=20,
       max_workers=8
   )

带Schema文件
~~~~~~~~~~~~

.. code-block:: python

   from src.db_build.graph_db.graph_extraction import (
       extract_entity_from_csv,
       extract_relation_from_csv
   )

   # 从CSV文件加载Schema
   entity_schema = extract_entity_from_csv("config/entity_schema.csv")
   relation_schema = extract_relation_from_csv("config/relation_schema.csv")

   # 使用Schema进行提取
   all_relations, all_nodes = batch_extract_kg_from_chunks(
       text_chunks=text_chunks,
       entity_schema_str=entity_schema,
       relation_schema_str=relation_schema,
       # ... 其他参数
   )

注意事项
--------

* 需要确保LLM服务可用且响应格式正确
* Schema字符串格式需要符合LLM提示模板的要求
* 输出目录会自动创建，建议使用空目录
* 处理大规模数据时注意内存和磁盘空间使用
* 支持断点续传，可根据需要调整batch_size
