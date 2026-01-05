.. _csv2db_module:

CSV到数据库转换器
=================

本模块提供CSV文件到数据库的高级转换功能，支持多表关联、数据转换、完整性检查等高级特性，是CSV数据导入的增强版本。

功能特性
--------

* **多表支持**：支持将单个CSV文件转换为多个相关表
* **数据转换**：内置丰富的数据转换和清洗功能
* **关联关系**：自动识别和创建表间关联关系
* **数据质量**：全面的数据质量检查和修复
* **增量更新**：支持数据的增量更新而非全量替换
* **事务安全**：完整的事务管理和回滚机制

核心类
------

CSVToDBConverter
~~~~~~~~~~~~~~~~

CSV到数据库的高级转换器。

.. autoclass:: src.db_build.sql_db.csv2db.CSVToDBConverter
   :members:
   :undoc-members:

主要方法
~~~~~~~~

__init__
^^^^^^^^

初始化转换器。

.. automethod:: src.db_build.sql_db.csv2db.CSVToDBConverter.__init__

load_and_validate
^^^^^^^^^^^^^^^^^

加载并验证CSV数据。

.. automethod:: src.db_build.sql_db.csv2db.CSVToDBConverter.load_and_validate

normalize_schema
^^^^^^^^^^^^^^^^

规范化数据模式。

.. automethod:: src.db_build.sql_db.csv2db.CSVToDBConverter.normalize_schema

create_tables
^^^^^^^^^^^^^

创建数据库表结构。

.. automethod:: src.db_build.sql_db.csv2db.CSVToDBConverter.create_tables

transform_data
^^^^^^^^^^^^^^

转换和清洗数据。

.. automethod:: src.db_build.sql_db.csv2db.CSVToDBConverter.transform_data

import_data
^^^^^^^^^^^

执行数据导入。

.. automethod:: src.db_build.sql_db.csv2db.CSVToDBConverter.import_data

高级功能
--------

多表拆分
~~~~~~~~

将宽表拆分为多个规范化表：

.. code-block:: python

   from src.db_build.sql_db.csv2db import CSVToDBConverter

   converter = CSVToDBConverter(
       csv_path="data/customer_orders.csv",
       db_config=db_config,
       multi_table=True
   )

   # 自动拆分为客户表、订单表、产品表等
   converter.convert()

数据转换
~~~~~~~~

内置的数据转换规则：

.. code-block:: python

   # 配置转换规则
   converter = CSVToDBConverter(
       csv_path="data/raw_data.csv",
       transformations={
           "date_column": "str_to_date",
           "price_column": "clean_currency",
           "status_column": {"mapping": {"active": 1, "inactive": 0}}
       }
   )

关联关系处理
~~~~~~~~~~~~

自动识别和创建外键关系：

.. code-block:: python

   # 自动处理关联关系
   converter = CSVToDBConverter(
       csv_path="data/relational_data.csv",
       auto_relations=True,
       foreign_keys={
           "order_table.customer_id": "customer_table.id",
           "order_item.order_id": "order_table.id"
       }
   )

使用示例
--------

完整转换流程
~~~~~~~~~~~~

.. code-block:: python

   from src.db_build.sql_db.csv2db import CSVToDBConverter

   # 创建转换器
   converter = CSVToDBConverter(
       csv_path="data/complex_data.csv",
       db_config={
           "host": "localhost",
           "port": 5432,
           "database": "analytics_db",
           "user": "postgres",
           "password": "secure_pass"
       },
       table_prefix="analytics_",
       create_indexes=True,
       validate_data=True
   )

   # 执行完整转换
   result = converter.full_convert()

   print(f"创建了 {result['tables_created']} 个表")
   print(f"导入了 {result['rows_imported']} 行数据")

自定义转换
~~~~~~~~~~

.. code-block:: python

   # 自定义数据转换函数
   def custom_transform(value):
       # 自定义转换逻辑
       return value.upper() if isinstance(value, str) else value

   converter = CSVToDBConverter(
       csv_path="data/custom_data.csv",
       custom_transforms={
           "name_column": custom_transform,
           "email_column": "validate_email"
       }
   )

