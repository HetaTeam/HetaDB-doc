.. _quick_start_guide:

快速入门指南
============

本指南将帮助您在 **5 分钟内** 完成 HetaDB 系统的环境搭建、依赖安装与首个示例运行

.. raw:: html

    <div class="quick-start">
        <h3>⚡ 5 分钟快速体验</h3>
        <p>无需复杂配置，三步即可运行完整流程：文件解析 → 数据入库 → 智能问答。</p>
    </div>

环境准备
--------

系统要求
^^^^^^^^

- **操作系统**：Linux / macOS / Windows（推荐 Linux）
- **Python 版本**：3.10 或更高（不支持 3.9 及以下）
- **内存**：≥ 8 GB RAM（处理大文档时建议 ≥ 16 GB）
- **存储**：≥ 10 GB 可用空间（用于模型缓存、中间文件与数据库）
- **GPU（推荐）**：NVIDIA GPU（CUDA 支持），用于加速嵌入计算与 LLM 推理
- **无 GPU 也可运行**：系统会自动回退到 CPU 模式（速度较慢）

安装步骤
^^^^^^^^

1. **创建虚拟环境**（推荐使用 ``uv``，速度更快）

   .. code-block:: bash

      # 安装 uv（现代 Python 包管理器）
      pip install --upgrade pip
      pip install uv

      # 创建并激活虚拟环境
      uv venv heta-db --python=3.10
      source heta-db/bin/activate      # Linux / macOS
      # 或
      heta-db\Scripts\activate         # Windows

   **备选方案：使用 conda**

   .. code-block:: bash

      conda create -n heta-db python=3.10 -y
      conda activate heta-db

2. **安装 HetaDB 及其依赖**

   .. code-block:: bash

      # 以可编辑模式安装项目（开发模式）
      uv pip install -e .

   此命令将自动安装：

   * 向量数据库（Milvus 2.6.6 / pymilvus）
   * 数据库驱动（psycopg2 2.9.11）
   * 文档解析工具（MinerU 2.7.1、beautifulsoup4 4.14.3、pandas 2.3.3 等）
   * Web框架（FastAPI 0.128.0、uvicorn 0.40.0）

3. **验证安装**

   .. code-block:: bash

      # 验证 PyTorch
      python -c "import torch; print('✅ PyTorch', torch.__version__, '| CUDA available:', torch.cuda.is_available())"

      # 验证 FastAPI
      python -c "import fastapi; print('✅ FastAPI', fastapi.__version__)"

      # 验证 Transformers
      python -c "import transformers; print('✅ Transformers', transformers.__version__)"

      # 验证 HetaDB 模块
      python -c "from src.file_parsing import ParserAssignment; print('✅ HetaDB core imported successfully')"

   > 若无报错且输出版本信息，则安装成功。

框架结构
--------

HetaDB 实现了完整的 **知识图谱构建与检索增强生成 (RAG)** 流水线，采用理论驱动的方法论，系统性地将非结构化数据转换为结构化知识表示。

流水线架构
^^^^^^^^^^

HetaDB 流水线采用分阶段处理策略，系统性地将原始文档转换为结构化知识表示：

**阶段一：数据清理与文件解析**

.. code-block:: bash

   # 执行完整知识图谱构建流水线
   python scripts/file_process.py

该阶段基于 ``ParserAssignment`` 类实现多模态文档解析：

* **文件类型检测**：自动识别 PDF、DOCX、HTML、图像等格式
* **内容抽取**：调用 MinerU、BeautifulSoup4 等工具解析文档结构
* **LLM/VLM 增强**：对复杂文档使用语言模型和视觉模型进行内容理解
* **结构化存储**：生成标准 JSONL 格式，包含文本、图像、元数据

**阶段二：文本分块处理**

基于 ``text_chunker`` 模块实现智能文本分割：

* **动态分块**：根据 ``chunk_size`` 和 ``overlap`` 参数进行文本切分
* **语义完整性**：确保分块边界不破坏句子和段落结构
* **去重合并**：通过 ``chunks_merge_func`` 基于语义相似度合并重复内容
* **重新切分**：使用 ``rechunk_by_source`` 优化分块粒度

**阶段三：知识图谱抽取**

基于 ``graph_extraction`` 模块构建实体关系图谱：

* **实体识别**：使用 LLM 进行命名实体识别和分类
* **关系抽取**：基于预定义 schema 抽取实体间关系
* **批量处理**：支持 ``batch_size`` 参数控制并发处理规模
* **Schema 驱动**：支持 CSV 文件定义实体和关系类型约束

**阶段四：节点去重与合并**

基于 ``node_dedup_merge`` 模块实现实体消歧：

* **语义去重**：使用 LLM 判断实体语义等价性
* **向量嵌入**：将节点转换为向量表示用于相似度计算
* **聚类合并**：基于向量相似度和阈值进行实体聚类
* **映射管理**：维护实体合并的映射关系

**阶段五：关系处理与优化**

基于 ``rel_dedup_merge`` 模块优化关系数据：

