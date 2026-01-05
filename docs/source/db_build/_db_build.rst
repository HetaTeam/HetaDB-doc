.. _db_build:

数据库安装
============

本指南将帮助您快速安装和配置HetaDB所需的核心数据库服务。

Docker 环境中使用的各种组件的默认版本，通过 ``docker/docker-compose.yml`` 安装。

默认版本
^^^^^^^^^
* PostgreSQL: 18.0
* Milvus: v2.5.4
* etcd: v3.5.5
* MinIO: RELEASE.2024-09-13T20-26-02Z
* Attu: v2.4.0

安装步骤
^^^^^^^^^

1. **准备环境变量**

   设置数据卷目录环境变量（可选）：

   .. code-block:: bash

      # 设置Docker数据卷目录，默认为当前目录
      export DOCKER_VOLUME_DIRECTORY=/path/to/your/data

   如果不设置，将使用docker-compose.yml文件所在目录。

2. **创建必要的目录**

   .. code-block:: bash

      cd docker

      # 为PostgreSQL创建数据目录
      mkdir -p ${DOCKER_VOLUME_DIRECTORY:-.}/pg

      # 为Milvus相关服务创建目录
      mkdir -p ${DOCKER_VOLUME_DIRECTORY:-.}/milvus/volumes/etcd
      mkdir -p ${DOCKER_VOLUME_DIRECTORY:-.}/milvus/volumes/minio
      mkdir -p ${DOCKER_VOLUME_DIRECTORY:-.}/milvus/volumes/milvus

3. **启动数据库服务**

   .. code-block:: bash

      # 启动所有数据库服务
      docker-compose up -d

   此命令将下载并启动以下容器：

   * **PostgreSQL**：关系型数据库，用于存储结构化数据和元数据
   * **Milvus**：向量数据库，用于向量相似度搜索和检索
   * **etcd**：分布式键值存储，作为Milvus的元数据存储
   * **MinIO**：对象存储，用于存储Milvus的向量数据
   * **Attu**：Milvus的可视化管理界面

4. **验证服务状态**

   检查服务是否正常启动：

   .. code-block:: bash

      # 查看所有服务状态
      docker-compose ps

      # 查看服务日志
      docker-compose logs -f [service_name]

数据库端口配置
^^^^^^^^^^^^^^^^

.. list-table:: 服务端口映射
   :header-rows: 1
   :widths: 20 20 30 30

   * - 服务
     - 容器端口
     - 主机端口
     - 说明
   * - PostgreSQL
     - 5432
     - 5432
     - 数据库读写端口
   * - Milvus
     - 19530
     - 19530
     - Milvus服务端口
   * - Milvus
     - 9091
     - 9091
     - Milvus健康检查端口
   * - MinIO API
     - 9000
     - 9000
     - 对象存储API端口
   * - MinIO Console
     - 9001
     - 9001
     - MinIO管理界面
   * - Attu
     - 3000
     - 8000
     - Milvus可视化管理界面

服务访问地址
^^^^^^^^^^^^^

启动完成后，可以通过以下地址访问各服务：

* **PostgreSQL**: ``postgresql://postgres:postgres@localhost:5432/postgres``
* **Milvus**: ``localhost:19530``
* **MinIO Console**: http://localhost:9001 (用户名/密码: minioadmin/minioadmin)
* **Attu**: http://localhost:8000

服务依赖关系
^^^^^^^^^^^^^

各服务的启动依赖关系如下：

* **etcd** 和 **MinIO** 可以并行启动
* **Milvus** 依赖 **etcd** 和 **MinIO** 的健康状态
* **Attu** 依赖 **Milvus** 服务

健康检查
^^^^^^^^

所有服务都配置了健康检查：

* **etcd**: 通过 ``etcdctl endpoint health`` 检查
* **MinIO**: 通过HTTP健康检查端点检查
* **Milvus**: 通过 ``/healthz`` 端点检查

.. note::
   所有服务都运行在 ``milvus-network`` 网络中，确保服务间的网络互通。

数据持久化
^^^^^^^^^^

所有服务的数据都通过Docker卷进行持久化：

* **PostgreSQL**: ``${DOCKER_VOLUME_DIRECTORY}/pg``
* **etcd**: ``${DOCKER_VOLUME_DIRECTORY}/milvus/volumes/etcd``
* **MinIO**: ``${DOCKER_VOLUME_DIRECTORY}/milvus/volumes/minio``
* **Milvus**: ``${DOCKER_VOLUME_DIRECTORY}/milvus/volumes/milvus``

停止服务
^^^^^^^^

.. code-block:: bash

   # 停止所有服务
   docker-compose down

   # 停止服务并删除数据卷（慎用）
   docker-compose down -v

数据处理流水线
--------------

.. toctree::
   :maxdepth: 2
   :caption: 数据处理模块

   _data_processing_pipeline.rst