.. _text_chunker_module:

文本分块器模块
==============

``src/db_build/graph_db/text_chunker.py`` 提供了文本分块功能，支持将长文本按照指定大小和重叠比例切分为多个块，为后续的知识图谱提取和向量化处理做准备。

核心功能
--------

* **递归文本分块**：支持按字符数进行文本分块
* **重叠处理**：支持设置相邻块之间的重叠字符数
* **批量处理**：支持并发处理多个文件
* **文件管理**：自动处理大文件的分批写入
* **重新分块**：支持基于源文档重新切分chunks

主要函数
--------

chunk_directory
~~~~~~~~~~~~~~~

对整个目录下的JSONL文件进行分块处理：

.. code-block:: python

   def chunk_directory(
       input_dir: Path,
       output_dir: Path,
       max_batch_bytes: int = 3 * 1024 * 1024 * 1024,
       max_workers: int = 16,
       chunk_size: int = 1024,
       overlap: int = 50
   ) -> None:

**参数说明：**

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 类型
     - 说明
   * - ``input_dir``
     - Path
     - 输入目录路径，包含待分块的JSONL文件
   * - ``output_dir``
     - Path
     - 输出目录路径，用于保存分块结果
   * - ``max_batch_bytes``
     - int
     - 单次批处理的最大字节数
   * - ``max_workers``
     - int
     - 并发处理的最大线程数
   * - ``chunk_size``
     - int
     - 单个文本块的最大字符数
   * - ``overlap``
     - int
     - 相邻文本块的重叠字符数

**处理流程：**

1. 扫描输入目录下的所有JSONL文件
2. 为每个文件创建对应的输出文件管理器
3. 使用线程池并发处理文件
4. 对每个文档的文本内容进行分块
5. 将分块结果写入JSONL文件

rechunk_by_source
~~~~~~~~~~~~~~~~~

基于源文档信息重新进行分块：

.. code-block:: python

   def rechunk_by_source(
       chunk_dir: Path,
       output_dir: Path,
       chunk_size: int = 1024,
       overlap: int = 50
   ) -> None:

**参数说明：**

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 类型
     - 说明
   * - ``chunk_dir``
     - Path
     - 包含原始chunk文件的目录
   * - ``output_dir``
     - Path
     - 重新分块的输出目录
   * - ``chunk_size``
     - int
     - 重新分块的大小
   * - ``overlap``
     - int
     - 重叠字符数

**处理逻辑：**

1. 读取原始chunk文件
2. 按照源文档（source_title）对chunks进行分组
3. 对每个源文档的所有chunks进行重新分块
4. 生成新的chunk文件

数据格式
--------

输入数据格式
~~~~~~~~~~~~

输入的JSONL文件每行包含一个文档对象：

.. code-block:: json

   {
     "title": "文档标题",
     "content": "文档的完整文本内容...",
     "source_url": "文档来源URL",
     "hash": "文档哈希值"
   }

输出数据格式
~~~~~~~~~~~~

输出的JSONL文件每行包含一个文本块：

.. code-block:: json

   {
     "ChunkId": "chunk_0001",
     "Text": "分块后的文本内容...",
     "SourceTitle": "文档标题",
     "SourceUrl": "文档来源URL",
     "StartChar": 0,
     "EndChar": 1024
   }

**字段说明：**

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 字段
     - 类型
     - 说明
   * - ``ChunkId``
     - string
     - 文本块的唯一标识符
   * - ``Text``
     - string
     - 分块后的文本内容
   * - ``SourceTitle``
     - string
     - 源文档标题
   * - ``SourceUrl``
     - string
     - 源文档URL
   * - ``StartChar``
     - int
     - 在原文中的起始字符位置
   * - ``EndChar``
     - int
     - 在原文中的结束字符位置

性能优化
--------

* **内存管理**：通过 ``max_batch_bytes`` 限制单次处理的数据量
* **并发处理**：使用 ``ThreadPoolExecutor`` 实现多线程并发
* **文件分片**：自动按文件大小切分输出文件，避免单个文件过大
* **进度显示**：集成 ``tqdm`` 进度条，实时显示处理进度

配置建议
--------

.. list-table::
   :header-rows: 1
   :widths: 25 20 20 35

   * - 场景
     - chunk_size
     - overlap
     - 说明
   * - 通用文档
     - 1024
     - 50
     - 适用于大多数文本处理场景
   * - 长文档
     - 2048
     - 100
     - 适用于技术文档、论文等长文本
   * - 短文档
     - 512
     - 25
     - 适用于新闻、短文等短文本

使用示例
--------

基本分块
~~~~~~~~

.. code-block:: python

   from pathlib import Path
   from src.db_build.graph_db.text_chunker import chunk_directory

   # 对目录下的所有文档进行分块
   chunk_directory(
       input_dir=Path("data/dataset/parsed_file/text_json_out"),
       output_dir=Path("data/dataset/kg_file/original_chunk"),
       chunk_size=1024,
       overlap=50,
       max_workers=8
   )

重新分块
~~~~~~~~

.. code-block:: python

   from src.db_build.graph_db.text_chunker import rechunk_by_source

   # 基于源文档重新分块
   rechunk_by_source(
       chunk_dir=Path("data/dataset/kg_file/original_chunk"),
       output_dir=Path("data/dataset/kg_file/rechunked"),
       chunk_size=1024,
       overlap=50
   )

注意事项
--------

* 输入文件必须是UTF-8编码的JSONL格式
* 输出目录会自动创建，如果已存在会覆盖
* 大文件会被自动切分，避免单个JSONL文件过大
* 处理过程中会显示详细的日志信息
* 建议在处理大规模数据时适当调整 ``max_workers`` 以避免资源耗尽
