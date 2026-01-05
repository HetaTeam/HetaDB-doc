.. _merge_mappings_module:

合并映射模块
============

``src/db_build/graph_db/merge_mappings.py`` 提供了节点合并映射的管理功能，支持自适应合并映射关系的生成和维护。

核心功能
--------

* **自适应映射**：根据合并结果自动生成映射关系
* **映射关系维护**：维护节点ID的变换关系
* **文件管理**：自动读写映射文件
* **关系传递**：支持多级映射关系的传递

主要函数
--------

merge_mappings_adaptive
~~~~~~~~~~~~~~~~~~~~~~~

自适应合并映射：

.. code-block:: python

   def merge_mappings_adaptive(
       batch_merge_dir: Path,
       final_nodes_dir: Path,
       output_dir: Path
   ) -> None:

**参数说明：**

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 类型
     - 说明
   * - ``batch_merge_dir``
     - Path
     - 批处理合并结果目录
   * - ``final_nodes_dir``
     - Path
     - 最终节点目录
   * - ``output_dir``
     - Path
     - 输出目录，用于保存映射文件

**功能说明：**

该函数读取批处理合并结果和最终节点数据，生成节点ID映射关系文件 ``final_mapping.json``，用于维护节点合并前后的ID对应关系。

映射文件格式
~~~~~~~~~~~~

.. code-block:: json

   {
     "old_node_id_1": "new_node_id_1",
     "old_node_id_2": "new_node_id_2",
     ...
   }

使用示例
--------

.. code-block:: python

   from src.db_build.graph_db.merge_mappings import merge_mappings_adaptive
   from pathlib import Path

   # 生成合并映射
   merge_mappings_adaptive(
       batch_merge_dir=Path("data/kg_file/batch_merge_nodes"),
       final_nodes_dir=Path("data/kg_file/final_nodes"),
       output_dir=Path("data/kg_file/final_nodes")
   )

注意事项
--------

* 映射文件对于关系去重至关重要
* 确保输入目录包含正确的合并结果文件
* 输出目录会自动创建如果不存在
