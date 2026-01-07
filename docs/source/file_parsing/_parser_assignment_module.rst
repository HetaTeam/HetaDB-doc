.. _parser_assignment_module:

文件解析分配模块
============================================

``ParserAssignment`` 类提供了一套端到端的原始文件处理流程，主要包括以下步骤：

1. **哈希重命名**：使用 SHA256 对原始文件进行重命名，确保文件名唯一且可复现。
2. **智能解析**：根据文件后缀自动选择合适的解析器进行批量解析。
3. **结构化输出**：将解析结果（文本/图像描述为 JSONL，表格为 CSV）组织到清晰的子目录中。

所有输出结果均保存在 ``data/<数据集名称>/parsed_file/`` 目录下。

使用示例
--------

.. code-block:: python

   from src.file_parsing.parser_assignment import ParserAssignment

   pa = ParserAssignment(
       data_dir="data",
       dataset_name="my_dataset",
       raw_file_dir="raw_file",
       pasrsed_dir="parsed_file",
       exclude_files=["temp_", "draft_"],
       config_supported_ext="default"
   )
   pa.cleanup()
   pa.step1_assignment()
   pa.step2_batch_parse(llm=my_llm_client, vlm=my_vlm_client)

类参考
------

.. autoclass:: src.file_parsing.parser_assignment.ParserAssignment
   :members:
   :undoc-members:
   :show-inheritance:

   .. automethod:: src.file_parsing.parser_assignment.ParserAssignment.__init__
   .. automethod:: src.file_parsing.parser_assignment.ParserAssignment.cleanup
   .. automethod:: src.file_parsing.parser_assignment.ParserAssignment.step1_assignment
   .. automethod:: src.file_parsing.parser_assignment.ParserAssignment.extract_archive
   .. automethod:: src.file_parsing.parser_assignment.ParserAssignment.step2_batch_parse

构造函数参数
~~~~~~~~~~~~

``data_dir`` : str  
   项目根目录路径（例如：``"data"``）。

``dataset_name`` : str  
   数据集子目录名称（例如：``"news_corpus"``）。

``raw_file_dir`` : str 或 list[pathlib.Path]  
   原始文件输入路径，支持两种形式：
     - 字符串：相对于 ``<data_dir>/<dataset_name>/`` 的目录路径；
     - ``Path`` 对象列表：直接指定待处理的文件列表。

``pasrsed_dir`` : str，可选（默认：``"parsed_file"``）  
   解析结果的输出目录名称（相对于数据集目录）。

``exclude_files`` : list[str]，可选  
   排除规则：若文件名包含列表中的任意子字符串，则跳过该文件（例如 ``["tmp", "backup"]``）。

``config_supported_ext`` : str 或 set，可选  
   - 若为 ``"default"``，则启用所有内置支持的文件扩展名；
   - 若为 ``set``，则仅处理指定的扩展名（是否带点均可）。  
     示例：``{".pdf", "txt", "jpg"}`` 会被标准化为 ``{".pdf", ".txt", ".jpg"}``。

生成的输出目录
~~~~~~~~~~~~~~

初始化后，以下子目录会自动创建于 ``parsed_dir`` 下：

- ``hash_dir/``：哈希重命名后的原始文件。
- ``image_dir/``：从 PDF 等文档中提取出的图像。
- ``text_json_out/``：包含文本内容和元数据的 JSONL 文件。
- ``csv_out/``：解析后的电子表格（CSV 格式）。
- ``image_desc_out/``：由视觉语言模型（VLM）生成的图像描述（JSONL）。
- ``table_desc_out/``：由大语言模型（LLM）生成的表格语义摘要（JSONL）。

方法说明
~~~~~~~~

``cleanup()``  
   删除已存在的 ``parsed_dir`` 目录，确保输出干净。

``step1_assignment()``  
   - 自动检测并解压归档文件（支持 ``.zip``, ``.7z``, ``.rar``, ``.tar*`` 等格式）；
   - 根据扩展名和支持规则过滤文件；
   - 使用 SHA256 对文件重命名，并生成 ``mapping.json`` 映射文件，记录原始文件名与哈希文件名的对应关系。

``extract_archive(filepath)``  
   原地解压单个归档文件。支持的格式包括：
   - ZIP（通过 ``zipfile``）
   - 7z（通过 ``py7zr``）
   - RAR（通过 ``rarfile``）
   - TAR（包括 ``.tar.gz``, ``.tar.xz``, ``.tar.bz2`` 等压缩变体）

``step2_batch_parse(llm, vlm)``  
   并行执行各类文件的批量解析：

   - **文本文件** （如 ``.txt``, ``.md``）：使用 ``text_parser.parse()`` → 输出 JSONL；
   - **文档文件** （如 ``.pdf``, ``.docx``, ``.pptx``）：使用 ``doc_parser.batch_parse()`` → 提取文本与内嵌图像；
   - **HTML 文件**：使用 ``html_parser.parse()`` → 输出 JSONL；
   - **图像文件** （原始图像 + 文档中提取的图像）：通过 ``image_parser.create_description()`` 调用 VLM 生成描述（异步）；
   - **电子表格** （如 ``.csv``, ``.xlsx``）：转换为 CSV，并调用 LLM 生成表格语义摘要（异步）。

   .. note::
      需传入有效的 LLM 和 VLM 客户端对象（例如来自 OpenAI、Anthropic 或本地推理服务）。

主函数
------

.. autofunction:: src.file_parsing.parser_assignment.main

.. note::
   ``main()`` 函数使用默认参数实例化 ``ParserAssignment``，适用于脚本或命令行调用。实际使用中，建议根据需求传入自定义路径和模型客户端。

支持的文件扩展名
----------------

默认支持以下扩展名：

- **文档类**：``.pdf``, ``.doc``, ``.docx``, ``.ppt``, ``.pptx``
- **网页类**：``.html``
- **图像类**：``.jpg``, ``.jpeg``, ``.png``, ``.gif``, ``.webp``, ``.tiff``, ``.bmp``, ``.ico``
- **纯文本类**：``.txt``, ``.text``, ``.md``, ``.markdown``
- **表格类**：``.csv``, ``.xls``, ``.xlsx``, ``.ods``

此外，归档文件（``.zip``, ``.7z``, ``.rar``, ``.tar*``）会在预处理阶段自动解压。

依赖项
------

- ``py7zr``
- ``rarfile``
- 自定义模块（位于 ``src.file_parsing`` 下）：
  - ``html_parser``
  - ``image_parser``
  - ``doc_parser``
  - ``sheet_parser``
  - ``text_parser``
  - ``hash_filename.rename_files_to_hash``
