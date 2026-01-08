.. _doc_parser_module:

复合文档批量解析模块
==============

本模块用于将一批异构文档（包括 PDF、Office 文档、图像等）批量转换为结构化的统一输出格式（``UnifiedDoc``），  
通过调用 `MinerU` 的核心分析流水线，提取文本、公式、表格、图像等元素，并最终输出符合规范的 JSONL 文件。

模块支持从本地文件系统读取多种格式输入，将其统一转换为 PDF 内存流后送入解析器，适用于构建符合文档知识库。

对外接口简洁，仅需调用 ``batch_parse()`` 函数即可完成端到端处理。

功能亮点
--------

- **多格式输入支持**：PDF、Office（.doc/.docx/.ppt/.pptx）均可解析；
- **结构化元素提取**：支持正文文本、内联/行间公式、表格、图像及其图注；
- **图像本地化存储**：所有解析出的图像保存为本地文件，URL 字段记录文件名；
- **语言与解析策略可配**：支持中英文语言选择及自动/指定解析策略；
- **自动合并输出**：将MinerU原始结构化结果转换为统一的JSONL格式；

使用示例
--------

.. code-block:: python

   from src.file_parsing.mineru_batch_parser import batch_parse
   from pathlib import Path

   path_list = [
       "data/raw/doc1.docx",
       "data/raw/slide1.pptx",
       "data/raw/image1.png",
       "data/raw/report.pdf",
   ]
   jsonls_dir = Path("data/my_dataset/parsed_file/text_json_out")
   image_dir = Path("data/my_dataset/parsed_file/images")
   dataset = "enterprise_docs"
   mapping_json = Path("data/my_dataset/parsed_file/hash_dir/mapping.json")

   batch_parse(
       path_list=path_list,
       jsonls_dir=jsonls_dir,
       image_dir=image_dir,
       dataset=dataset,
       mapping_json=mapping_json,
       lang="zh",
       parse_method="auto",
       formula_enable=True,
       table_enable=True,
   )

函数参考
--------

.. autofunction:: src.file_parsing.mineru_batch_parser.batch_parse

参数说明
~~~~~~~~

``path_list`` : Sequence[str | Path]  
   待解析的本地文件路径列表，支持 PDF、Office 文档、图像等。

``jsonls_dir`` : Path  
   最终 JSONL 输出目录（通常为 ``text_json_out/``）。

``image_dir`` : Path  
   解析出的所有图像将拷贝至此目录，供后续引用。

``dataset`` : str  
   数据集名称，用于填充元数据字段。

``mapping_json`` : Path  
   哈希映射文件路径（如 ``hash_dir/mapping.json``），用于将文件哈希名还原为原始文件名，填入 ``meta.source``。

``lang`` : Literal["zh", "en"], optional, default="en"  
   文档语言，影响 OCR 与公式识别策略。

``parse_method`` : str, optional, default="auto"  
   解析策略，可为 "ocr"、"native" 或 "auto"，由 MinerU 解析器使用。

``formula_enable`` : bool, optional, default=True  
   是否启用行间公式识别。

``table_enable`` : bool, optional, default=True  
   是否启用表格识别。

``start_page_id``, ``end_page_id`` : int, optional  
   当前保留接口，暂未在批量解析中启用（MinerU 单文档全页解析）。

输出格式
--------

每个输入文件生成一个 ``UnifiedDoc``，结构如下：

.. code-block:: json

   {
     "meta": {
       "source": "original_file.docx",
       "hash_name": "a1b2c3d4...pdf",
       "dataset": "enterprise_docs",
       "timestamp": "2026-01-07T14:30:00.123456",
       "total_pages": 5,
       "file_type": "pdf",
       "description": ""
     },
     "json_content": {
       "page_0": [
         { "id": "text_0_0", "type": "text", "text": "正文内容...", "bbox": [x1,y1,x2,y2] },
         { "id": "image_0_0", "type": "image", "url": "img_001.jpg", "caption": "示意图", "bbox": [...] },
         { "id": "image_0_1", "type": "table", "url": "tbl_001.jpg", "caption": "数据表", "bbox": [...] },
         { "id": "image_0_2", "type": "interline_equation", "url": "eq_001.jpg", "caption": "E=mc²", "bbox": [...] },
         { "id": "merge_text_0", "type": "merge_text", "text": "全文拼接结果..." }
       ],
       "page_1": [ ... ]
     }
   }

**注意**：  

- ``url`` 字段为图像文件名（非 URL），图像实际存储于 ``image_dir``；  
- ``bbox`` 为 [x1, y1, x2, y2] 格式的整数坐标，表示元素在页面中的位置；  
- ``merge_text_X`` 为该页所有文本内容（不含公式/表格图像）的拼接结果，用于全文检索。

核心处理流程
------------

1. **格式统一化**：将所有输入文件（Office、图像等）通过工具链转换为 PDF 字节流；
2. **MinerU 分析**：调用 ``doc_analyze`` 对 PDF 进行结构化识别（文本、公式、表格、图像）；
3. **中间 JSON 生成**：将 MinerU 的模型输出转为中间 JSON 格式（含图像路径）；
4. **结构转换**：使用 ``MinuerUNoEmbedConverter`` 将中间格式转为 ``UnifiedDoc`` 元素；
5. **图像拷贝**：将每份文档的图像从临时目录拷贝至统一 ``image_dir``；
6. **合并输出**：调用 ``process_middle_files`` 将所有中间 JSON 合并为 JSONL 并清理临时目录。

支持的输入格式
--------------

- **PDF**：直接读取；
- **Office**：通过系统 `libreoffice` 转换为 PDF（需预装）；
- **图像** （暂未使用到）：单张 JPG/PNG 封装为单页 PDF（使用 Pillow）。

依赖项
------

- **系统依赖**：

  - ``libreoffice`` （用于 Office 文档转换）
  
- **Python 包**：

  - ``Pillow`` （图像处理）
  - ``loguru`` （日志）
  - ``beautifulsoup4`` （间接依赖，用于统一输出）
  
- **内部模块**：

  - ``src.file_parsing.convert_to_unified``
  - ``mineru.backend.pipeline.*`` （MinerU 核心分析模块）

注意事项
--------

- 所有输入文件必须存在于本地磁盘；
- ``mapping_json`` 必须包含输入文件（或其 PDF 形式）到哈希名的映射；
- 图像、表格、公式均以图片形式输出，不提供原始 LaTeX 或 HTML 表格；
- 临时目录 ``mineru_output/`` 在合并完成后自动删除；
- 若某文件转换或解析失败，仅记录日志并跳过，不中断整体流程；
- ``libreoffice`` 转换需确保系统环境可用，且对大文件可能较慢。