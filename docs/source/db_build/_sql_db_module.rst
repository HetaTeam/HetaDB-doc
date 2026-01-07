.. _sql_db_module:

PostgreSQL数据库操作模块
==========================

本模块提供完整的PostgreSQL数据库操作功能，专门用于HetaDB知识库系统的结构化数据存储和管理。支持高效的批量数据插入、索引管理和查询优化，特别针对文本块、知识图谱实体和关系数据进行了优化。

功能特性
--------

* **批量数据操作**：使用psycopg2的execute_values进行高效批量插入
* **自动表管理**：自动创建和管理数据库表结构及索引
* **数据清理功能**：支持按条件删除实体、关系和文本块数据
* **查询功能**：支持数据映射查询和关系查询
* **并发安全**：多线程环境下安全的数据操作
* **错误处理**：完善的异常处理和事务管理

核心函数
--------

连接管理
~~~~~~~~

**get_postgres_connection()**

获取PostgreSQL数据库连接对象。

.. code-block:: python

   def get_postgres_connection() -> psycopg2.extensions.connection:
       """获取PostgreSQL连接"""

表创建和管理
~~~~~~~~~~~~

**create_chunk_table(chunk_table, postgres_config)**

创建用于存储文本分块的PostgreSQL表结构。

**create_graph_tables(dataset)**

创建知识图谱相关的PostgreSQL表结构，包括实体表、关系表和关联表。

**create_cluster_chunk_relation_table(dataset)**

创建实体/关系与文本分块关联关系的表结构。

数据插入
~~~~~~~~

**insert_entities_to_pg(records, dataset)**

批量插入实体数据到PostgreSQL实体表。

**insert_relations_to_pg(records, dataset)**

批量插入关系数据到PostgreSQL关系表。

**batch_insert_chunks_pg(chunks_data, postgres_config, chunk_table, postgres_batch_size)**

批量插入文本分块数据到PostgreSQL分块表。

**insert_cluster_chunk_relations(relations, dataset)**

插入实体/关系与文本分块的关联关系数据。

数据删除
~~~~~~~~

**delete_entities_from_pg(ids, dataset)**

按实体ID删除PostgreSQL实体表中的数据。

**delete_relations_from_pg(ids, dataset)**

按关系ID删除PostgreSQL关系表中的数据。

**delete_chunks_by_source_ids(source_ids, dataset)**

按来源ID删除分块表中的数据。

**delete_cluster_chunk_relations_by_cluster_ids(cluster_ids, dataset)**

按集群ID删除集群-分块关系表中的数据。

**delete_cluster_chunk_relations_by_urls(urls, dataset)**

按URL删除集群-分块关系表中的数据。

数据查询
~~~~~~~~

**get_chunk_source_mapping(chunk_ids, chunk_table)**

从分块表中查询分块ID对应的来源信息映射。

**query_cluster_chunk_relations_by_urls(urls, dataset)**

按URL查询集群-分块关系表中的关联数据。

**get_cluster_chunk_mapping(cluster_ids)**

从集群-分块关系表中查询集群ID对应的分块信息映射。

数据读取
~~~~~~~~

**read_local_chunks(data_dir)**

从本地目录读取所有文本分块的JSONL文件数据。

**read_local_nodes(data_dir)**

从本地目录读取所有实体节点的JSONL文件数据。

**read_local_relations(data_dir)**

从本地目录读取所有实体关系的JSONL文件数据。

数据表结构
----------

Chunk表结构
~~~~~~~~~~~

存储文本分块数据的表结构：

.. code-block:: sql

   CREATE TABLE {table_name} (
       id SERIAL PRIMARY KEY,
       chunk_id VARCHAR(128),
       content_text TEXT,
       source_id TEXT,
       source_chunk TEXT,
       created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
   );

   CREATE INDEX idx_{table_name}_chunk_id ON {table_name}(chunk_id);

实体表结构
~~~~~~~~~~~

存储知识图谱实体数据的表结构：

.. code-block:: sql

   CREATE TABLE {dataset}_entities (
       id SERIAL PRIMARY KEY,
       node_name VARCHAR(500),
       type VARCHAR(100),
       sub_type VARCHAR(100),
       description TEXT,
       node_id VARCHAR(128),
       created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
   );

   CREATE INDEX idx_{dataset}_entities_node_id ON {dataset}_entities(node_id);

关系表结构
~~~~~~~~~~~

存储知识图谱关系数据的表结构：

.. code-block:: sql

   CREATE TABLE {dataset}_relations (
       id SERIAL PRIMARY KEY,
       node1 VARCHAR(500),
       node2 VARCHAR(500),
       type VARCHAR(100),
       semantics VARCHAR(100),
       description TEXT,
       node_id VARCHAR(128),
       created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
   );

   CREATE INDEX idx_{dataset}_relations_node_id ON {dataset}_relations(node_id);

集群-分块关系表结构
~~~~~~~~~~~~~~~~~~~~

存储实体/关系与文本分块关联关系的表结构：

