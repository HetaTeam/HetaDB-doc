.. _sql_db_module:

SQL数据库集成
============

本模块提供PostgreSQL数据库的集成功能，支持将文本块(chunk)和知识图谱(graph)数据高效插入到PostgreSQL数据库中。模块采用批量插入和并发处理策略，优化了大数据量的导入性能。

功能特性
--------

* **高效批量插入**：使用PostgreSQL的execute_values进行批量数据插入
* **并发处理**：支持多线程并发执行，提高处理速度
* **自动表管理**：自动创建和管理数据库表结构
* **数据清理**：支持数据的清理和重置操作
* **连接池管理**：高效的数据库连接管理和复用
* **错误处理**：完善的异常处理和日志记录

核心类
------

PostgreSQLConnector
~~~~~~~~~~~~~~~~~~~

PostgreSQL数据库连接器，负责数据库连接管理和基本操作。

**主要方法：**

.. automethod:: src.db_build.sql_db.sql_db.PostgreSQLConnector.__init__

.. automethod:: src.db_build.sql_db.sql_db.PostgreSQLConnector.connect

.. automethod:: src.db_build.sql_db.sql_db.PostgreSQLConnector.disconnect

.. automethod:: src.db_build.sql_db.sql_db.PostgreSQLConnector.execute_query

.. automethod:: src.db_build.sql_db.sql_db.PostgreSQLConnector.batch_insert

ChunkInserter
~~~~~~~~~~~~~

文本块数据插入器，专门处理chunk数据的批量插入。

**主要方法：**

.. automethod:: src.db_build.sql_db.sql_db.ChunkInserter.insert_chunks

.. automethod:: src.db_build.sql_db.sql_db.ChunkInserter.create_chunk_table

.. automethod:: src.db_build.sql_db.sql_db.ChunkInserter.clear_table

GraphInserter
~~~~~~~~~~~~~

知识图谱数据插入器，处理节点和关系数据的批量插入。

**主要方法：**

.. automethod:: src.db_build.sql_db.sql_db.GraphInserter.insert_nodes

.. automethod:: src.db_build.sql_db.sql_db.GraphInserter.insert_relations

.. automethod:: src.db_build.sql_db.sql_db.GraphInserter.create_graph_tables

.. automethod:: src.db_build.sql_db.sql_db.GraphInserter.clear_tables

数据表结构
----------

Chunk表结构
~~~~~~~~~~~

.. code-block:: sql

   CREATE TABLE {table_name} (
       id SERIAL PRIMARY KEY,
       chunk_id VARCHAR(255) UNIQUE NOT NULL,
       content_text TEXT NOT NULL,
       source_file VARCHAR(500),
       source_url TEXT,
       metadata JSONB,
       embedding VECTOR(1536),  -- 如果使用pgvector
       created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
   );

节点表结构
~~~~~~~~~~~

.. code-block:: sql

   CREATE TABLE {dataset}_entities (
       id SERIAL PRIMARY KEY,
       entity_id VARCHAR(255) UNIQUE NOT NULL,
       entity_name VARCHAR(500) NOT NULL,
       entity_type VARCHAR(100),
       description TEXT,
       properties JSONB,
       embedding VECTOR(1536),  -- 如果使用pgvector
       created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
   );

关系表结构
~~~~~~~~~~~

.. code-block:: sql

   CREATE TABLE {dataset}_relations (
       id SERIAL PRIMARY KEY,
       relation_id VARCHAR(255) UNIQUE NOT NULL,
       head_entity VARCHAR(255) NOT NULL,
       relation_type VARCHAR(100) NOT NULL,
       tail_entity VARCHAR(255) NOT NULL,
       description TEXT,
       properties JSONB,
       embedding VECTOR(1536),  -- 如果使用pgvector
       created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
       FOREIGN KEY (head_entity) REFERENCES {dataset}_entities(entity_id),
       FOREIGN KEY (tail_entity) REFERENCES {dataset}_entities(entity_id)
   );

使用示例
--------

基本使用
~~~~~~~~

.. code-block:: python

   from src.db_build.sql_db.sql_db import PostgreSQLConnector, ChunkInserter

   # 创建连接器
   connector = PostgreSQLConnector()

   # 创建chunk插入器
   chunk_inserter = ChunkInserter(connector)

   # 批量插入chunks
   chunks_data = [
       {
           'chunk_id': 'chunk_001',
           'content_text': '这是文本内容...',
           'source_file': 'document.pdf',
           'metadata': {'page': 1}
       }
   ]

   chunk_inserter.insert_chunks(chunks_data, table_name='test_chunks')

图谱数据插入
~~~~~~~~~~~~

.. code-block:: python

   from src.db_build.sql_db.sql_db import GraphInserter

   # 创建图谱插入器
   graph_inserter = GraphInserter(connector)

   # 插入节点
   nodes_data = [
       {
           'entity_id': 'entity_001',
           'entity_name': '实体名称',
           'entity_type': 'Person',
           'description': '实体描述',
           'properties': {'age': 30}
       }
   ]

   graph_inserter.insert_nodes(nodes_data, dataset='test_dataset')

   # 插入关系
   relations_data = [
       {
           'relation_id': 'rel_001',
           'head_entity': 'entity_001',
           'relation_type': 'works_for',
           'tail_entity': 'entity_002',
           'description': '工作关系',
           'properties': {'since': '2020'}
       }
   ]

   graph_inserter.insert_relations(relations_data, dataset='test_dataset')

批量处理配置
------------

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 默认值
     - 说明
   * - ``batch_size``
     - 1000
     - 单次批量插入的数据量
   * - ``max_workers``
     - 4
     - 并发处理的最大线程数
   * - ``retry_attempts``
     - 3
     - 失败重试次数
   * - ``retry_delay``
     - 1.0
     - 重试间隔时间(秒)

性能优化
--------

* **批量插入**：使用execute_values进行批量插入，减少数据库往返次数
* **并发处理**：多线程并发执行，提高处理速度
* **索引优化**：自动创建必要的索引以提升查询性能
* **内存管理**：分批处理大数据，避免内存溢出
* **连接复用**：连接池管理，提高连接利用率

错误处理
--------

模块实现了完善的错误处理机制：

* **连接错误**：自动重试连接，记录详细错误信息
* **插入失败**：分批重试失败的数据，提供详细的错误报告
* **数据验证**：插入前验证数据格式和完整性
* **日志记录**：详细的操作日志和错误追踪

配置依赖
--------

模块依赖以下配置文件：

* **PostgreSQL配置**：数据库连接信息
* **批量处理配置**：批量大小、并发数等参数
* **表命名规则**：数据集和表的命名规范

依赖项
------

* **psycopg2**：PostgreSQL数据库驱动
* **psycopg2.extras**：execute_values等高级功能
* **concurrent.futures**：线程池支持
* **pathlib**：路径处理
* **json**：JSON数据处理

注意事项
--------

* **数据一致性**：插入操作会验证外键约束
* **性能监控**：建议监控数据库性能和磁盘空间
* **备份策略**：重要数据建议定期备份
* **字符编码**：确保数据使用UTF-8编码
* **表命名**：避免使用SQL关键字作为表名