增量更新
~~~~~~~~

.. code-block:: python

   # 增量更新模式
   converter = CSVToDBConverter(
       csv_path="data/updates.csv",
       incremental=True,
       update_key="id",  # 基于ID进行更新
       conflict_resolution="update"  # 冲突时更新
   )

   converter.update_existing_data()

数据质量保证
------------

验证规则
~~~~~~~~

内置的验证规则包括：

* **数据类型验证**：确保数据类型正确
* **范围验证**：检查数值范围合理性
* **格式验证**：验证字符串格式（如邮箱、电话）
* **唯一性验证**：检查唯一约束
* **引用完整性**：验证外键关系

.. code-block:: python

   # 配置验证规则
   converter = CSVToDBConverter(
       validation_rules={
           "email": "email_format",
           "age": {"range": [0, 150]},
           "phone": "phone_format",
           "customer_id": "unique"
       }
   )

数据修复
~~~~~~~~

自动的数据修复功能：

.. code-block:: python

   # 启用自动修复
   converter = CSVToDBConverter(
       auto_repair=True,
       repair_rules={
           "trim_whitespace": True,
           "fix_dates": True,
           "standardize_formats": True
       }
   )

性能优化
--------

并行处理
~~~~~~~~

.. code-block:: python

   converter = CSVToDBConverter(
       parallel_processing=True,
       max_workers=8,
       chunk_size=5000
   )

内存优化
~~~~~~~~

.. code-block:: python

   converter = CSVToDBConverter(
       memory_efficient=True,
       streaming=True,
       batch_size=1000
   )

配置参数
--------

核心参数
~~~~~~~~

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 默认值
     - 说明
   * - ``multi_table``
     - ``false``
     - 是否启用多表拆分
   * - ``auto_relations``
     - ``true``
     - 自动识别关联关系
   * - ``incremental``
     - ``false``
     - 增量更新模式
   * - ``validate_data``
     - ``true``
     - 启用数据验证

转换参数
~~~~~~~~~

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 默认值
     - 说明
   * - ``table_prefix``
     - ``""``
     - 表名前缀
   * - ``create_indexes``
     - ``true``
     - 自动创建索引
   * - ``foreign_keys``
     - ``{}``
     - 自定义外键关系

质量参数
~~~~~~~~

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 默认值
     - 说明
   * - ``auto_repair``
     - ``false``
     - 自动修复数据
   * - ``strict_mode``
     - ``false``
     - 严格模式（遇到错误即停止）
   * - ``error_threshold``
     - ``0.05``
     - 错误容忍率（5%）

错误处理
--------

异常类型
~~~~~~~~

* **FileNotFoundError**：CSV文件不存在
* **ValidationError**：数据验证失败
* **DatabaseError**：数据库操作失败
* **TransformationError**：数据转换失败

错误恢复
~~~~~~~~

.. code-block:: python

   try:
       result = converter.full_convert()
   except ValidationError as e:
       print(f"验证错误: {e}")
       # 处理验证错误
   except DatabaseError as e:
       print(f"数据库错误: {e}")
       # 回滚事务
   except Exception as e:
       print(f"未知错误: {e}")
       # 清理临时资源

日志记录
~~~~~~~~

详细的操作日志：

* **INFO**：正常操作记录
* **WARNING**：警告信息（如数据问题）
* **ERROR**：错误信息和堆栈跟踪
* **DEBUG**：详细的调试信息

依赖项
------

* **pandas**：数据处理和分析
* **sqlalchemy**：ORM和数据库抽象
* **psycopg2**：PostgreSQL驱动
* **chardet**：编码检测
* **validators**：数据验证库

应用场景
--------

1. **企业数据迁移**：将遗留系统数据迁移到新数据库
2. **数据仓库建设**：构建企业级数据仓库
3. **业务智能**：为BI系统准备规范化数据
4. **数据质量改进**：清理和规范化业务数据