.. code-block:: sql

   CREATE TABLE {dataset}_cluster_chunk_relation (
       id BIGSERIAL PRIMARY KEY,
       created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
       updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
       meta JSONB,
       cluster_id VARCHAR(100) NOT NULL,
       chunk_id VARCHAR(125) NOT NULL,
       url TEXT NOT NULL,
       type VARCHAR(32)
   );

   CREATE INDEX idx_{dataset}_cluster_chunk_relation_cluster_id
   ON {dataset}_cluster_chunk_relation(cluster_id);

   CREATE INDEX idx_{dataset}_cluster_chunk_relation_chunk_id
   ON {dataset}_cluster_chunk_relation(chunk_id);

使用示例
--------

创建表结构
~~~~~~~~~~

.. code-block:: python

   from src.db_build.sql_db.sql_db import create_chunk_table, create_graph_tables

   # 创建chunk表
   postgres_config = {
       "host": "localhost",
       "port": 5432,
       "database": "heta_db",
       "user": "heta_user",
       "password": "password"
   }
   create_chunk_table("my_chunks", postgres_config)

   # 创建图谱表（实体表、关系表、关联表）
   create_graph_tables("my_dataset")

插入数据
~~~~~~~~

.. code-block:: python

   from src.db_build.sql_db.sql_db import (
       insert_entities_to_pg,
       insert_relations_to_pg,
       batch_insert_chunks_pg
   )

   # 插入实体数据
   entities_data = [
       {
           "Id": "entity_001",
           "NodeName": "人工智能",
           "Type": "attr",
           "SubType": "technology",
           "Description": "人工智能技术领域"
       }
   ]
   insert_entities_to_pg(entities_data, "my_dataset")

   # 插入关系数据
   relations_data = [
       {
           "Id": "rel_001",
           "Node1": "人工智能",
           "Node2": "机器学习",
           "Relation": "包含",
           "Type": "triple",
           "Description": "人工智能包含机器学习"
       }
   ]
   insert_relations_to_pg(relations_data, "my_dataset")

   # 插入文本块数据
   chunks_data = [
       {
           "chunk_id": "chunk_001",
           "text": "人工智能是计算机科学的一个分支...",
           "source": "ai_intro.pdf",
           "source_chunk": '["chunk_001"]'
       }
   ]
   batch_insert_chunks_pg(chunks_data, postgres_config, "my_chunks", 1000)

数据查询
~~~~~~~~

.. code-block:: python

   from src.db_build.sql_db.sql_db import get_chunk_source_mapping, query_cluster_chunk_relations_by_urls

   # 查询chunk的来源映射
   chunk_ids = ["chunk_001", "chunk_002"]
   source_mapping = get_chunk_source_mapping(chunk_ids, "my_chunks")
   # 返回: {"chunk_001": "ai_intro.pdf", "chunk_002": "ml_guide.pdf"}

   # 查询集群-分块关系
   urls = ["ai_intro.pdf", "ml_guide.pdf"]
   relations = query_cluster_chunk_relations_by_urls(urls, "my_dataset")

配置参数
--------

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 默认值
     - 说明
   * - ``BATCH_SIZE``
     - 1000
     - 单次批量插入的数据量
   * - ``NUM_THREADS``
     - 4
     - 并发处理的最大线程数
   * - PostgreSQL配置
     - 通过 ``get_postgres_conn_config()`` 获取
     - 数据库连接配置（host, port, database, user, password）

性能优化
--------

* **批量插入优化**：使用psycopg2的execute_values进行高效批量插入
* **索引自动创建**：为关键字段自动创建索引以提升查询性能
* **并发安全**：多线程环境下安全的数据操作和事务管理
* **内存管理**：分批处理大数据，避免内存溢出
* **字符串清理**：自动清理和截断字符串字段以确保数据完整性

错误处理
--------

模块实现了完善的错误处理和事务管理：

* **连接异常**：PostgreSQL连接失败时抛出异常并记录详细错误信息
* **事务管理**：使用commit/rollback确保数据一致性
* **数据验证**：插入前进行基本的数据验证和字符串清理
* **日志记录**：详细的操作日志，包括成功插入数量和错误详情
* **异常传播**：非致命异常记录日志，致命异常抛出给调用方

配置依赖
--------

模块依赖以下配置：

* **PostgreSQL连接配置**：通过 ``get_postgres_conn_config()`` 获取数据库连接信息
* **批量处理参数**：``BATCH_SIZE`` 和 ``NUM_THREADS`` 全局常量
* **字符串清理**：``clean_str`` 工具函数用于数据预处理

依赖项
------

* **psycopg2**：PostgreSQL数据库连接和操作
* **psycopg2.extras**：``execute_values`` 批量插入功能
* **concurrent.futures**：``ThreadPoolExecutor`` 并发处理
* **pathlib**：路径和文件系统操作
* **json**：JSON数据解析和序列化
* **src.utils.load_config**：配置加载模块
* **src.utils.utils**：工具函数模块

注意事项
--------

* **字符串长度限制**：各字段有最大长度限制，超长内容会被截断
* **主键唯一性**：``node_id`` 和 ``chunk_id`` 字段在各自表中必须唯一
* **数据集命名**：表名通过 ``{dataset}_{table_type}`` 格式生成
* **并发安全**：多线程环境下确保数据操作的线程安全性
* **事务完整性**：插入操作使用事务，确保数据一致性
* **字段兼容性**：支持字段名的多种别名（如 ``Id``/``id``、``Text``/``text``）
