.. _doc_parser_module:

文档解析模块
===========================

本模块用于将多种格式的文档文件（如 PDF、Office 文档、图片等）转换为统一的结构化中间格式（``UnifiedDoc``），并最终合并输出为标准化的 JSONL 文件。该模块基于 MinerU 开源工具实现文档的智能解析和布局分析。

处理流程
--------

1. 将输入文档转换为 PDF 格式的字节流；
2. 使用 MinerU 的文档分析管道进行智能解析，提取文本、图像、表格和公式；
3. 将解析结果转换为中间 JSON 文件（存放于临时目录）；
4. 从哈希映射文件中恢复原始文件名，填充元数据（``meta``）；
5. 所有中间文件通过 ``process_middle_files`` 合并为滚动式 JSONL 输出；
6. 临时中间目录在合并完成后自动删除。

支持的文件格式
--------------

- **PDF**: 直接处理
- **Office 文档**: ``.doc``, ``.docx``, ``.ppt``, ``.pptx`` （需要安装 LibreOffice）
- **图片**: ``.jpg``, ``.jpeg``, ``.png`` （转换为单页 PDF）

使用示例
--------

.. code-block:: python

   from src.file_parsing.doc_parser import batch_parse
   from pathlib import Path

   file_list = [
       Path("data/raw/document1.pdf"),
       Path("data/raw/document2.docx"),
       Path("data/raw/image1.jpg")
   ]
   jsonls_dir = Path("data/my_dataset/parsed_file/doc_json_out")
   image_dir = Path("data/my_dataset/parsed_file/images")
   dataset = "my_dataset"
   mapping_json = Path("data/my_dataset/parsed_file/hash_dir/mapping.json")

   batch_parse(
       file_list,
       jsonls_dir,
       image_dir,
       dataset,
       mapping_json,
       lang="zh",  # 或 "en"
       parse_method="auto",
       formula_enable=True,
       table_enable=True
   )

函数参考
--------

.. autofunction:: src.file_parsing.doc_parser.batch_parse

参数说明
~~~~~~~~

``path_list`` : Sequence[str | Path]
   待解析的文档文件路径列表（支持 PDF、Office 文档、图片格式）。

``jsonls_dir`` : Path
   最终 JSONL 输出目录（通常是 ``doc_json_out/``）。
   临时中间文件将生成在该目录的同级目录 ``mineru_output/`` 中。

``image_dir`` : Path
   提取的图像、表格、公式等非文本元素的存储目录。

``dataset`` : str
   所属数据集名称，用于填充元数据字段 ``meta.dataset``。

``mapping_json`` : Path
   哈希映射文件路径（通常为 ``hash_dir/mapping.json``），用于将哈希文件名映射回原始文件名，以还原 ``meta.source`` 字段。

``lang`` : Literal["zh", "en"], optional
   文档主要语言，默认为 ``"en"``。影响 OCR 识别和文本处理。

``parse_method`` : str, optional
   MinerU 解析方法，默认为 ``"auto"``。

``formula_enable`` : bool, optional
   是否启用公式识别，默认为 ``True``。

``table_enable`` : bool, optional
   是否启用表格识别，默认为 ``True``。

``start_page_id`` : int, optional
   起始页码 ID，默认为 ``0``。

``end_page_id`` : int | None, optional
   结束页码 ID，默认为 ``None`` （处理到文档末尾）。

输出格式
--------

每个文档被转换为如下结构的 ``UnifiedDoc``：

.. code-block:: text

   {
     "meta": {
       "source": "original_document.pdf",
       "hash_name": "a1b2c3d4...pdf",
       "dataset": "my_dataset",
       "timestamp": "2025-12-12T10:30:45.123456",
       "total_pages": 10,
       "file_type": "pdf",
       "description": ""
     },
     "json_content": {
       "page_0": [
         {
           "id": "text_0_0",
           "type": "text",
           "text": "文档文本内容...",
           "bbox": [100, 200, 400, 250]
         },
         {
           "id": "image_0_0",
           "type": "image",
           "url": "image_001.png",
           "bbox": [150, 300, 350, 400],
           "caption": "图片标题"
         },
         {
           "id": "merge_text_0",
           "type": "merge_text",
           "text": "页面全部文本内容合并..."
         }
       ],
       "page_1": [...],
       ...
     }
   }

元素类型说明
~~~~~~~~~~~~

- **text**: 文本段落，包含边界框坐标
- **image**: 图片元素，包含图片 URL 和标题
- **table**: 表格元素（作为图片形式保存）
- **interline_equation**: 行间公式（作为图片形式保存）
- **merge_text**: 页面所有文本的合并版本

依赖项
------

- **MinerU**: 开源文档解析工具
- **LibreOffice**: Office 文档转 PDF（可选）
- **Pillow (PIL)**: 图片处理
- ``src.file_parsing.convert_to_unified`` 中的以下组件：
  - ``MetaDict``、``TextElement``、``ImageElement``、``UnifiedDoc``（类型定义）
  - ``_now_iso()``（生成 ISO 时间戳）
  - ``load_hash_mapping()``（加载哈希映射）
  - ``process_middle_files()``（合并中间文件）

注意事项
--------

- Office 文档转换需要系统安装 LibreOffice，并确保 ``libreoffice`` 命令可用；
- 图片文件会被转换为单页 PDF 进行处理；
- 表格和公式以图片形式保存，便于后续处理；
- 临时目录 ``mineru_output/`` 会在合并完成后**自动删除**，无需手动清理；
- 对于复杂布局的文档，建议启用 ``table_enable`` 和 ``formula_enable`` 以获得更好的解析效果。
