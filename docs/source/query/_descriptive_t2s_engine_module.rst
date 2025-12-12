.. _descriptive_t2s_engine_module:

描述性文本到 SQL 引擎
====================

本模块实现 **自然语言 → 结构化 SQL → 自然语言答案** 的端到端问答流程，专为已导入 PostgreSQL 的结构化表格数据设计。  
它利用 LLM 生成安全 SQL，并基于执行结果生成简洁、可读的中文回答。

适用于面向业务人员的自然语言数据查询（NLQ）场景。

核心能力
--------

- **语义理解**：结合表用途、字段说明、示例查询，精准理解用户意图；
- **安全 SQL 生成**：强制使用双引号包裹字段名，禁止写操作；
- **安全执行**：仅允许 ``SELECT``，设置 30 秒超时，防止注入与慢查询；
- **结果摘要**：将数据库返回结果（最多前 5 行）转化为自然语言；
- **零格式输出**：最终答案为纯文本，无 Markdown 或技术细节。

使用示例
--------

.. code-block:: python

   import asyncio
   from src.qa.descriptive_t2s_engine import DescriptiveText2SQLEngine

   async def llm(prompt: str) -> str:
       # 实现你的 LLM 调用（如 OpenAI、Qwen 等）
       return "SELECT \"sales\" FROM \"sales_table\" WHERE \"region\" LIKE '%华东%';"

   engine = DescriptiveText2S Engine(
       table_info_dir="data/kg/table_info",
       postgres_config={
           "host": "localhost",
           "port": 5432,
           "database": "mydb",
           "user": "user",
           "password": "pass"
       },
       use_llm=llm
   )

   answer = await engine.query(
       question="华东地区的销售额是多少？",
       table_name="sales_summary"
   )
   print(answer)  # 输出：华东地区的销售额为 1,250,000 元。

类参考
------

.. autoclass:: src.qa.descriptive_t2s_engine.DescriptiveText2SQLEngine
   :members:
   :undoc-members:

构造函数参数
~~~~~~~~~~~~

``table_info_dir`` : str  
   表结构描述目录（由 ``AutoSchemaCSVIngestor`` 生成），包含 ``<表名>.json`` 文件。

``postgres_config`` : dict  
   PostgreSQL 连接配置，需包含：``host``, ``port``, ``database``, ``user``, ``password``。

``use_llm`` : Callable  
   异步 LLM 客户端函数，需满足：
   .. code-block:: python
      async def llm(prompt: str) -> str:

主方法
~~~~~~

.. automethod:: src.qa.descriptive_t2s_engine.DescriptiveText2SQLEngine.query

**输入**：
- ``question``：用户的自然语言问题；
- ``table_name``：要查询的表名（必须已存在描述文件）。

**输出**：自然语言答案（字符串），若失败则返回 ``"None"``。

辅助方法
~~~~~~~~

.. automethod:: src.qa.descriptive_t2s_engine.DescriptiveText2SQLEngine.load_table_info

.. automethod:: src.qa.descriptive_t2s_engine.DescriptiveText2SQLEngine.sanitize_and_execute_sql

表描述文件格式
--------------

每张表对应一个 JSON 文件（``<表名>.json``），由 LLM 生成，包含以下字段：

.. code-block:: json

   {
     "table_purpose": "销售汇总表，记录各区域季度销售额",
     "field_descriptions": {
       "\"region\"": "销售区域，如'华东'、'华南'",
       "\"quarter\"": "财季，格式为'Q1'-'Q4'",
       "\"sales\"": "销售额（单位：元）"
     },
     "example_queries": [
       "SELECT \"region\", SUM(\"sales\") FROM \"sales_table\" GROUP BY \"region\";",
       "SELECT * FROM \"sales_table\" WHERE \"quarter\" = 'Q1';"
     ]
   }

> **注意**：字段名在 ``field_descriptions`` 的 key 中**已包含双引号**，以确保 LLM 生成合规 SQL。

LLM 提示词设计
---------------

模块使用两阶段提示：

1. **SQL 生成提示**（``query_info_prompt``）  
   - 提供：问题、表名、用途、字段列表、字段说明、示例 SQL  
   - 强制规则：
     - 字段名必须用双引号；
     - 仅允许 ``SELECT``；
     - 数值字段显式转为 ``::NUMERIC``；
     - 模糊匹配优先使用 ``LIKE``；
     - 输出仅为 SQL 语句，以分号结尾。

2. **答案生成提示**（``answer_info_prompt``）  
   - 提供：原始问题、生成的 SQL、执行结果（最多前 5 行）  
   - 强制规则：
     - 输出纯中文段落；
     - 简洁直给，不列数据；
     - 信息不足时返回 ``"None"``；
     - 不使用任何格式。

安全机制
--------

- **SQL 白名单**：仅允许以 ``SELECT`` 开头的语句；
- **关键词黑名单**：拒绝包含 ``INSERT/UPDATE/DELETE/DROP/CREATE/ALTER`` 的 SQL；
- **执行隔离**：使用 SQLAlchemy 引擎，连接参数设置 ``statement_timeout=30s``；
- **字段名规范化**：依赖 LLM 输出带双引号的字段，避免 SQL 注入。

依赖项
------

- **Python 包**：
  - ``pandas``（SQL 执行与结果处理）
  - ``sqlalchemy``（数据库连接）
  - ``asyncio`` / ``logging``
- **外部服务**：
  - PostgreSQL
  - LLM 服务（如 OpenAI、Qwen、本地 API）

注意事项
--------

- **LLM 必须可靠**：若 LLM 未遵守输出规则（如返回解释文本），SQL 解析将失败；
- **表描述质量决定效果**：字段说明越清晰，SQL 生成越准确；
- **结果截断**：仅使用前 5 行结果生成答案，避免长文本影响 LLM 输出；
- **错误处理**：任何 SQL 执行异常均返回 ``"None"``，适合前端直接展示；
- **性能**：每次查询需两次 LLM 调用 + 一次数据库查询，建议缓存高频问题。

日志
----

- 模块使用名为 ``sql_ans`` 的 logger；
- 仅记录 SQL 执行异常，不记录用户问题或结果（保护隐私）。
