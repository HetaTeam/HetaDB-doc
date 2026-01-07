.. _chunks_merge_module:

分块合并模块
============

``src/db_build/graph_db/chunks_merge.py`` 提供了文本分块的智能合并功能，通过向量相似度和LLM推理实现语义层面的分块合并和去重。

核心功能
--------

* **向量相似度合并**：基于嵌入向量计算分块相似度
* **LLM辅助推理**：使用大语言模型进行内容相关性判断
* **多轮迭代合并**：支持多轮合并以达到更好的聚类效果
* **数据库集成**：自动写入PostgreSQL和Milvus
* **进度监控**：实时显示合并进度和统计信息

主要函数
--------

main 函数
~~~~~~~~~

分块合并的主入口函数：

.. code-block:: python

   def main(
       data_dir: str,
       write_pg: bool = True,
       milvus_collections: list = None,
       postgres_config: dict = None,
       chunk_table: str = None,
       run_merge: bool = True,
       run_chunks_path: str = None,
       run_collection_name: str = None,
       run_top_k: int = 10000,
       run_nprobe: int = 16,
       run_merge_threshold: float = 0.05,
       run_max_rounds: int = 10,
       run_num_topk_param: int = 5,
       run_num_threads_param: int = 1,
       run_milvus_host: str = "127.0.0.1",
       run_milvus_port: int = 19530,
       run_target_merge_collection: str = None,
       embedding_batch_size: int = 2000,
       embedding_num_thread: int = 8,
       embedding_api_base: str = None,
       embedding_model: str = None,
       embedding_api_key: str = None,
       embedding_dim: int = 1536,
       postgres_batch_size: int = 1000,
       use_llm: callable = None,
       merge_and_refine_prompt: str = None,
       merge_prompt: str = None,
       merged_chunks_file: str = None
   ) -> tuple[list, list]:

**核心参数说明：**

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 类型
     - 说明
   * - ``run_merge_threshold``
     - float
     - 合并相似度阈值，低于此值的分块不会被合并
   * - ``run_max_rounds``
     - int
     - 最大合并轮数，避免无限循环
   * - ``run_top_k``
     - int
     - 向量检索返回的Top-K结果数
   * - ``run_nprobe``
     - int
     - Milvus向量检索的参数
   * - ``run_num_topk_param``
     - int
     - 聚类时考虑的近邻数量

处理流程
--------

1. **数据准备**：
   - 加载原始分块数据
   - 初始化嵌入向量数据库连接

2. **相似度计算**：
   - 对每个分块计算向量嵌入
   - 在向量空间中查找相似分块

3. **智能合并**：
   - 使用LLM判断内容相关性
   - 合并高度相关的分块
   - 生成合并后的新分块

4. **迭代优化**：
   - 多轮迭代合并
   - 逐步减少分块数量
   - 提升内容聚合度

5. **结果存储**：
   - 写入PostgreSQL数据库
   - 更新Milvus向量索引
   - 生成合并统计报告

数据格式
--------

输入格式
~~~~~~~~

原始分块JSONL格式：

.. code-block:: json

   {
     "chunk_id": "chunk_0001",
     "text": "分块文本内容...",
     "source": "文档来源标识"
   }

输出格式
~~~~~~~~

合并后的分块格式：

.. code-block:: json

   {
     "chunk_id": "merged_chunk_0001",
     "text": "合并后的文本内容...",
     "embedding": [0.1, 0.2, 0.3, ...],
     "source_id": "文档来源标识",
     "source_chunk": "[\"chunk_0001\", \"chunk_0002\"]"
   }

配置建议
--------

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 推荐值
     - 说明
   * - ``merge_threshold``
     - 0.05-0.1
     - 较低值会合并更多内容，较高值更保守
   * - ``max_rounds``
     - 5-15
     - 根据数据量调整，通常5-10轮足够
   * - ``top_k``
     - 5000-20000
     - 根据数据集大小调整

使用示例
--------

.. code-block:: python

   from src.db_build.graph_db.chunks_merge import main as chunks_merge_func

   # 执行分块合并
   merged_chunks, stats = chunks_merge_func(
       data_dir="data/dataset/kg_file/original_chunk",
       write_pg=True,
       milvus_collections=["chunks", "merged_chunks"],
       postgres_config=pg_config,
       chunk_table="dataset_chunks",
       run_merge=True,
       run_collection_name="merged_chunks",
       run_top_k=10000,
       run_merge_threshold=0.05,
       run_max_rounds=10,
       # ... 其他参数
   )

注意事项
--------

* 合并阈值需要根据具体数据特点调整
* 大数据集建议分批处理以控制内存使用
* 合并后的分块ID会发生变化，需要更新相关引用
