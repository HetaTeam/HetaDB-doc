.. _csv_ingestor_module:

CSV数据导入器
=============

本模块提供CSV文件的自动化导入和处理功能，支持将结构化数据转换为数据库记录，并生成相应的表结构和元数据描述。

功能特性
--------

* **自动表结构生成**：根据CSV文件自动创建数据库表
* **数据类型推断**：智能识别和转换数据类型
* **批量数据导入**：高效的批量数据插入操作
* **元数据生成**：自动生成表和字段的描述信息
* **数据验证**：导入前的数据完整性和格式验证
* **错误处理**：完善的异常处理和错误恢复机制

核心类
------

CSVIngestor
~~~~~~~~~~~

CSV数据导入器的主要类。

.. autoclass:: src.db_build.sql_db.csv_ingestor.CSVIngestor
   :members:
   :undoc-members:

主要方法
~~~~~~~~

__init__
^^^^^^^^

初始化CSV导入器。

.. automethod:: src.db_build.sql_db.csv_ingestor.CSVIngestor.__init__

analyze_csv
^^^^^^^^^^^

分析CSV文件的结构和数据类型。

.. automethod:: src.db_build.sql_db.csv_ingestor.CSVIngestor.analyze_csv

create_table
^^^^^^^^^^^^

创建数据库表结构。

.. automethod:: src.db_build.sql_db.csv_ingestor.CSVIngestor.create_table

import_data
^^^^^^^^^^^

导入CSV数据到数据库。

.. automethod:: src.db_build.sql_db.csv_ingestor.CSVIngestor.import_data

generate_metadata
^^^^^^^^^^^^^^^^^

生成表的元数据描述。

.. automethod:: src.db_build.sql_db.csv_ingestor.CSVIngestor.generate_metadata

使用示例
--------

完整导入流程
~~~~~~~~~~~~

.. code-block:: python

   from src.db_build.sql_db.csv_ingestor import CSVIngestor

   # 创建导入器
   ingestor = CSVIngestor(
       csv_path="data/sales_data.csv",
       table_name="sales_records",
       db_config={
           "host": "localhost",
           "port": 5432,
           "database": "business_db",
           "user": "postgres",
           "password": "password"
       }
   )

   # 执行完整导入
   success = ingestor.full_import()
   if success:
       print("数据导入成功")
   else:
       print("导入失败")

分步导入
~~~~~~~~

.. code-block:: python

   # 1. 分析CSV结构
   schema = ingestor.analyze_csv()
   print(f"检测到 {len(schema['columns'])} 列")

   # 2. 创建表结构
   ingestor.create_table(schema)

   # 3. 导入数据
   ingestor.import_data()

   # 4. 生成元数据
   metadata = ingestor.generate_metadata()
   print(f"表描述: {metadata['description']}")

自定义配置
~~~~~~~~~~

.. code-block:: python

   # 高级配置
   ingestor = CSVIngestor(
       csv_path="data/complex_data.csv",
       table_name="complex_table",
       db_config=db_config,
       encoding="utf-8",
       delimiter=",",
       quote_char='"',
       batch_size=1000,
       skip_rows=1,  # 跳过标题行
       type_mapping={
           "price": "DECIMAL(10,2)",
           "date": "DATE"
       }
   )

数据类型映射
------------

自动类型推断
~~~~~~~~~~~~

模块支持自动识别以下数据类型：

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - CSV类型
     - SQL类型
     - 识别规则
   * - 整数
     - INTEGER/BIGINT
     - 纯数字，无小数点
   * - 浮点数
     - DECIMAL/NUMERIC
     - 包含小数点的数字
   * - 日期
     - DATE/DATETIME
     - 标准日期格式
   * - 布尔值
     - BOOLEAN
     - true/false, yes/no, 1/0
   * - 文本
     - VARCHAR/TEXT
     - 其他所有类型

自定义类型映射
~~~~~~~~~~~~~~

支持手动指定字段类型：

.. code-block:: python

   type_mapping = {
       "customer_id": "VARCHAR(50)",
       "order_date": "TIMESTAMP",
       "total_amount": "DECIMAL(12,2)",
       "is_active": "BOOLEAN"
   }

配置参数
--------

基本参数
~~~~~~~~

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 默认值
     - 说明
   * - ``csv_path``
     - 必需
     - CSV文件路径
   * - ``table_name``
     - 必需
     - 目标表名
   * - ``db_config``
     - 必需
     - 数据库连接配置
   * - ``encoding``
     - ``utf-8``
     - 文件编码格式

解析参数
~~~~~~~~

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 默认值
     - 说明
   * - ``delimiter``
     - ``,``
     - 字段分隔符
   * - ``quote_char``
     - ``"``
     - 引号字符
   * - ``escape_char``
     - ``\``
     - 转义字符
   * - ``skip_rows``
     - ``0``
     - 跳过前N行

性能参数
~~~~~~~~

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 默认值
     - 说明
   * - ``batch_size``
     - ``1000``
     - 批量插入大小
   * - ``max_workers``
     - ``4``
     - 并发处理线程数
   * - ``chunk_size``
     - ``8192``
     - 文件读取块大小

质量保证
--------

数据验证
~~~~~~~~

* **格式验证**：检查CSV文件的格式正确性
* **数据完整性**：验证必填字段和数据约束
* **类型一致性**：确保数据类型与表结构匹配
* **引用完整性**：检查外键约束（如果适用）

错误处理
~~~~~~~~

* **文件不存在**：清晰的错误提示和建议
* **编码错误**：自动尝试多种编码格式
* **数据错误**：详细的错误位置和原因报告
* **数据库错误**：事务回滚和详细错误信息

性能优化
--------

* **流式处理**：对大文件进行流式读取
* **批量插入**：减少数据库往返次数
* **并发处理**：多线程并行处理
* **内存管理**：控制内存使用避免溢出

元数据生成
----------

自动生成的元数据包括：

* **表描述**：基于表名和内容生成的自然语言描述
* **字段描述**：每个字段的含义和数据类型说明
* **示例查询**：常用的查询示例
* **数据统计**：基本的数据分布统计

依赖项
------

* **pandas**：数据处理和类型推断
* **sqlalchemy**：数据库操作和类型映射
* **psycopg2**：PostgreSQL数据库驱动
* **chardet**：自动编码检测

应用场景
--------

1. **业务数据导入**：将业务系统数据导入数据库
2. **历史数据迁移**：迁移 legacy 系统的数据
3. **批量数据处理**：处理大量结构化数据的导入
4. **数据仓库建设**：构建企业数据仓库的基础数据
