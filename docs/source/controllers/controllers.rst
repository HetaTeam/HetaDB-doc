.. _controllers_module:

API控制器
=========

HetaDB的API控制器模块提供了完整的RESTful API接口，支持知识库查询、文件管理和系统配置管理。所有控制器都基于FastAPI框架开发，提供类型安全的数据验证和自动API文档生成。

控制器架构
----------

控制器模块采用功能划分的设计模式：

* **chat_controller**：知识库聊天和查询接口
* **files_controller**：文件上传和管理接口
* **config_controller**：系统配置管理接口

每个控制器都包含：

* **路由定义**：基于FastAPI的路由装饰器
* **数据模型**：基于Pydantic的请求/响应模型
* **业务逻辑**：核心功能实现
* **错误处理**：统一的异常处理机制

核心特性
--------

* **类型安全**：使用Pydantic进行数据验证
* **异步支持**：原生支持异步请求处理
* **自动文档**：基于OpenAPI规范的API文档
* **统一响应**：标准化的响应格式
* **错误处理**：详细的错误信息和状态码
* **日志记录**：完整的请求追踪和错误日志

基础接口
--------

所有控制器都遵循统一的接口设计模式：

请求格式
~~~~~~~~

.. code-block:: json

   {
     "参数名": "参数值",
     "参数名2": "参数值2"
   }

成功响应格式
~~~~~~~~~~~~

.. code-block:: json

   {
     "success": true,
     "message": "操作成功",
     "data": {...},
     "total_count": 0,
     "request_id": "uuid",
     "code": 200,
     "query_info": {...},
     "response": "..."
   }

错误响应格式
~~~~~~~~~~~~

.. code-block:: json

   {
     "success": false,
     "message": "错误描述",
     "data": [],
     "total_count": 0,
     "request_id": "uuid",
     "code": 400,
     "query_info": {},
     "response": null
   }

状态码说明
~~~~~~~~~~

* **200**：请求成功
* **400**：请求参数错误
* **500**：服务器内部错误

公共参数
--------

所有控制器接口都支持以下公共参数：

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 参数
     - 类型
     - 说明
   * - ``request_id``
     - str
     - 请求唯一标识符（可选）
   * - ``user_id``
     - str
     - 用户标识符（部分接口必需）
   * - ``timestamp``
     - datetime
     - 请求时间戳（自动生成）

配置要求
--------

控制器模块依赖以下配置：

* **数据库连接**：PostgreSQL和Milvus配置
* **API密钥**：LLM/VLM/Embedding服务密钥
* **系统配置**：FastAPI服务配置
* **日志配置**：日志输出和轮转配置

使用前请确保所有配置文件正确设置。
