.. _database_installation:

数据库安装
============

本指南将帮助您快速安装项目所需数据库。

Docker 环境中使用的各种组件的默认版本，通过docker-compose.yml安装。

默认版本
^^^^^^^^^
* Docker: 4.35.0
* Milvus: 2.4.0
* MySQL: 5.7

安装步骤
^^^^^^^^^

1. **在docker目录下创建目录：**

   .. code-block:: bash

      cd docker
      mkdir -p milvus
      mkdir -p mysql/data


2. **一键安装数据库**

   .. code-block:: bash

      # 安装并启动数据库服务
      docker-compose up -d

此命令将下载并启动以下容器：

* milvus：用于向量相似度搜索


3. **数据库端口**

.. list-table:: 服务端口映射
   :header-rows: 1
   :widths: 25 25 25

   * - Service
     - Front-end Port
     - Read/Write Port
   * - Milvus
     - 8000
     - 19530
   * - MySQL
     - 8080
     - 3306


.. note::
   Front-end Port用于浏览器查看数据服务的可视化界面，Read/Write Port用于数据库的读写操作。