* **关系去重**：基于实体映射和语义相似度消除重复关系
* **关系嵌入**：为关系生成向量表示
* **关系合并**：聚合同义关系并更新连接性
* **一致性保证**：确保关系与节点合并后的引用一致性

**阶段六：索引构建与导出**

构建检索索引并导出最终数据：

* **向量索引**：在 Milvus 中创建 IVF_FLAT 索引
* **关系导出**：导出 cluster-chunk 关系映射
* **数据验证**：确保所有数据的完整性和一致性

配置管理理论
^^^^^^^^^^^^

HetaDB 采用 **配置即代码** 的软件工程实践，通过结构化配置管理实现系统可复现性：

**模型配置体系**

.. code-block:: yaml

   llm:
     multi_hop:
       model: "your-llm-model"  # 替换为实际的LLM模型名称
       api_key: "your-api-key"  # 替换为实际的API密钥
       base_url: "https://your-api-endpoint/v1"  # 替换为实际的API端点
       timeout: 120

   vlm:
     model: "your-vlm-model"  # 替换为实际的视觉语言模型
     api_key: "your-vlm-api-key"  # 替换为实际的VLM API密钥

   embedding:
     model: "your-embedding-model"  # 替换为实际的嵌入模型
     dim: 1024  # 根据选择的模型调整维度
     batch_size: 32

**数据库配置**

.. code-block:: yaml

   database:
     postgres:
       host: "your-postgres-host"  # PostgreSQL服务器地址
       port: 5432
       database: "your-database-name"  # 数据库名称
       user: "your-username"  # 数据库用户名
       password: "your-password"  # 数据库密码

     milvus:
       host: "your-milvus-host"  # Milvus服务器地址
       port: 19530
       collection_prefix: "heta_"  # 集合前缀

**处理参数调优**

.. code-block:: yaml

   processing:
     chunk_size: 512          # 文本分块大小，可根据文档类型调整
     overlap: 50              # 分块重叠大小，确保上下文连贯性
     top_k: 10                # 检索返回的Top-K结果数量
     merge_threshold: 0.85    # 相似度合并阈值，建议0.8-0.9之间
     max_concurrent: 4        # 最大并发处理数量，根据系统资源调整

运行示例
--------

**完整知识库构建**

.. code-block:: bash

   # 执行基于 scripts/file_process.py 的完整流水线
   python scripts/file_process.py

该命令按实际处理顺序执行六个阶段：

1. **数据清理阶段**：清理 ``clear_data_by_sources()`` 相关旧数据
2. **文件解析阶段**：调用 ``run_file_parse_pipeline()`` 解析多模态文档
3. **分块处理阶段**：执行 ``run_chunk_processing_pipeline()`` 进行文本切分
4. **图谱抽取阶段**：运行 ``run_graph_extraction_pipeline()`` 构建知识图谱
5. **节点处理阶段**：通过 ``run_node_processing_pipeline()`` 进行实体去重合并
6. **关系处理阶段**：执行 ``run_relation_processing_pipeline()`` 优化关系数据

**智能问答服务启动**

.. code-block:: bash

   # 启动 FastAPI 知识库查询服务
   python src/api_entry.py

服务启动后，通过标准 REST API 接口进行查询：

.. code-block:: bash

   # 基于真实 QueryRequest 模型的查询示例
   curl -X POST "http://localhost:8000/api/v1/chat/" \
        -H "Content-Type: application/json" \
        -d '{
              "query": "项目报告中的关键技术有哪些？",
              "user_id": "demo_user",
              "kb_id": "kb001",
              "top_k": 5,
              "max_results": 10,
              "mode": 2
            }'

**真实 API 响应格式**

基于 ``QueryResponse`` 模型的标准化响应：

.. code-block:: json

   {
     "success": true,
     "message": "查询成功",
     "data": [
       {
         "kb_id": "kb001",
         "kb_name": "项目文档库",
         "score": 0.87,
         "content": "关键技术包括知识图谱构建、向量检索、LLM集成等...",
         "text": "关键技术包括知识图谱构建、向量检索、LLM集成等...",
         "source_id": ["chunk_001", "chunk_002"]
       }
     ],
     "total_count": 1,
     "query_info": {
       "query_time": 1.2,
       "retrieval_mode": "multi_hop",
       "chunks_searched": 1500,
       "nodes_processed": 45
     },
     "request_id": "req_abc123def456"
   }

**检索策略说明**

基于 ``chat_controller.py`` 的实际实现：

* ``mode=0``：**朴素知识库查询** - ``perform_naive_kb_response()``
* ``mode=1``：**查询重写器查询** - ``perform_query_rewriter_kb_response()``
* ``mode=2``：**多跳推理查询** - ``perform_multi_hop_kb_response()``
* ``mode=3``：**朴素回答生成** - ``perform_naive_response()``

**数据管理工具**

* ``scripts/file_process.py``：完整流水线处理脚本
* ``scripts/parse_any_file.py``：通用文档解析工具
* ``scripts/get_emails_by_imap.py``：邮件数据获取工具

