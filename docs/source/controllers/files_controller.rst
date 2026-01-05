.. _files_controller_module:

文件控制器
==========

文件控制器提供了文件上传、管理和知识库组织的核心API接口，支持文件的生命周期管理和知识库的层次化组织结构。

功能特性
--------

* **文件上传**：支持多种格式文件的上传
* **文件管理**：文件的查看、详情获取和删除
* **知识库管理**：知识库的创建、查看和管理
* **数据集管理**：数据集的组织和文件关联
* **层次化结构**：支持知识库-数据集-文件的多层组织

文件管理
--------

GET /api/v1/files
~~~~~~~~~~~~~~~~~

--todo 获取文件列表。

响应示例：

.. code-block:: json

   [
     {
       "id": "file_001",
       "name": "document.pdf",
       "size": 1024000,
       "upload_time": "2024-01-01T12:00:00Z",
       "type": "pdf"
     }
   ]

POST /api/v1/files/upload
~~~~~~~~~~~~~~~~~~~~~~~~

--todo 上传文件。

请求参数：

* **file**：上传的文件（multipart/form-data）

响应示例：

.. code-block:: json

   {
     "success": true,
     "message": "文件上传成功",
     "file_id": "file_001"
   }

GET /api/v1/files/{file_id}
~~~~~~~~~~~~~~~~~~~~~~~~~~~

--todo 获取文件详情。

路径参数：

* **file_id**：文件ID

响应示例：

.. code-block:: json

   {
     "success": true,
     "message": "文件详情获取成功",
     "file_id": "file_001",
     "name": "document.pdf",
     "size": 1024000,
     "upload_time": "2024-01-01T12:00:00Z",
     "type": "pdf",
     "status": "processed"
   }

DELETE /api/v1/files/{file_id}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

--todo 删除文件。

路径参数：

* **file_id**：文件ID

响应示例：

.. code-block:: json

   {
     "success": true,
     "message": "文件 file_001 删除成功"
   }

知识库管理
----------

GET /api/v1/files/knowledge-bases
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

--todo 获取知识库列表。

响应示例：

.. code-block:: json

   [
     {
       "name": "tech_docs",
       "description": "技术文档知识库",
       "dataset_count": 3,
       "created_time": "2024-01-01T10:00:00Z"
     }
   ]

GET /api/v1/files/knowledge-bases/{kb_name}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

--todo 获取知识库详情。

路径参数：

* **kb_name**：知识库名称

响应示例：

.. code-block:: json

   {
     "success": true,
     "message": "知识库详情获取成功",
     "kb_name": "tech_docs",
     "description": "技术文档知识库",
     "datasets": ["ds001", "ds002"],
     "total_files": 150,
     "created_time": "2024-01-01T10:00:00Z"
   }

GET /api/v1/files/knowledge-bases/{kb_name}/datasets
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

--todo 获取知识库关联的数据集。

路径参数：

* **kb_name**：知识库名称

响应示例：

.. code-block:: json

   {
     "success": true,
     "message": "知识库数据集获取成功",
     "kb_name": "tech_docs",
     "datasets": [
       {
         "name": "ds001",
         "description": "API文档",
         "file_count": 50,
         "created_time": "2024-01-01T11:00:00Z"
       }
     ]
   }

数据集管理
----------

GET /api/v1/files/datasets
~~~~~~~~~~~~~~~~~~~~~~~~~

--todo 获取数据集列表。

响应示例：

.. code-block:: json

   [
     {
       "name": "ds001",
       "description": "API文档数据集",
       "file_count": 50,
       "created_time": "2024-01-01T11:00:00Z"
     }
   ]

GET /api/v1/files/datasets/{dataset_name}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

--todo 获取数据集详情。

路径参数：

* **dataset_name**：数据集名称

响应示例：

.. code-block:: json

   {
     "success": true,
     "message": "数据集详情获取成功",
     "dataset_name": "ds001",
     "description": "API文档数据集",
     "files": [
       {
         "id": "file_001",
         "name": "api_guide.pdf",
         "size": 2048000,
         "upload_time": "2024-01-01T12:00:00Z"
       }
     ],
     "total_files": 50,
     "created_time": "2024-01-01T11:00:00Z"
   }

GET /api/v1/files/datasets/{dataset_name}/files
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

--todo 获取数据集文件列表。

路径参数：

* **dataset_name**：数据集名称

响应示例：

.. code-block:: json

   {
     "success": true,
     "message": "数据集文件列表获取成功",
     "dataset_name": "ds001",
     "files": [
       {
         "id": "file_001",
         "name": "api_guide.pdf",
         "size": 2048000,
         "type": "pdf",
         "upload_time": "2024-01-01T12:00:00Z"
       }
     ],
     "total_count": 50
   }

DELETE /api/v1/files/datasets/{dataset_name}/files/{filename}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

--todo 删除数据集中的文件。

路径参数：

* **dataset_name**：数据集名称
* **filename**：文件名

响应示例：

.. code-block:: json

   {
     "success": true,
     "message": "数据集 ds001 中的文件 api_guide.pdf 删除成功"
   }

数据模型
--------

FileInfo
~~~~~~~~

文件信息模型：

.. code-block:: python

   class FileInfo(BaseModel):
       id: str
       name: str
       size: int
       upload_time: str
       type: str

KnowledgeBaseInfo
~~~~~~~~~~~~~~~~~

知识库信息模型：

.. code-block:: python

   class KnowledgeBaseInfo(BaseModel):
       name: str
       description: str
       dataset_count: int
       created_time: str

DatasetInfo
~~~~~~~~~~~

数据集信息模型：

.. code-block:: python

   class DatasetInfo(BaseModel):
       name: str
       description: str
       file_count: int
       created_time: str

文件处理流程
------------

1. **文件上传**：接收文件并进行基础验证
2. **文件存储**：保存到指定存储位置
3. **元数据记录**：记录文件元信息到数据库
4. **异步处理**：触发后台文件解析和向量化
5. **状态更新**：更新文件处理状态

支持的文件格式
--------------

* **文档文件**：PDF、DOC、DOCX、TXT、MD
* **表格文件**：XLS、XLSX、CSV
* **演示文件**：PPT、PPTX
* **网页文件**：HTML、XML
* **图片文件**：JPG、PNG、GIF（用于OCR）

错误处理
--------

* **400 Bad Request**：文件格式不支持或文件过大
* **404 Not Found**：文件或知识库不存在
* **500 Internal Server Error**：服务器存储错误

配置依赖
--------

文件控制器依赖以下配置：

* **存储配置**：文件存储路径和大小限制
* **数据库配置**：元数据存储数据库连接
* **处理配置**：文件解析和向量化参数
