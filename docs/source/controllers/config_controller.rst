.. _config_controller_module:

配置控制器
==========

配置控制器提供了系统配置的动态管理和查询API接口，支持运行时配置的查看、更新和重置操作。

功能特性
--------

* **配置查询**：获取当前系统配置信息
* **配置更新**：动态修改配置参数
* **配置重置**：恢复默认配置设置
* **配置验证**：确保配置参数的合法性

API接口
--------

GET /api/v1/config
~~~~~~~~~~~~~~~~~~

--todo 获取当前系统配置。

响应示例：

.. code-block:: json

   {
     "success": true,
     "message": "配置获取成功",
     "config": {
       "env": "dev",
       "host": "localhost",
       "port": 8000,
       "database": "hetadb",
       "llm_model": "gpt-4",
       "embedding_dim": 1536
     }
   }

PUT /api/v1/config
~~~~~~~~~~~~~~~~~~

--todo 更新配置参数。

请求参数：

.. list-table::
   :header-rows: 1
   :widths: 20 20 60

   * - 参数
     - 类型
     - 说明
   * - ``key``
     - str
     - 配置项键名（必需）
   * - ``value``
     - Any
     - 配置项值（必需）

请求示例：

.. code-block:: json

   {
     "key": "llm_model",
     "value": "gpt-3.5-turbo"
   }

响应示例：

.. code-block:: json

   {
     "success": true,
     "message": "配置项 llm_model 更新成功",
     "config": {
       "llm_model": "gpt-3.5-turbo"
     }
   }

POST /api/v1/config/reset
~~~~~~~~~~~~~~~~~~~~~~~~

--todo 重置所有配置为默认值。

响应示例：

.. code-block:: json

   {
     "success": true,
     "message": "配置重置成功"
   }

数据模型
--------

ConfigUpdate
~~~~~~~~~~~~

配置更新模型：

.. code-block:: python

   class ConfigUpdate(BaseModel):
       key: str
       value: Any

ConfigResponse
~~~~~~~~~~~~~~

配置响应模型：

.. code-block:: python

   class ConfigResponse(BaseModel):
       success: bool
       message: str
       config: Dict[str, Any]

配置分类
--------

系统配置主要分为以下几类：

运行环境配置
~~~~~~~~~~~~

* **env**：运行环境（dev/prod）
* **host**：服务主机地址
* **port**：服务端口
* **reload**：自动重载（开发模式）

数据库配置
~~~~~~~~~~

* **database**：数据库名称
* **db_host**：数据库主机
* **db_port**：数据库端口
* **db_user**：数据库用户名
* **db_password**：数据库密码

AI服务配置
~~~~~~~~~~

* **llm_model**：大语言模型名称
* **llm_api_key**：LLM API密钥
* **embedding_model**：嵌入模型名称
* **embedding_dim**：向量维度
* **vlm_model**：视觉语言模型名称

查询配置
~~~~~~~~

* **top_k**：默认检索数量
* **threshold**：相似度阈值
* **similarity_weight**：相似度权重
* **occur_weight**：出现次数权重

配置验证
--------

控制器会对配置更新进行以下验证：

* **类型检查**：确保参数类型正确
* **值范围检查**：确保数值参数在合理范围内
* **格式验证**：确保字符串参数格式正确
* **依赖检查**：确保相关配置参数一致

配置持久化
----------

配置更新支持两种持久化方式：

* **内存持久化**：仅在当前运行时生效，重启后失效
* **文件持久化**：写入配置文件，重启后仍然生效

配置更新流程
------------

1. **参数验证**：验证请求参数的格式和类型
2. **权限检查**：检查用户是否有配置修改权限
3. **配置验证**：验证新配置值的合法性
4. **配置更新**：更新内存中的配置
5. **持久化存储**：可选的文件持久化
6. **热更新**：通知相关服务进行热更新

安全考虑
--------

* **敏感信息保护**：API密钥等敏感信息在响应中脱敏
* **权限控制**：仅授权用户可以修改配置
* **审计日志**：记录所有配置变更操作
* **回滚机制**：支持配置变更的回滚操作

错误处理
--------

* **400 Bad Request**：参数格式错误或值非法
* **403 Forbidden**：权限不足
* **500 Internal Server Error**：配置更新失败

配置依赖
--------

配置控制器依赖以下组件：

* **配置加载器**：从文件加载配置
* **配置验证器**：验证配置参数
* **配置存储器**：持久化配置变更
* **服务发现**：通知其他服务配置变更
