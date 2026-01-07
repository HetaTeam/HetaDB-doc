.. _graph_vector_module:

图谱向量化模块
==============

本模块负责将知识图谱和文本块数据转换为向量表示，通过调用嵌入API服务生成密集向量，为向量数据库存储和语义检索提供支持。

功能特性
--------

* **图谱向量化**：将图谱节点和关系转换为向量表示
* **文本块向量化**：将文本分块转换为向量表示
* **API调用封装**：统一封装OpenAI兼容的嵌入API调用
* **批量处理**：支持大规模数据的批量向量化处理
* **多线程并发**：利用多线程提高向量化效率
* **文件分片管理**：自动管理大文件的分片写入
* **容错机制**：完善的异常处理和重试逻辑

核心函数
--------

embed_graph
~~~~~~~~~~~

对图谱结构（节点和关系）进行向量化处理。

.. autofunction:: src.db_build.graph_db.graph_vector.embed_graph

embed_chunks
~~~~~~~~~~~~

对文本分块进行向量化处理。

.. autofunction:: src.db_build.graph_db.graph_vector.embed_chunks

embedding
~~~~~~~~~

调用嵌入API生成文本向量。

.. autofunction:: src.db_build.graph_db.graph_vector.embedding

使用示例
--------

图谱向量化
~~~~~~~~~~

.. code-block:: python

   from src.db_build.graph_db.graph_vector import embed_graph

   # 图谱向量化
   attr_count, triple_count = embed_graph(
       api_key="your-api-key",
       embedding_url="https://api.openai.com/v1",
       embedding_model="text-embedding-ada-002",
       embedding_timeout=30,
       attr_input_path="/path/to/nodes.jsonl",
       triples_input_path="/path/to/relations.jsonl",
       output_dir="/path/to/output",
       batch_size=2000,
       max_file_size_bytes=3*1024*1024*1024,  # 3GB
       num_threads=8
   )

文本块向量化
~~~~~~~~~~~~

.. code-block:: python

   from src.db_build.graph_db.graph_vector import embed_chunks

   # 文本块向量化
   chunk_count = embed_chunks(
       api_key="your-api-key",
       embedding_url="https://api.openai.com/v1",
       embedding_model="text-embedding-ada-002",
       embedding_timeout=30,
       chunks_file="/path/to/chunks.jsonl",
       chunk_output_dir="/path/to/output",
       batch_size=2000,
       max_file_size_bytes=3*1024*1024*1024,  # 3GB
       num_threads=8
   )

嵌入API调用
~~~~~~~~~~~~

.. code-block:: python

   from src.db_build.graph_db.graph_vector import embedding
   import numpy as np

   # 批量获取文本嵌入向量
   texts = ["人工智能技术", "机器学习算法", "深度学习模型"]
   embeddings = embedding(
       texts=texts,
       api_key="your-api-key",
       embedding_url="https://api.openai.com/v1",
       embedding_model="text-embedding-ada-002",
       embedding_timeout=30,
       max_retries=5,
       retry_delay=2
   )

   # embeddings 是 numpy 数组，形状为 (len(texts), embedding_dim)

配置参数
--------

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 默认值
     - 说明
   * - ``api_key``
     - 必需参数
     - 嵌入API的访问密钥
   * - ``embedding_url``
     - ``https://api.openai.com/v1``
     - 嵌入API的基础URL
   * - ``embedding_model``
     - ``text-embedding-ada-002``
     - 嵌入模型名称
   * - ``embedding_timeout``
     - ``30``
     - API请求超时时间（秒）
   * - ``batch_size``
     - ``2000``
     - 批量处理大小，减少API调用次数
   * - ``max_file_size_bytes``
     - ``3 GiB``
     - 输出文件的最大大小，超限后自动分片
   * - ``num_threads``
     - ``8``
     - 并行处理的线程数量
   * - ``max_retries``
     - ``5``
     - API调用失败时的最大重试次数
   * - ``retry_delay``
     - ``2``
     - 重试间隔时间（秒）

性能优化
--------

* **多线程并发**：通过线程池实现并发API调用，提高处理效率
* **批量处理**：将多个文本批量发送到API，减少网络往返次数
* **文件分片**：自动将大文件分割成多个小文件，避免单个文件过大
* **内存管理**：控制队列大小和批处理大小，避免内存溢出
* **容错重试**：自动重试失败的API调用，确保处理稳定性
* **进度监控**：实时显示处理进度和统计信息

错误处理
--------

* **API连接失败**：在开始处理前测试API连接可用性
* **请求超时**：可配置的超时时间，自动处理网络延迟
* **API限流**：指数退避重试策略，自动处理速率限制
* **网络异常**：完整的异常捕获和错误日志记录
* **数据格式错误**：跳过格式错误的记录，继续处理其他数据
* **批量失败**：返回零向量作为失败请求的替代方案

依赖项
------

* **openai**：OpenAI API客户端，用于调用嵌入服务
* **numpy**：数值计算和向量处理
* **tqdm**：进度条显示
* **threading**：多线程并发处理
* **queue**：线程安全的任务队列
* **pathlib**：路径处理
* **json**：JSON数据处理

应用场景
--------

1. **知识库向量化**：将图谱节点和关系转换为向量表示，便于存储到向量数据库
2. **文本检索预处理**：对文档分块进行向量化，为语义搜索做准备
3. **大规模数据处理**：支持海量图谱和文本数据的批量向量化处理
4. **API集成**：通过标准化的API接口集成到现有系统中
5. **离线处理**：支持离线批量处理，减少在线服务的计算压力
