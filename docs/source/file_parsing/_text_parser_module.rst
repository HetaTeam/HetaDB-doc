.. _text_parser_module:

纯文本解析模块
===========================

本模块用于将一批纯文本文件（如 ``.txt``、``.md`` 等）转换为统一的结构化中间格式（``UnifiedDoc``），并最终合并输出为标准化的 JSONL 文件。

处理流程
--------

1. 为每个输入文本文件生成一个中间 JSON 文件（存放于临时目录）；
2. 每个文件的内容被封装为单页文档（``page_0``）中的一个 ``merge_text`` 元素；
3. 从哈希映射文件中恢复原始文件名，填充元数据（``meta``）；
4. 所有中间文件通过 ``process_middle_files`` 合并为滚动式 JSONL 输出；
5. 临时中间目录在合并完成后自动删除。

使用示例
--------

.. code-block:: python

   from src.file_parsing.text_parser import parse
   from pathlib import Path

   file_list = [Path("data/raw/file1.txt"), Path("data/raw/file2.md")]
   jsonls_dir = Path("data/my_dataset/parsed_file/text_json_out")
   dataset = "my_dataset"
   mapping_json = Path("data/my_dataset/parsed_file/hash_dir/mapping.json")

   parse(file_list, jsonls_dir, dataset, mapping_json)

函数参考
--------

.. autofunction:: src.file_parsing.text_parser.parse

参数说明
~~~~~~~~

``file_list`` : list[Path]  
   待解析的文本文件路径列表（支持任意纯文本格式）。

``jsonls_dir`` : Path  
   最终 JSONL 输出目录（通常是 ``text_json_out/``）。  
   临时中间文件将生成在该目录的同级目录 ``text_parser_temp_output/`` 中。

``dataset`` : str  
   所属数据集名称，用于填充元数据字段 ``meta.dataset``。

``mapping_json`` : Path  
   哈希映射文件路径（通常为 ``hash_dir/mapping.json``），用于将哈希文件名映射回原始文件名，以还原 ``meta.source`` 字段。

编码处理策略
------------

- 优先尝试以 **UTF-8** 编码读取文件；
- 若失败（如遇到 ``UnicodeDecodeError``），则回退到系统默认编码；
- 若仍失败，则打印错误并跳过该文件（**不会中断整个流程**）。

输出格式
--------

每个文本文件被转换为如下结构的 ``UnifiedDoc``：

.. code-block:: json

   {
     "meta": {
       "source": "original_name.txt",
       "hash_name": "a1b2c3d4...txt",
       "dataset": "my_dataset",
       "timestamp": "2025-12-12T10:30:45.123456",
       "total_pages": 1,
       "file_type": "text",
       "description": ""
     },
     "json_content": {
       "page_0": [
         {
           "id": "merge_text_0",
           "type": "merge_text",
           "text": "文件的全部内容..."
         }
       ]
     }
   }

该中间 JSON 文件随后被 ``process_middle_files`` 合并到滚动式 JSONL 输出中。

依赖项
------

- ``src.file_parsing.convert_to_unified`` 中的以下组件：
  - ``MetaDict``、``TextElement``、``UnifiedDoc``（类型定义）
  - ``_now_iso()``（生成 ISO 时间戳）
  - ``load_hash_mapping()``（加载哈希映射）
  - ``process_middle_files()``（合并中间文件）

注意事项
--------

- 本模块假设输入文件均为**单页纯文本**，因此 ``total_pages`` 固定为 1；
- 不提取图像、表格或结构化布局信息；
- 临时目录 ``text_parser_temp_output/`` 会在合并完成后**自动删除**，无需手动清理。