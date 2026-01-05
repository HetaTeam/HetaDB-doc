.. _data_processing_pipeline_module:

数据处理流水线模块
==================

本模块包含了从文本分块到知识图谱构建的完整数据处理流水线，包括文本分块、图谱提取、节点去重合并、关系去重合并等核心功能。

.. toctree::
   :maxdepth: 2
   :caption: 数据处理组件

   _text_chunker_module.rst
   _chunks_merge_module.rst
   _graph_extraction_module.rst
   _node_dedup_merge_module.rst
   _rel_dedup_merge_module.rst
   _merge_mappings_module.rst
