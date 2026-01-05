.. _node_dedup_merge_module:

节点去重合并模块
================

``src/db_build/graph_db/node_dedup_merge.py`` 提供了知识图谱节点的智能去重和合并功能，包括基于LLM的语义去重和基于向量的聚类合并。

核心功能
--------

* **LLM语义去重**：使用大语言模型进行节点语义相似性判断
* **向量聚类合并**：基于嵌入向量的层次聚类合并
* **多轮迭代优化**：支持多轮去重和合并
* **映射关系维护**：自动维护节点ID映射关系
* **Milvus向量去重**：大规模数据的高效去重

主要函数
--------

dedup_nodes
~~~~~~~~~~~

节点语义去重：

.. code-block:: python

   def dedup_nodes(
       use_llm: callable,
       dedup_template: str,
       input_path: Path,
       output_path: Path,
       workers: int = 8
   ) -> None:

embed_nodes
~~~~~~~~~~~

节点向量化：

.. code-block:: python

   def embed_nodes(
       api_key: str,
       embedding_url: str,
       embedding_model: str,
       embedding_timeout: int,
       nodes_input_path: Path,
       output_dir: Path,
       batch_size: int = 2000,
       num_threads: int = 8
   ) -> None:

run_merge_pipeline
~~~~~~~~~~~~~~~~~~

节点聚类合并流水线：

.. code-block:: python

   def run_merge_pipeline(
       embedding_dir: Path,
       output_dir: Path,
       use_llm: callable,
       emb_cfg: dict,
       merge_cluster_prompt: str,
       batch_size: int = 20,
       n: int = 4
   ) -> None:

run_milvus_dedup
~~~~~~~~~~~~~~~~

基于Milvus的向量去重：

.. code-block:: python

   def run_milvus_dedup(
       input_data_dir: Path,
       output_data_dir: Path,
       use_llm: callable,
       merge_cluster_prompt: str,
       dataset: str,
       emb_cfg: dict
   ) -> None:

使用示例
--------

.. code-block:: python

   from src.db_build.graph_db.node_dedup_merge import (
       dedup_nodes, embed_nodes, run_merge_pipeline, run_milvus_dedup
   )

   # 1. 节点去重
   dedup_nodes(
       use_llm=llm_func,
       dedup_template=dedup_prompt,
       input_path=Path("data/kg_file/node/nodes_0000.jsonl"),
       output_path=Path("data/kg_file/dedup/dedup_node.jsonl")
   )

   # 2. 节点嵌入
   embed_nodes(
       api_key="your_api_key",
       embedding_url="http://localhost:8000",
       embedding_model="text-embedding-ada-002",
       nodes_input_path=Path("data/kg_file/dedup/dedup_node.jsonl"),
       output_dir=Path("data/kg_file/dedup_node_emb")
   )

   # 3. 聚类合并
   run_merge_pipeline(
       embedding_dir=Path("data/kg_file/dedup_node_emb"),
       output_dir=Path("data/kg_file/batch_merge_nodes"),
       use_llm=llm_func,
       emb_cfg=embedding_config,
       merge_cluster_prompt=merge_prompt
   )

   # 4. Milvus去重
   run_milvus_dedup(
       input_data_dir=Path("data/kg_file/batch_merge_nodes"),
       output_data_dir=Path("data/kg_file/final_nodes"),
       use_llm=llm_func,
       merge_cluster_prompt=merge_prompt,
       dataset="my_dataset",
       emb_cfg=embedding_config
   )
