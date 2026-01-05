.. _graph_vector_module:

图谱向量化模块
==============

本模块负责将知识图谱数据转换为向量表示，支持节点和关系的向量化处理，为图谱数据的检索和相似度计算提供基础。

功能特性
--------

* **节点向量化**：将图谱节点转换为密集向量表示
* **关系向量化**：将图谱关系转换为向量表示
* **批量处理**：支持大规模图谱数据的批量向量化
* **多模型支持**：支持不同的嵌入模型
* **相似度计算**：提供向量相似度计算功能

核心函数
--------

graph_vectorize_nodes
~~~~~~~~~~~~~~~~~~~~~

对图谱节点进行向量化处理。

.. autofunction:: src.db_build.graph_db.graph_vector.graph_vectorize_nodes

graph_vectorize_relations
~~~~~~~~~~~~~~~~~~~~~~~~~

对图谱关系进行向量化处理。

.. autofunction:: src.db_build.graph_db.graph_vector.graph_vectorize_relations

compute_similarity
~~~~~~~~~~~~~~~~~~

计算向量之间的相似度。

.. autofunction:: src.db_build.graph_db.graph_vector.compute_similarity

使用示例
--------

节点向量化
~~~~~~~~~~

.. code-block:: python

   from src.db_build.graph_db.graph_vector import graph_vectorize_nodes

   # 节点数据
   nodes = [
       {"id": "node_1", "name": "人工智能", "description": "人工智能技术"},
       {"id": "node_2", "name": "机器学习", "description": "机器学习算法"}
   ]

   # 进行向量化
   vectorized_nodes = graph_vectorize_nodes(nodes)

关系向量化
~~~~~~~~~~

.. code-block:: python

   from src.db_build.graph_db.graph_vector import graph_vectorize_relations

   # 关系数据
   relations = [
       {"head": "node_1", "relation": "包含", "tail": "node_2"},
       {"head": "node_2", "relation": "属于", "tail": "node_1"}
   ]

   # 进行向量化
   vectorized_relations = graph_vectorize_relations(relations)

相似度计算
~~~~~~~~~~

.. code-block:: python

   from src.db_build.graph_db.graph_vector import compute_similarity

   # 计算两个向量之间的余弦相似度
   similarity = compute_similarity(vector1, vector2)

配置参数
--------

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 默认值
     - 说明
   * - ``model_name``
     - ``text-embedding-ada-002``
     - 嵌入模型名称
   * - ``batch_size``
     - ``32``
     - 批量处理大小
   * - ``max_length``
     - ``512``
     - 文本最大长度
   * - ``normalize``
     - ``true``
     - 是否对向量进行L2归一化

性能优化
--------

* **GPU加速**：支持CUDA加速的向量计算
* **内存优化**：分批处理避免内存溢出
* **缓存机制**：对重复文本进行缓存避免重复计算
* **并行处理**：支持多线程并行向量化

错误处理
--------

* **模型加载失败**：自动降级到CPU模式
* **API限流**：自动重试和限流处理
* **数据格式错误**：详细的输入验证和错误提示
* **网络异常**：连接超时和重试机制

依赖项
------

* **transformers**：HuggingFace模型加载
* **torch**：PyTorch深度学习框架
* **numpy**：数值计算
* **faiss**：相似度搜索（可选）

应用场景
--------

1. **图谱检索**：基于向量相似度的图谱节点检索
2. **关系发现**：发现图谱中的相似关系模式
3. **聚类分析**：对图谱节点进行聚类分析
4. **推荐系统**：基于图谱向量的智能推荐
