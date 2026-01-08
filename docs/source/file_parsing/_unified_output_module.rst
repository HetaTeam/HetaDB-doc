.. _unified_output_module:

统一输出模块
=====================================

不同格式的解析器会输出统一的多个 ``*_middle.json`` 作为中间文件，本模块将这些中间文件整合为标准化的统一输出格式。

输出结构清晰，支持 **自动滚动写入** （单个 JSONL 文件超过 3 GB 时自动切换到新文件），确保大体量数据处理的稳定性和可扩展性。

功能特点
--------

- 从指定目录读取所有 ``*_middle.json`` 中间文件；
- 提取 ``meta`` 与 ``json_content`` 字段，封装为统一文档格式；
- 自动滚动写入 JSONL（默认阈值：3 GB）；
- 完成后自动清理临时中间文件目录；

输出目录结构示例
------------------

::

  output/
  ├── doc_20251212103045_123456.jsonl
  ├── doc_20251212114520_789012.jsonl
  └── ...（按时间戳滚动生成）

使用示例
--------

.. code-block:: python

   from src.file_parsing.unified_output import process_middle_files

   temp_output = Path("data/my_dataset/temp_middle")
   jsonls_dir = Path("data/my_dataset/jsonl_output")
   process_middle_files(temp_output, jsonls_dir)

模块参考
--------

常量
~~~~

``ROLL_SIZE``  
   单个 JSONL 文件的最大大小（默认：3 GB = ``3 * 1024**3`` 字节）。当写入下一行将超出此限制时，自动创建新文件。

数据结构（TypedDict）
~~~~~~~~~~~~~~~~~~~~~~~~

.. class:: MetaDict

   文档元信息，字段均为可选：

   - ``source`` (str)：原始文件路径
   - ``hash_name`` (str)：哈希后的文件名
   - ``dataset`` (str)：所属数据集名称
   - ``timestamp`` (str)：处理时间（ISO 8601 格式）
   - ``total_pages`` (int)：总页数
   - ``file_type`` (str)：文件类型（如 "pdf", "html"）
   - ``tag`` (list[str])：标签列表
   - ``description`` (str)：关于文档内容的描述

.. class:: UnifiedDoc

   统一文档结构：

   - ``meta``: :class:`MetaDict` 类型
   - ``json_content``: dict[str, list] —— 包含 ``texts`` 和 ``images`` 等字段的解析内容

.. class:: TextElement

   文本元素：

   - ``id`` (str)
   - ``type`` (str) —— 如 "paragraph", "heading"
   - ``text`` (str)
   - ``bbox`` (list[int]) —— [x0, y0, x1, y1] 坐标

.. class:: ImageElement

   非文本元素（图像、表格、行内公式等，字段可选）：

   - ``id`` (str)
   - ``type`` (str) —— 取值：``"image"``, ``"table"``, ``"interline_equation"``
   - ``url`` (str) —— 本地相对路径或 URL
   - ``bbox`` (list[int])
   - ``source`` (str) —— 来源文件
   - ``hash_name`` (str)
   - ``caption`` (str) —— 图注
   - ``desc`` (str) —— 由 VLM 生成的详细描述

函数说明
~~~~~~~~

.. autofunction:: src.file_parsing.unified_output.load_hash_mapping

   读取原始文件的hash映射

.. autofunction:: src.file_parsing.unified_output._now_iso

   输出标准化时间

.. autofunction:: src.file_parsing.unified_output.process_middle_files

   处理中间文件的核心函数

核心类：JsonlRoller
~~~~~~~~~~~~~~~~~~~

.. class:: src.file_parsing.unified_output.JsonlRoller

   该类负责 JSONL 文件的滚动写入。每个新文件以 ``{prefix}_{UTC时间戳}.jsonl`` 命名（如 ``doc_20251212103045_123456.jsonl``），确保全局唯一且可排序。

   滚动逻辑：在写入每一行前，检查当前文件大小 + 新行字节数是否超过 ``roll_size``。若超过，则关闭当前文件并创建新文件。

   使用示例：

   .. code-block:: python

      roller = JsonlRoller(out_dir=Path("output"), prefix="mydata")
      roller.write(my_unified_doc)
      roller.close()

依赖项
------

- ``loguru``：用于日志记录
- Python 标准库：``json``, ``pathlib``, ``shutil``, ``datetime``

注意事项
--------

- 中间文件目录（``temp_output``）在处理完成后会被 **自动删除**。