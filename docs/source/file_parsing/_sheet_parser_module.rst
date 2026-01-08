.. _sheet_parser_module:

表格解析与描述生成模块
=================================

本模块用于批量解析电子表格文件（CSV、Excel、ODS），将其内容转换为结构化 CSV 文件，并调用大语言模型（LLM）为每个工作表（Sheet）生成语义描述（标题 + 内容摘要）。  
输出包括：

- 每个工作表对应的 CSV 文件（存于 ``csv_out/``）；
- 每个工作表对应的结构化描述 JSON 文件（存于 ``jsonls_dir/``），符合 ``UnifiedDoc`` 规范。

该模块支持多 Sheet 文件（如 .xlsx），并为每个 Sheet 独立处理。

功能特点
--------

- **多格式支持**：CSV、XLS、XLSX、ODS；
- **自动 Schema 提取**：获取列名及数据类型，构造 LLM 提示；
- **上下文感知描述**：利用列名、前三行数据、原始文件名生成高质量表格摘要；
- **异步并发处理**：基于 ``asyncio`` 和 ``tqdm_asyncio``，高效批量调用 LLM；
- **结构化输出**：CSV + JSON 双输出，便于后续检索或入库；
- **哈希映射还原**：通过 ``mapping.json`` 恢复原始文件名，确保溯源性。

使用示例
--------

.. code-block:: python

   from src.file_parsing.sheet_parser import parse
   from pathlib import Path
   import asyncio

   # 假设已初始化 LLM 客户端（需实现 callable 接口）
   async def llm_client(prompt: str) -> str:
       # 实际调用 OpenAI/Gemini/Qwen 等
       return '{"caption": "销售数据表", "desc": "2024年Q1各区域销售额..."}'

   file_list = [Path("data/raw/sales.xlsx"), Path("data/raw/users.csv")]
   csv_out = Path("data/my_dataset/parsed_file/csv_out")
   jsonls_dir = Path("data/my_dataset/parsed_file/table_desc_out")
   dataset = "business_data"
   mapping_json = Path("data/my_dataset/parsed_file/hash_dir/mapping.json")

   asyncio.run(
       parse(
           file_list=file_list,
           csv_out=csv_out,
           jsonls_dir=jsonls_dir,
           dataset=dataset,
           mapping_json=mapping_json,
           llm=llm_client,
       )
   )

函数参考
--------

.. autofunction:: src.file_parsing.sheet_parser.parse

参数说明
~~~~~~~~

``file_list`` : list[Path]  
   待解析的表格文件路径列表（支持 ``.csv``, ``.xls``, ``.xlsx``, ``.ods``）。

``csv_out`` : Path  
   CSV 输出目录，每个工作表将生成独立的 CSV 文件（命名格式：``table_<文件名>_page_<序号>.csv``）。

``jsonls_dir`` : Path  
   表格描述 JSON 文件输出目录（每个工作表一个 JSON 文件，命名格式类似 CSV）。

``dataset`` : str  
   数据集名称，用于填充元数据字段 ``meta.dataset``。

``mapping_json`` : Path  
   哈希映射文件路径（如 ``hash_dir/mapping.json``），用于将哈希文件名还原为原始文件名（填入 ``meta.source``）。

``llm`` : Callable  
   异步 LLM 客户端函数，需满足签名：
   .. code-block:: python

      async def llm(prompt: str) -> str:

   返回值应为包含 ``{"caption": "...", "desc": "..."}`` 的 JSON 字符串（或可解析子串）。

LLM Prompt 设计
---------------

系统使用以下提示词调用模型：

.. code-block::

   请给这个表格提供说明，根据表格列名和前三行内容推测图片标题，并给出表格内容的描述；
   表格列名为：col1 (int64), col2 (object), ...
   前几行内容为：...（pandas DataFrame 头三行）
   表格的文件名是 original_name.xlsx，需要甄别是否与表格内容有关
   要求语言简洁凝练
   严格按照下列JSON格式输出，不要其他任何解释：{"caption": "表格标题", "desc": "表格内容描述"}

**说明**：  

- 列名附带数据类型（如 ``sales (int64)``），帮助模型理解结构；  
- 前三行数据提供内容示例；  
- 原始文件名作为辅助上下文（可能包含业务线索）。

输出格式
--------

1. **CSV 文件**  
   路径：``csv_out/table_<stem>_page_<idx>.csv``  
   内容：原始表格数据（不含索引）。

2. **描述 JSON 文件** （符合 ``UnifiedDoc``）  
   路径：``jsonls_dir/table_<stem>_page_<idx>.json``  
   内容示例：

   .. code-block:: json

      {
        "meta": {
          "source": "sales_report.xlsx",
          "hash_name": "a1b2c3d4...xlsx",
          "dataset": "business_data",
          "timestamp": "2025-12-12T10:30:45.123456",
          "total_pages": 1,
          "file_type": "text",
          "description": "business_data_2024年Q1销售汇总"
        },
        "json_content": {
          "page_0": [
            {
              "id": "text_0_0",
              "type": "text",
              "text": "2024年Q1销售汇总: 包含华东、华南等区域的月度销售额与同比增长率..."
            }
          ]
        }
      }

**注意**：  

- 每个工作表被视为独立“文档”；  
- ``meta.description`` 包含数据集名和表格标题；  
- 文本内容为 ``caption: desc`` 的拼接，便于全文检索。

核心工具函数
------------

.. autofunction:: src.file_parsing.sheet_parser.parse_table_structure

.. autofunction:: src.file_parsing.sheet_parser.get_sheet_desc_async

依赖项
------

- **Python 包**：

  - ``pandas`` （表格读取与写入）
  - ``openai`` （示例 LLM 客户端，实际可替换）
  - ``tqdm`` （异步进度条）
  - ``pyyaml`` （配置加载，间接依赖）
  
- **内部模块**：

  - ``src.file_parsing.convert_to_unified``（统一数据结构：``MetaDict``, ``TextElement``, ``UnifiedDoc`` 等）

注意事项
--------

- **LLM 客户端需自行实现**：模块不绑定具体模型，只需提供兼容的异步函数；
- **多 Sheet 处理**：单个 Excel 文件中的每个工作表将生成独立的 CSV 和 JSON；
- **输出目录自动创建**：若 ``csv_out`` 或 ``jsonls_dir`` 不存在，会自动创建；
- **无中间临时目录**：与其它解析模块不同，本模块**不使用临时中间目录**，直接输出到目标位置（原代码中相关逻辑已注释）；
- **重试机制**：LLM 调用失败时最多重试 3 次；
- **命名规范**：输出文件名包含 ``table_`` 前缀和 ``page_<idx>`` 后缀，确保唯一性。
