.. _rel_dedup_merge_module:

关系去重合并模块
================

``src/db_build/graph_db/rel_dedup_merge.py`` 提供了知识图谱关系的智能去重和合并功能，支持基于节点映射的关系去重和向量化的聚类合并。

核心功能
--------

* **映射关系去重**：基于节点ID映射进行关系去重
* **语义相似度判断**：使用LLM进行关系语义相似性分析
* **向量聚类合并**：基于嵌入向量的关系聚类
* **多轮迭代优化**：支持多轮去重和合并
* **大规模处理**：支持大规模关系数据的处理

主要函数
--------

dedup_relations
~~~~~~~~~~~~~~~

关系去重：

.. code-block:: python

   def dedup_relations(
       use_llm: callable,
       rel_dedup_prompt: str,
       input_path: Path,
       mapping_path: Path,
       output_path: Path
   ) -> None:

embed_rels
~~~~~~~~~~~

关系向量化：

.. code-block:: python

   def embed_rels(
       api_key: str,
       embedding_url: str,
       embedding_model: str,
       embedding_timeout: int,
       rels_input_path: Path,
       output_dir: Path,
       batch_size: int = 2000,
       num_threads: int = 8
   ) -> None:

run_rel_merge_pipeline
~~~~~~~~~~~~~~~~~~~~~

关系聚类合并流水线：

.. code-block:: python

   def run_rel_merge_pipeline(
       embedding_dir: Path,
       output_dir: Path,
       use_llm: callable,
       emb_cfg: dict,
       merge_rel_prompt: str,
       batch_size: int = 20,
       n: int = 4
   ) -> None:

run_rel_milvus_dedup
~~~~~~~~~~~~~~~~~~~~

基于Milvus的关系向量去重：

.. code-block:: python

   def run_rel_milvus_dedup(
       input_data_dir: Path,
       output_dir: Path,
       use_llm: callable,
       merge_rel_prompt: str,
       dataset: str,
       emb_cfg: dict
   ) -> None:

使用示例
--------

.. code-block:: python

   from src.db_build.graph_db.rel_dedup_merge import (
       dedup_relations, embed_rels, run_rel_merge_pipeline, run_rel_milvus_dedup
   )

   # 1. 关系去重
   dedup_relations(
       use_llm=llm_func,
       rel_dedup_prompt=dedup_prompt,
       input_path=Path("data/kg_file/relation/relations_0000.jsonl"),
       mapping_path=Path("data/kg_file/final_nodes/final_mapping.json"),
       output_path=Path("data/kg_file/dedup_rel/dedup_rel.jsonl")
   )

   # 2. 关系嵌入
   embed_rels(
       api_key="your_api_key",
       embedding_url="http://localhost:8000",
       embedding_model="text-embedding-ada-002",
       rels_input_path=Path("data/kg_file/dedup_rel/dedup_rel.jsonl"),
       output_dir=Path("data/kg_file/dedup_rel_emb")
   )

   # 3. 聚类合并
   run_rel_merge_pipeline(
       embedding_dir=Path("data/kg_file/dedup_rel_emb"),
       output_dir=Path("data/kg_file/batch_merge_rels"),
       use_llm=llm_func,
       emb_cfg=embedding_config,
       merge_rel_prompt=merge_prompt
   )

   # 4. Milvus去重
   run_rel_milvus_dedup(
       input_data_dir=Path("data/kg_file/batch_merge_rels"),
       output_data_dir=Path("data/kg_file/final_res"),
       use_llm=llm_func,
       merge_rel_prompt=merge_prompt,
       dataset="my_dataset",
       emb_cfg=embedding_config
   )
