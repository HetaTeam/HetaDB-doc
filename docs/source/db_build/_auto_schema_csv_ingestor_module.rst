.. _auto_schema_csv_ingestor_module:

自动建表 CSV 导入模块
====================

本模块实现 **零配置** 的 CSV 文件自动入库功能：  
自动推导表结构、创建 PostgreSQL 表、全量导入数据，并调用大语言模型（LLM）生成结构化表描述（含字段说明与示例查询）。  
同时支持将表格注册为知识图谱中的“数据报表”节点。

适用于快速构建可查询、可解释的结构化数据知识库。

核心能力
--------

- **自动建表**：根据 CSV 列名动态创建 PostgreSQL 表（所有字段为 ``TEXT`` 类型）；
- **安全标识符处理**：支持 Unicode 列名（如中文），通过双引号包裹确保 SQL 合法性；
- **自动描述生成**（可选）：调用 LLM 生成表用途、字段说明、示例查询；
- **断点续处理**：通过 ``record.txt`` 记录已处理文件，避免重复导入；
- **知识图谱集成**：将每张表注册为知识图谱节点（类型：`数据报表`）；
- **批量异步处理**：支持多 CSV 并行处理（通过 ``asyncio``）。

使用示例
--------

.. code-block:: python

   import asyncio
   from pathlib import Path
   from src.kg.ingest_csv import AutoSchemaCSVIngestor

   async def llm(prompt: str) -> str:
       # 实现你的 LLM 调用逻辑（如 OpenAI、Qwen 等）
       return '{"table_purpose": "...", "field_descriptions": {...}, "example_queries": [...]}'

   ingestor = AutoSchemaCSVIngestor(
       csv_dir="data/parsed_file/csv_out",
       table_desc_dir="data/parsed_file/table_desc_out",
       table_info_dir="data/kg/table_info",
       kg_node_dir="data/kg/nodes",
       postgres_config={
           "host": "localhost",
           "port": 5432,
           "database": "mydb",
           "user": "user",
           "password": "pass"
       },
       use_llm=llm
   )

   asyncio.run(ingestor.run())

类参考
------

.. autoclass:: src.kg.ingest_csv.AutoSchemaCSVIngestor
   :members:
   :undoc-members:

构造函数参数
~~~~~~~~~~~~

``csv_dir`` : str  
   CSV 文件所在目录（通常为 ``csv_out/``）。

``table_desc_dir`` : str  
   表格描述 JSON 文件目录（来自 ``sheet2csv`` 模块输出，如 ``table_desc_out/``）。

``table_info_dir`` : str  
   生成的结构化表描述与处理记录的存储目录。

``kg_node_dir`` : str  
   知识图谱节点输出目录（将生成 ``table_node.jsonl``）。

``postgres_config`` : dict  
   PostgreSQL 连接配置，包含 ``host``, ``port``, ``database``, ``user``, ``password`` 等。

``use_llm`` : Callable | None  
   异步 LLM 客户端函数。若为 ``None``，则跳过描述生成（当前代码强制要求，建议传入）。

核心方法
~~~~~~~~

.. automethod:: src.kg.ingest_csv.AutoSchemaCSVIngestor.run

.. automethod:: src.kg.ingest_csv.AutoSchemaCSVIngestor.process_csv

.. automethod:: src.kg.ingest_csv.AutoSchemaCSVIngestor.create_table_from_csv

.. automethod:: src.kg.ingest_csv.AutoSchemaCSVIngestor.load_csv_and_insert

.. automethod:: src.kg.ingest_csv.AutoSchemaCSVIngestor.generate_table_description

辅助函数
--------

.. autofunction:: src.kg.ingest_csv.normalize_identifier

**说明**：  
该函数将任意字符串（如 CSV 列名）转换为安全的 PostgreSQL 标识符：
- 保留 Unicode 字符（如中文）；
- 去除首尾空白；
- 空字符串转为 ``"_"``；
- **不自动加双引号**，调用方需用 ``f'"{col}"'`` 包裹以确保合法。

处理流程
--------

1. **扫描输入**：
   - 读取 ``csv_dir`` 下所有 ``.csv`` 文件；
   - 从 ``table_desc_dir`` 加载对应描述，提取 ``meta.description`` 作为表名来源；
2. **断点检查**：
   - 读取 ``table_info_dir/record.txt``，跳过已处理文件；
3. **建表**：
   - 读取 CSV 列名 → 标准化 → 创建 ``public.<表名>`` 表（全 TEXT）；
4. **数据导入**：
   - 全量读取 CSV → 空值转空串 → 批量插入（``execute_values``，1000 行/批）；
5. **生成描述**（调用 LLM）：
   - 提取前 3 行样本 + 列名；
   - 调用 LLM 生成 JSON 描述（含用途、字段说明、示例查询）；
   - 保存至 ``table_info_dir/<表名>.json``；
6. **注册图谱节点**：
   - 追加节点到 ``kg_node_dir/table_node.jsonl``，格式：
     .. code-block:: json

        {
          "NodeName": "销售汇总表",
          "Type": "文献实体",
          "SubType": "数据报表",
          "Description": "表用途|{字段说明字典}",
          "Id": "table_sales_page_0.csv"
        }
7. **记录完成**：将文件名写入 ``record.txt``。

LLM 提示词设计
---------------

系统使用以下提示调用 LLM：

.. code-block::

   你是一个数据库文档工程师。请根据以下 CSV 样本数据，为 PostgreSQL 表生成描述。
   表名: sales_summary
   前3行样本数据: [...]
   
   请输出一个 JSON 对象，包含以下字段：
   - "table_purpose": 表的总体用途（一句话）
   - "field_descriptions": 对象，key 为字段名（带双引号），value 为该字段的中文说明
   - "example_queries": 字符串数组，包含3个有代表性的 SELECT 查询示例（使用双引号，以分号结尾）
   只输出JSON格式字符串，不要用markdown等其他格式，不要任何其他内容，可以json.loads直接读取。

> **注意**：LLM 返回必须为纯 JSON 字符串，可被 ``json.loads()`` 直接解析。

输出产物
--------

- **PostgreSQL 表**：``public.<标准化表名>``
- **结构化描述**：``table_info_dir/<表名>.json``
- **处理记录**：``table_info_dir/record.txt``
- **图谱节点**：``kg_node_dir/table_node.jsonl``（追加模式）

依赖项
------

- **Python 包**：
  - ``pandas``（CSV 读取）
  - ``psycopg2``（PostgreSQL 连接）
  - ``asyncio`` / ``logging``
- **外部服务**：
  - PostgreSQL 数据库
  - LLM 服务（如 OpenAI、Qwen、本地模型 API）

注意事项
--------

- **字段类型**：当前所有列强制为 ``TEXT``，适用于探索性分析；生产环境可扩展类型推断；
- **表名冲突**：若多个 CSV 的 ``description`` 相同，将写入同一张表（需确保结构一致）；
- **LLM 必须实现**：当前代码假设 ``self.llm`` 非空，若无需描述生成，需修改逻辑；
- **空列处理**：空列名统一转为 ``"_"``，若多列同名需手动处理（当前未去重）；
- **性能**：大数据量 CSV 建议分批或使用 ``COPY``，当前使用 ``execute_values`` 适用于中小规模。

> **提示**：可通过扩展 ``normalize_identifier`` 或引入列名去重逻辑提升鲁棒性。